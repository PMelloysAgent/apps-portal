# apps-portal

The serving layer for `apps.paulmelloy.com.au` — a small Caddy **portal** that shows
the app index page and routes `/shiny/<AppName>/*` to Shiny containers, plus a single
generic **app host** image that every app gets deployed *into*.

```
apps.paulmelloy.com.au  ──TLS──▶  Cloudflare Tunnel  ──http──▶  :3838 (Caddy portal)
                                                               ├── /                          → index.html
                                                               ├── /shiny/<App>/  (stripped)  → <app>:3838 (Shiny)
                                                               └── ...one route per app
```

## Why this shape

* Shiny cannot mount an app on a URL subpath (verified against shiny 1.14: no
  `base_url` support) — but it emits **relative** asset URLs and derives its
  websocket path from the page URL. So a proxy that **strips the prefix** keeps
  the whole session inside `/shiny/<App>/` with zero app-side changes.
* Apps are **mounted, not baked**: one image (`Dockerfile.app-host`), every app
  is its own git repo cloned under `apps/`. Updating an app = `git pull` +
  restart, **no image rebuild**.
* Plain HTTP by design — TLS terminates at the Cloudflare Tunnel edge.

## Layout

| File | Role |
|---|---|
| `docker-compose.yml` | portal (Caddy) + one service block per app |
| `Caddyfile` | index serving + the app route table |
| `index.html` | the app list at the site root |
| `Dockerfile.app-host` | R + the shared package universe (rocker/r2u, multi-arch) |
| `apps/` | gitignored — each app repo gets cloned here |

## Adding an app

1. **Clone it in:** `git clone https://github.com/<you>/<NewApp> apps/NewApp`
2. **Service block** in `docker-compose.yml` (copy `motorcyclefuel:`, adjust paths):
   ```yaml
   newapp:
     image: app-host:1.0
     container_name: newapp
     restart: unless-stopped
     volumes:
       - ./apps/NewApp:/srv/app:ro
     command: ["R", "-e", "shiny::runApp('/srv/app', port=3838, host='0.0.0.0', launch.browser=FALSE)"]
     environment: [TZ=Australia/Brisbane]
     healthcheck:
       test: ["CMD", "curl", "-sf", "http://localhost:3838/"]
       interval: 30s
       timeout: 10s
       retries: 3
       start_period: 30s
     networks: [apps-net]
   ```
   If the app loads data via paths outside its folder (e.g. `../data/x.csv`),
   mount that sibling folder too, mirroring the repo layout.
3. **Route** in `Caddyfile` (before the portal `handle` block):
   ```
   redir /shiny/NewApp /shiny/NewApp/ 302
   handle_path /shiny/NewApp/* {
       reverse_proxy newapp:3838
   }
   ```
4. **Card** in `index.html` (copy an existing `<a class="card">` block).
5. **Bring it up:** `docker compose up -d` (Caddyfile-only changes can hot-reload:
   `docker exec apps-portal-caddy caddy reload --config /etc/caddy/Caddyfile`)

If the app needs R packages outside the shared universe, add
`r-cran-<pkg>` to `Dockerfile.app-host`, then `docker compose build && docker
compose up -d` (all binary, a few minutes).

## Updating an app

```bash
git -C apps/<Name> pull
docker compose restart <service>
```

## Cloudflare Tunnel

One ingress rule for the whole portal (no per-app paths):

```yaml
ingress:
  - hostname: apps.paulmelloy.com.au
    service: http://localhost:3838
  - service: http_status:404
```

## Operational notes

* Container names are pinned — this compose project should only be run from its
  canonical checkout (`~/selfhost/apps-portal` on hermpi).
* hermpi's kernel silently drops cgroup memory limits — no `deploy.resources`
  limits in this file.
* A single shared image means one R/shiny version for all apps; upgrading
  `Dockerfile.app-host` affects every app at once.
