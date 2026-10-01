# Deployment

Copies of the production server config, so a rebuilt box can be brought back to
the same state. These files are **not** applied automatically — they are a record
of what is running.

Live on `24fastgo.com` (VPS `187.127.179.65`, a CloudPanel box — leave its other
sites and services alone):

| What | Where |
|---|---|
| Repo checkout | `/var/www/24fastgo` |
| Built frontend | `/var/www/24fastgo-web` |
| `nginx/24fastgo.conf` | `/etc/nginx/sites-enabled/24fastgo.conf` (nginx only loads `*.conf` there) |
| TLS cert | Let's Encrypt via `certbot --webroot -w /var/www/24fastgo-acme`, auto-renews and reloads nginx |
| MongoDB 8.0 | local, `127.0.0.1:27017`, auth on, db `24fastgo_taxi`; credentials in `/root/.24fastgo-mongo-credentials` |
| Redis | the box's shared Redis, DB index `5` |

The app runs under PM2 as four instances (`24fastgo-api-5000` … `5003`, see
`Backend/ecosystem.config.cjs`), which nginx load-balances via the upstream block.
`pm2 save` + the `pm2-root` systemd unit bring them back after a reboot.

## Frontend deploys

```
cd /var/www/24fastgo && git pull
cd frontend && npm ci && npm run build
rsync -a --delete dist/ /var/www/24fastgo-web/
```

`frontend/.env.production` (not in git) holds the build-time `VITE_*` values.

`--delete` is deliberate: it clears the previous build's hashed asset files so
they do not accumulate.

### Why `index.html` must not be cached

`index.html` names the hashed chunks, and each deploy replaces them and removes
the previous set. If a browser holds a stale `index.html`, it requests chunk
filenames that no longer exist, gets 404s, and renders a **blank page** — the
shell paints but nothing else does.

The config therefore splits the two:

- `location = /index.html` → `no-cache, must-revalidate`
- `location /assets/` → `public, immutable`, one year

Keep that split if the config is ever rewritten.

## Backend deploys

Backend changes need no build:

```
cd /var/www/24fastgo && git pull
cd Backend && npm ci --omit=dev
pm2 restart ecosystem.config.cjs --update-env
```

`--update-env` matters when `Backend/.env` has changed; without it the processes
keep their old environment.
