# WordPress (Docker Compose)

A small, self-contained WordPress stack: the official `wordpress` image plus MySQL 8.4,
one published port, named volumes for site files and database data. No reverse proxy,
no platform-side template — just `docker compose`.

## Quick start

```sh
cp .env.example .env
# edit .env: set DB_PASSWORD (required); optionally set WP_HOME / WP_SITEURL
docker compose up -d
docker compose logs -f wordpress
```

No build step: every image is pulled, and the stack declares no bind mounts, so it deploys
as-is on platforms that only run pre-built images.

Then open <http://localhost:8080> and complete the WordPress installer
(site title, admin user, password). Everything is stored in the two named volumes,
so `docker compose down` is safe and `docker compose down -v` destroys the site.
Keys and salts are generated automatically on first boot — there is nothing to paste.

## What's in the box

| File | Purpose |
| --- | --- |
| `compose.yaml` | `wordpress` + `db` services, healthchecks, volumes, log rotation |
| `.env.example` | Every supported variable, with defaults filled in |

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
| `WP_HOME`, `WP_SITEURL` | empty | Optional; pin the public site URL (see below) |
| `WORDPRESS_DEBUG` | `0` | `1` enables `WP_DEBUG` |
| `WORDPRESS_IMAGE` | `wordpress:7.1-php8.3-apache` | Pin/override the WordPress image |
| `DB_IMAGE` | `mysql:8.4` | Pin/override the MySQL image |

Generate a password:

```sh
openssl rand -base64 36
```

Keys and salts are **not** in this list on purpose — see below.

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

- **Keys and salts need no configuration.** On first boot the entrypoint writes `wp-config.php`
  and substitutes each `put your unique phrase here` placeholder with a unique random value
  (`sha1sum` of 1 MB from `/dev/urandom`), so all eight keys/salts are unique per site and live in
  `wp-config.php` inside the `wp_app` volume. Do not add `WORDPRESS_AUTH_KEY` & friends as *empty*
  strings — the entrypoint then writes empty constants, and WordPress falls back to generating its
  own salts into the `wp_options` table. To pin your own set (e.g. to survive `down -v`), add them
  with real 64+ character values; changing them later logs every user out, which is intended.
- **`WP_HOME`/`WP_SITEURL` are optional, and are defined through `WORDPRESS_CONFIG_EXTRA`, not a
  `WORDPRESS_*` variable.** The image has no environment variable for the site URL. WordPress
  resolves its own URLs from the `home`/`siteurl` rows in `wp_options`; these constants override
  those rows.
  - *Set them* when the public URL is known: every emitted URL is pinned, so a later domain or host
    change is one `.env` edit with no database surgery.
  - *Leave them empty* to let the database drive everything. The installer seeds `home` and
    `siteurl` from the URL it was reached at, so a later domain change means
    `wp option update home …` and `wp option update siteurl …`.
  - Never set a `localhost` placeholder while serving a real domain — WordPress would redirect
    visitors and assets there. Set the URL visitors actually use, or leave it empty.
- **`memory_limit` and `max_execution_time` are raised at runtime** in the `WORDPRESS_CONFIG_EXTRA`
  block (`@ini_set` to 256M / 300s), because both are `PHP_INI_ALL` and the stock image ships no
  `php.ini`. `WP_MEMORY_LIMIT` is set alongside them. This covers media processing and plugin
  installs; it does not affect upload sizes, which are parser-level — see the upload-limit note above.
- **Editing `WORDPRESS_CONFIG_EXTRA`:** it is `eval()`'d as PHP, and compose interpolates the
  block, so escape every literal PHP `$` as `$$`. Never feed this block untrusted input.
- **Upload size limits are not configurable from this repo — here is exactly why.** The stack runs
  the stock image, so PHP's compiled-in defaults apply: `upload_max_filesize = 2M`,
  `post_max_size = 8M`, `max_file_uploads = 20` (the `php` image installs no `php.ini`, only the
  `php.ini-*` templates). Those two directives are consumed by PHP while it parses the request body,
  *before* any PHP code runs, so neither `ini_set()` in `WORDPRESS_CONFIG_EXTRA` nor
  `WP_MEMORY_LIMIT` can raise them — only a real `php.ini` can. That means one of:
  - a platform-level PHP/upload configuration, if your host exposes one;
  - an **absolute-path** bind mount of an ini file,
    e.g. `/srv/wordpress/uploads.ini:/usr/local/etc/php/conf.d/zz-uploads.ini:ro`. Relative
    sources such as `./uploads.ini` are rejected by the Docker engine itself (it parses them as
    named-volume names, which forbids `.` and `/`), so that form cannot work on any platform;
  - a custom image that copies the ini in (`FROM wordpress:7.1-php8.3-apache` +
    `COPY uploads.ini $PHP_INI_DIR/conf.d/`), for hosts that build images.
  `memory_limit` (128M) and `max_execution_time` (30) *can* be raised at runtime, and the
  `WORDPRESS_CONFIG_EXTRA` block already does so — see the note on that block below.
- **`DISALLOW_FILE_EDIT`** is on (no plugin/theme code editor in wp-admin). Set it to `false`
  in the `WORDPRESS_CONFIG_EXTRA` block if you want the editor back.
- **Database access** is intentionally not published to the host; use
  `docker compose exec db ...` for direct SQL.
- **Root password:** with `DB_ROOT_PASSWORD` empty, retrieve the generated one once with
  `docker compose logs db | grep -i "GENERATED ROOT PASSWORD"`. It is only printed on first boot.

## Requirements

Docker Engine with the Compose v2 plugin (`docker compose version`).
