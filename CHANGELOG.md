# Changelog

## 1.2.0 — 2026-11-12

- `LINK_PLUGINS_DIR` / `LINK_THEMES_DIR`: link every plugin/theme inside a folder (detected by their `Plugin Name:` / `Theme Name:` headers). Folder plugins are linked, not activated.
- Your linked copy replaces a same-named stock plugin/theme that ships with WordPress; if two of your own folders clash, the first wins and the other is skipped with a warning.
- Safety: the default-content cleanup no longer deletes a symlinked plugin or theme (WordPress would follow the link and erase the source files).
- A theme passed to `--theme=<dir>` that's already linked via `LINK_THEMES_DIR` no longer aborts the create.

## 1.1.4 — 2026-11-12

- `--multisite` no longer creates subsites by default (`MULTISITE_SITES=0`). Use `--sites=2` or `--sites=shop,blog` to add them.

## 1.1.3 — 2026-11-12

- Sites no longer need a parked folder: wpsite runs `valet link` for each new site unless `SITES_DIR` is parked (`VALET_LINK=auto|1|0`). `delete` unlinks it.
- `SITES_DIR` is created if it doesn't exist.
- Fix for macOS's bash 3.2: `delete` failed with `db…: unbound variable` (also affected `--php` and multisite subsite messages).
- Fix for macOS's bash 3.2: demo import with images and subdirectory multisite could fail with "unbound variable" on an empty array.

## 1.1.2 — 2026-11-12

- Accept MariaDB's own client names (`mariadb` + `mariadb-check`), used by WP-CLI 2.12+, not only `mysql` + `mysqlcheck`.
- Docs: new Database setup section (MariaDB and MySQL via Homebrew, dev user for MariaDB).

## 1.1.1 — 2026-11-12

- Fix: `create` failing with `env: illegal option -- n` when the `mysql` client isn't on `PATH`. wpsite now finds it in common Homebrew (keg-only), DBngin, MAMP and MySQL-installer locations, or uses `MYSQL_BIN_DIR`.
- Fail early with a clear fix-it message before downloading WordPress if no MySQL client is found.

## 1.1.0 — 2026-11-12

- Multisite: `create --multisite[=subdomain]` builds a subdirectory or subdomain network.
- `--sites=3` / `--sites=shop,blog` (and `MULTISITE_SITES`) create subsites with the main site's theme and settings.
- Plugins and WooCommerce are network-activated on multisite; the theme is network-enabled and activated on every subsite.
- Subdirectory networks: the Valet driver they need is installed automatically if missing (`MULTISITE_DRIVER_AUTOINSTALL`, `MULTISITE_DRIVER_URL`).
- `--secure` covers subsites (subsite URLs use https).
- `demo --url=<subsite>` imports demo data into one subsite.
- `list` marks multisite networks.

## 1.0.0 — 2026-11-12

First release.

- `create`: WordPress download, config, database and install in one command.
- Options: `--woo`, `--theme`, `--plugins`, `--php` (valet isolate), `--wp`, `--secure`, `--no-open`.
- `--demo=unittest|woo|all` and `--no-images` on create, plus a `demo` command for existing sites.
- WooCommerce sample import creates global Color/Size attributes first so variable products keep their variations.
- `LINK_PLUGINS` / `LINK_THEMES`: symlink local source folders into every site.
- `delete`: removes SSL cert, PHP isolation, database and files; only for sites with a `.wpsite` marker.
- `list`, `open`, `config` commands.
- Rollback on failed create that never drops a pre-existing database.
