# Changelog

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
