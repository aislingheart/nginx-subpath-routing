# nginx subpath routing for self-hosted apps

Serving a dozen self-hosted services from **one hostname and one port** by
giving each one a path prefix — `/sonarr`, `/jellyfin`, `/ai`, and so on —
with a dashboard at `/`.

This is a write-up of the config, but weighted toward the *problem*. Most
"how do I reverse proxy X" guides stop at the happy path. This one covers
the four things that actually break, all of which produce failures that look
like something other than a proxy problem.

```nginx
https://box.example.com/           -> Homarr dashboard
https://box.example.com/sonarr     -> Sonarr
https://box.example.com/jellyfin   -> Jellyfin
https://box.example.com/ai         -> Open WebUI
```

---

## Contents

- [The core problem](#the-core-problem)
- [Two ways to fix an app](#two-ways-to-fix-an-app)
- [Gotcha 1: the trailing slash](#gotcha-1-the-trailing-slash)
- [Gotcha 2: websockets](#gotcha-2-websockets)
- [Gotcha 3: `sub_filter` and gzip](#gotcha-3-sub_filter-and-gzip)
- [Gotcha 4: Base URL in the app, not just nginx](#gotcha-4-base-url-in-the-app-not-just-nginx)
- [Setup](#setup)
- [Verification checklist](#verification-checklist)
- [Troubleshooting table](#troubleshooting-table)

---

## The core problem

Most self-hosted apps assume they are the entire website. They don't know
they're being served from `/sonarr/`, so they generate links like:

```html
<a href="/dashboard">Dashboard</a>        <!-- goes to nginx, not Sonarr -->
<script src="/_app/bundle.js"></script>   <!-- 404 -->
fetch("/api/v3/series")                   <!-- hits the dashboard instead -->
```

Three different things break depending on the app:

1. **Links and redirects** — nginx receives `/dashboard` and has no idea you
   meant `/sonarr/dashboard`.
2. **Assets** — JS and CSS are requested from the root and 404.
3. **API calls** — `fetch("/api/...")` from a page served at `/sonarr/`
   targets the root, so it hits whatever else is mounted there.

A third category is subtler: **redirects**. An app that has just been told
its base URL is `/sonarr` will happily `302` you to `/sonarr/login` — correct
— but an app that has *not* will `302` you to `/login`, which drops you out
of the prefix entirely.

---

## Two ways to fix an app

Always try these in order. The first is far more robust.

### 1. Base URL setting (preferred)

Most of these apps have a setting, and it fixes links, assets, API calls and
redirects in one go:

| App | Where the setting lives |
| :--- | :--- |
| Sonarr | Settings → General → **Base URL** |
| Radarr | Settings → General → **Base URL** |
| Lidarr | Settings → General → **Base URL** |
| Prowlarr | Settings → General → **Base URL** |
| Bazarr | Settings → General → **Base URL** |
| Jellyfin | Dashboard → Networking → **Base URL** |

Set it to match the prefix exactly, trailing slash and all:

```
Base URL = /sonarr
```

nginx then only has to strip the prefix, which the trailing slash on
`proxy_pass` does for you.

### 2. Path rewrite with `sub_filter`

Some apps have no such setting. Open WebUI is the common one: it hardcodes
`/_app/`, `/static/` and `/api/` in its HTML, CSS and JavaScript. Here you
rewrite the strings in the response body as they pass through nginx:

```nginx
location /ai/ {
    proxy_pass http://open-webui:8080/;

    proxy_set_header Accept-Encoding "";

    sub_filter_types text/css application/javascript text/javascript application/json;
    sub_filter_once off;

    sub_filter '/_app/'   '/ai/_app/';
    sub_filter '/static/'  '/ai/static/';
    sub_filter '/api/'     '/ai/api/';
}
```

Read [Gotcha 3](#gotcha-3-sub_filter-and-gzip) before using this. Two of
those three lines exist to stop it silently doing nothing.

---

## Gotcha 1: the trailing slash

The single most common cause of "nginx returns 404 for every request even
though the container is fine":

```nginx
location /sonarr/ {
    proxy_pass http://sonarr:8989/;   # trailing slash — CORRECT
}
```

nginx treats a trailing slash on `proxy_pass` as "strip the matching
`location` prefix". So a request for `/sonarr/api/v3/series` becomes
`/api/v3/series` before it reaches the container.

Without the trailing slash, the prefix is **kept**, and the app receives
`/sonarr/api/v3/series` — a path it does not recognise. The app is healthy
and returning 200s when tested directly; it is only 404ing through nginx,
which sends people debugging the wrong container.

> **Rule:** if the app works on its own port but 404s through nginx, check
> this slash before anything else.

---

## Gotcha 2: websockets

Prowlarr, qBittorrent and Homarr hold a persistent websocket open for
progress bars and job updates. Without the right headers the page loads
fine and then **never updates** — it looks like a frozen app rather than a
proxy fault.

That needs a `map` in the **`http`** context, not inside a server block:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

```nginx
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

The map is not optional shorthand. `Connection` must be `close` for ordinary
requests and `upgrade` only when the client actually asked to upgrade.
Hardcoding `Connection: upgrade` on every request breaks keep-alive and
leaves connections hanging.

> If you put the `map` inside `server {}` nginx refuses to start with
> `"map" directive is not allowed here`. That's why this repo ships it as a
> separate `conf.d/` file.

---

## Gotcha 3: `sub_filter` and gzip

This one costs people hours, because **the symptom is "my config is correct
and nothing happens."**

`sub_filter` matches text in the response body. If the upstream sends
gzipped content, the body is binary, the patterns don't match, and the
rewrite silently does nothing. No error, no warning, no log line.

The fix is to ask the upstream not to compress:

```nginx
proxy_set_header Accept-Encoding "";
```

The cost is real: **responses are no longer compressed on the way out.**
For a text-heavy app on a LAN that's a fair trade. For something
bandwidth-heavy, prefer the base URL route and skip `sub_filter` entirely.

Two more that behave the same way — silently:

- **`sub_filter_types` defaults to `text/html` only.** Your JavaScript and
  CSS are not being rewritten unless you list them explicitly. Symptom: HTML
  looks right, the page is still broken.
- **`sub_filter_once` defaults to on**, rewriting only the *first* occurrence
  per body. Symptom: the page half-works — the first script loads, the rest
  404. This is the most confusing failure mode, because it looks like the app
  is flaky.

Also worth knowing: `sub_filter` is literal string replacement, not regex,
and it does not understand URL semantics. It will happily rewrite `/api/`
inside a comment or a string in your data. Keep the pattern list as narrow
as you can and re-check it when the upstream version changes.

---

## Gotcha 4: Base URL in the app, not just nginx

Setting the prefix in nginx is half the job. The app needs to agree.

Two symptoms of a mismatch:

- **Redirects drop the prefix.** You request `/sonarr`, the app `302`s to
  `/login`, nginx 404s.
- **Cross-app calls 404.** This is the subtle one: Sonarr is told its base
  URL is `/sonarr`, and it is configured to reach Radarr at
  `http://radarr:7878`. If Radarr is told its base URL is `/radarr`, Sonarr
  builds `http://radarr:7878/radarr/api/v3/...` — which is correct. But if
  you proxied Radarr at `/radarr` while its *own* base URL is unset, the
  two disagree and every inter-app call fails while both UIs look healthy.

The tell is that **one app's UI works and its integrations silently do
nothing** — download clients not appearing, indexers not connecting,
downloads never triggering.

> Keep a table of prefix → app → configured Base URL. When something breaks
> at 2am, this is the first thing to check.

---

## Setup

```bash
git clone https://github.com/YOUR_USER/nginx-subpath-routing.git
cd nginx-subpath-routing

# 1. the websocket map -- must be in the http context
sudo cp nginx/conf.d/00-websocket-map.conf /etc/nginx/conf.d/

# 2. edit nginx/subpath-routing.conf: set hostnames, ports, prefixes
sudo cp nginx/subpath-routing.conf /etc/nginx/

# 3. reference it from your server block
#    include /etc/nginx/subpath-routing.conf;
```

Then wrap it in a server block. **Note the split** — the `map` goes in
`conf.d/`, the `location` blocks go inside `server {}`:

```nginx
server {
    listen 443 ssl;
    server_name box.example.com;

    ssl_certificate     /etc/letsencrypt/live/box.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/box.example.com/privkey.pem;

    include /etc/nginx/subpath-routing.conf;
}
```

Validate before reloading. `nginx -t` catches structural mistakes, and
reloading a broken config takes the whole site down:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Container names (`sonarr`, `radarr`, …) resolve on a Docker network. If your
apps publish fixed host ports instead, use `127.0.0.1:8989` in the
`proxy_pass`.

---

## Verification checklist

Work through this when something misbehaves:

- [ ] `nginx -t` passes
- [ ] App works **directly** on its own port, bypassing nginx
- [ ] App's own **Base URL** matches the prefix in `location`
- [ ] `proxy_pass` has the **trailing slash**
- [ ] Websockets updating? (progress bars, job lists)
- [ ] Browser devtools → Network: are the 404s `/static/...` or `/api/...` at
      the **root**? That's the subpath problem, not a broken asset.
- [ ] For `sub_filter`: is `Accept-Encoding ""` set, and are
      `sub_filter_once off` + `sub_filter_types` covering the failing file?
- [ ] Cross-app calls working? (Sonarr → Radarr, *arr → qBittorrent)
- [ ] Redirects keeping the prefix?

---

## Troubleshooting table

| Symptom | Most likely cause | Fix |
| :--- | :--- | :--- |
| Every path 404s through nginx, app is fine directly | Missing trailing slash on `proxy_pass` | Add `/` |
| Page loads, assets 404 at root | App hardcodes absolute paths, no Base URL setting | `sub_filter` (+ read Gotcha 3) |
| Page half-works, some assets 404 | `sub_filter_once` left at default | Set `sub_filter_once off` |
| `sub_filter` does nothing at all | Upstream is gzipping | `proxy_set_header Accept-Encoding ""` |
| Loads, never updates | Websockets not passing through | Add the `map` + Upgrade/Connection headers |
| `302` to `/login` instead of `/sonarr/login` | App Base URL unset | Set the app's Base URL |
| One app fine, its integrations do nothing | Base URL mismatch between apps | Reconcile the prefix table |
| Video seeking stalls | Range header dropped or `proxy_buffering on` | Pass `Range`, set `proxy_buffering off` |
| `504` during a large import | Default 60s read timeout | Raise `proxy_read_timeout` |
| nginx won't start: `"map" directive is not allowed here` | `map` placed inside `server {}` | Move to `conf.d/` (http context) |
| nginx won't start: `host not found in upstream` | Container name doesn't resolve | Check the Docker network, or use `127.0.0.1:port` |

---

## A note on scope

This covers reverse proxying over HTTP. If you terminate TLS, the apps also
need to know they're behind a proxy, or they will generate `http://` links
and you'll get mixed-content failures:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

included above, and for some apps a trust-proxy setting on top.

## License

MIT — see [LICENSE](LICENSE).
