# Per-app setup after nginx is routing

The nginx config is only half of it. Each app also has to be told its own
base URL, or it will generate links and redirects that escape the prefix.

Copy this table and fill it in as you add services. When something breaks at
2am, this is the first thing to check — see Gotcha 4 in the main README.

| Prefix      | App           | App's Base URL setting | Where                        | Done |
| :--- | :--- | :--- | :--- | :--- |
| `/`         | Homarr        | n/a (served at root)   | —                            | [ ] |
| `/sonarr/`  | Sonarr        | `/sonarr`              | Settings → General           | [ ] |
| `/radarr/`  | Radarr        | `/radarr`              | Settings → General           | [ ] |
| `/lidarr/`  | Lidarr        | `/lidarr`              | Settings → General           | [ ] |
| `/prowlarr/`| Prowlarr      | `/prowlarr`            | Settings → General           | [ ] |
| `/bazarr/`  | Bazarr        | `/bazarr`              | Settings → General           | [ ] |
| `/jellyfin/`| Jellyfin      | `/jellyfin`            | Dashboard → Networking       | [ ] |
| `/ai/`      | Open WebUI    | none — uses `sub_filter` | see Gotcha 3                 | [ ] |

## Cross-app wiring

Services should talk to each other **by container name**, not through nginx.
Going via nginx means routing an internal API call through the external
entrypoint, which breaks whenever a base URL is mismatched:

```
Sonarr  -> Radarr   http://radarr:7878      (not https://box.example.com/radarr)
Sonarr  -> qBit     http://qbittorrent:8080 (not via /qbittorrent)
Prowlarr-> *arr     http://sonarr:8989      (not via /sonarr)
```

The only reason to use the proxied URL is if something genuinely needs to
reach the service from outside your network.

## Order of operations

Adding a new app, in the order that avoids a broken window:

1. Start the container, confirm it works on its own port.
2. Add the `location` block, `nginx -t`, reload.
3. Confirm it works through nginx at the subpath.
4. *Now* set the app's Base URL to match the prefix.
5. Check its integrations still connect.

Doing 4 before 3 tends to produce a confusing half-broken state, because
the app starts redirecting to a prefix nginx isn't serving yet.
