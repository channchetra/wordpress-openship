# WordPress (Docker Compose)

A small, self-contained WordPress stack: the official `wordpress` image plus MySQL 8.4,
one published port, named volumes for site files and database data. No reverse proxy,
no platform-side template — just `docker compose`.

## Quick start

```sh
cp .env.example .env
# edit .env: at minimum set DB_PASSWORD, WP_HOME, WP_SITEURL
docker compose up -d
docker compose logs -f wordpress
```

Then open <http://localhost:8080> and complete the WordPress installer
(site title, admin user, password). Everything is stored in the two named volumes,
so `docker compose down` is safe and `docker compose down -v` destroys the site.

## What's in the box

| File | Purpose |
| --- | --- |
| `compose.yaml` | `wordpress` + `db` services, healthchecks, volumes, log rotation |
| `.env.example` | Every supported variable, with defaults filled in |
| `uploads.ini` | PHP upload/memory limits, mounted read-only into the container |

Volumes: `wp_app` → `/var/www/html` (core, themes, plugins, uploads), `wp_data` → `/var/lib/mysql`.

## `.env` reference

`DB_PASSWORD` is required — compose refuses to start without it. Everything else has a default.

| Variable | Default | Notes |
| --- | --- | --- |
| `DB_NAME` | `wordpress` | Created on the db container's first boot |
| `DB_USER` | `wordpress` | Application user; db is not reachable from the host |
| `DB_PASSWORD` | — | **Required.** Long random value |
| `DB_TABLE_PREFIX` | `wp_` | Set before first boot; changing later orphans tables |
| `DB_ROOT_PASSWORD` | empty | Empty = strong random password generated on first boot |
| `WORDPRESS_PORT` | `8080` | Host port mapped to the container's port 80 |
| `WORDPRESS_BIND_ADDR` | `0.0.0.0` | Use `127.0.0.1` to keep it loopback-only |
| `WP_HOME`, `WP_SITEURL` | empty | Public site URL, including scheme |
| `WORDPRESS_DEBUG` | `0` | `1` enables `WP_DEBUG` |
| `WORDPRESS_*_KEY` / `*_SALT` | sample values | Eight values; replace all with your own |
| `WORDPRESS_IMAGE` | `wordpress:7.1-php8.3-apache` | Pin/override the WordPress image |
| `DB_IMAGE` | `mysql:8.4` | Pin/override the MySQL image |

Generate a password and a fresh salt set:

```sh
openssl rand -base64 36
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Changing the salts logs every user out — that is the intended effect.

## Adding a domain and TLS

This stack serves plain HTTP on a host port. To put a real domain in front of it,
terminate TLS in a reverse proxy (Caddy, nginx, Traefik) that forwards to
`127.0.0.1:8080`, then set `WORDPRESS_BIND_ADDR=127.0.0.1` and point
`WP_HOME`/`WP_SITEURL` at the public `https://` URL.

The official image already translates `X-Forwarded-Proto: https` into `$_SERVER['HTTPS']`,
so no extra PHP configuration is needed. Behind a proxy, forward that header —
otherwise WordPress will redirect to `http://` and logins can loop.

## Common operations

```sh
docker compose ps                      # status + health
docker compose logs -f db              # MySQL logs (first-boot root password appears here)
docker compose exec db mysql -u root -p                  # SQL shell
docker compose exec -u www-data wordpress wp plugin list # WP-CLI ships in the image
docker compose exec -u www-data wordpress wp db check
docker compose down                    # stop, keep all data
docker compose up -d --pull always     # update to the pinned images' latest digest
```

Backups: dump the database and archive the uploads volume.

```sh
docker compose exec db sh -c 'exec mysqldump --single-transaction -u "$MYSQL_USER" -p"$MYSQL_PWD" "$MYSQL_DATABASE"' > backup.sql
docker run --rm -v wordpress_wp_app:/data -v "$PWD":/backup alpine tar czf /backup/wp-content.tgz -C /data wp-content
```

## Configuration notes

- **`WP_HOME`/`WP_SITEURL` are defined through `WORDPRESS_CONFIG_EXTRA`, not a `WORDPRESS_*`
  variable.** The official image has no environment variable for the site URL; the snippet in
  `compose.yaml` reads them and defines the constants, logging a warning to the container log
  if a value is missing or not an `http(s)` URL. When they are empty WordPress falls back to the
  `siteurl` row in the database, which is the value written during installation.
- **Editing `WORDPRESS_CONFIG_EXTRA`:** it is `eval()`'d as PHP, and compose interpolates the
  block, so escape every literal PHP `$` as `$$`. Never feed this block untrusted input.
- **`uploads.ini`** raises `upload_max_filesize`/`post_max_size` to 64M and
  `memory_limit` to 256M. Keep `post_max_size >= upload_max_filesize` or large uploads are
  dropped before PHP sees them.
- **`DISALLOW_FILE_EDIT`** is on (no plugin/theme code editor in wp-admin). Set it to `false`
  in the `WORDPRESS_CONFIG_EXTRA` block if you want the editor back.
- **Database access** is intentionally not published to the host; use
  `docker compose exec db ...` for direct SQL.
- **Root password:** with `DB_ROOT_PASSWORD` empty, retrieve the generated one once with
  `docker compose logs db | grep -i "GENERATED ROOT PASSWORD"`. It is only printed on first boot.

## Requirements

Docker Engine with the Compose v2 plugin (`docker compose version`).
