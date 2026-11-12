# valet-wpsite

`wpsite` is a command-line tool that creates and deletes local WordPress sites in seconds with [Laravel Valet](https://laravel.com/docs/valet) and [WP-CLI](https://wp-cli.org/), with optional WooCommerce and demo data.

```bash
wpsite create shop --demo=woo      # WordPress + WooCommerce + sample products, ready in the browser
wpsite delete shop                 # files, database and SSL certificate gone
```

Inspired by [ValetPress](https://github.com/sdenike/valetpress) by Shelby DeNike, itself based on the original [ValetPress](https://github.com/AaronRutley/valetpress) by Aaron Rutley. wpsite is a rewrite of that idea with safer create/delete, per-site PHP versions, symlinked development plugins and built-in demo data. See [Credits](#credits).

---

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Commands](#commands)
  - [create](#create)
  - [demo](#demo)
  - [delete](#delete)
  - [list](#list)
  - [open](#open)
  - [config](#config)
- [Demo data](#demo-data)
- [Developing plugins and themes](#developing-plugins-and-themes)
- [What a new site looks like](#what-a-new-site-looks-like)
- [Safety](#safety)
- [Recipes](#recipes)
- [Troubleshooting](#troubleshooting)
- [Uninstall](#uninstall)
- [Credits](#credits)
- [License](#license)

---

## Features

- **One command to a working site:** downloads WordPress, creates the database, installs, sets permalinks and opens wp-admin.
- **WooCommerce ready:** `--woo` installs WooCommerce and skips the onboarding wizard and "coming soon" mode.
- **Demo data:** the WordPress Theme Unit Test data and/or WooCommerce sample products, with images or without.
- **Plugin/theme development:** your local source folders are symlinked (not copied) into every new site, so one checkout is tested across many sites.
- **Per-site PHP:** `--php=8.2` runs `valet isolate` for that site only.
- **HTTPS:** `--secure` runs `valet secure`; delete removes the certificate.
- **Safe delete:** only removes sites wpsite created, and never touches your symlinked source folders.
- **Clean rollback:** a failed create removes the half-built site and never drops a database it didn't create.
- **No extra dependencies:** plain bash, configured with a plain bash file. No `jq`, no Composer package.

## Requirements

| Tool | Notes |
| --- | --- |
| [Laravel Valet](https://laravel.com/docs/valet) | macOS. [Valet Linux](https://cpriego.github.io/valet-linux/) also works. Without Valet, wpsite still builds the site; you just serve the folder yourself. |
| [WP-CLI](https://wp-cli.org/#installing) | `wp` must be on your `PATH`. |
| MySQL or MariaDB | Running, with a user that can create and drop databases. |
| `curl`, `bash` 3.2+ | Preinstalled on macOS and Linux. |

Quick setup on macOS with Homebrew:

```bash
brew install php mysql wp-cli composer
brew services start mysql
composer global require laravel/valet
valet install
mkdir -p ~/Sites && cd ~/Sites && valet park
```

## Installation

Clone the repo, then symlink the script into any folder that's already on your `PATH`:

```bash
git clone https://github.com/hoangc7/valet-wpsite.git ~/.valet-wpsite
ln -s ~/.valet-wpsite/wpsite ~/bin/wpsite        # or ~/.home/bin, /usr/local/bin, …
```

No folder on your `PATH` yet? Create one and add it to `~/.zshrc` or `~/.bashrc`:

```bash
mkdir -p ~/bin
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Check it works with `wpsite --version`. A symlink (rather than a copy) means updates arrive with a `git pull`.

Then create your config:

```bash
wpsite config
```

Update later with:

```bash
git -C ~/.valet-wpsite pull
```

## Configuration

`wpsite config` creates `~/.config/wpsite/config` (or `$XDG_CONFIG_HOME/wpsite/config`) and opens it in `$EDITOR`. It is a plain bash file sourced on every run; a copy lives in [`config.sample`](config.sample).

| Variable | Default | Description |
| --- | --- | --- |
| `SITES_DIR` | `$HOME/Sites` | Folder you ran `valet park` in. Every site is a sub-folder. |
| `TLD` | *(auto)* | Site domain suffix. Empty = read from `valet tld`, falling back to `test`. |
| `DB_USER` | `root` | MySQL user. |
| `DB_PASS` | *(empty)* | MySQL password. |
| `DB_HOST` | `127.0.0.1` | MySQL host. |
| `DB_PREFIX` | `wp_` | Database name prefix. Site `my-shop` → database `wp_my_shop`. |
| `ADMIN_USER` | `admin` | WordPress admin username. |
| `ADMIN_PASS` | `password` | WordPress admin password. |
| `ADMIN_EMAIL` | `admin@example.test` | WordPress admin email. |
| `SECURE` | `0` | `1` = HTTPS for every new site (override with `--no-secure`). |
| `OPEN_BROWSER` | `1` | Open wp-admin after create. |
| `REMOVE_DEFAULTS` | `1` | Remove Hello Dolly, Akismet, inactive Twenty* themes and the sample post/page. |
| `WP_DEBUG` | `1` | Sets `WP_DEBUG` and `SCRIPT_DEBUG`. |
| `DEFAULT_PLUGINS` | *(empty)* | wordpress.org plugin slugs installed and activated on every site. |
| `WOO_PLUGINS` | *(empty)* | Extra plugins installed only on WooCommerce sites. |
| `DEFAULT_DEMO` | *(empty)* | Demo data for every new site: `unittest`, `woo` or `all`. |
| `DEMO_IMAGES` | `1` | `0` = skip demo images by default. |
| `LINK_PLUGINS` | *(empty)* | Space-separated local plugin folders symlinked into every site. |
| `LINK_THEMES` | *(empty)* | Space-separated local theme folders symlinked into every site. |

Advanced (rarely needed):

| Variable | Default | Description |
| --- | --- | --- |
| `CACHE_DIR` | `~/.cache/wpsite` | Where downloaded demo files are cached. |
| `UNITTEST_XML_URL` | WPTT theme-unit-test on GitHub | Source of the Theme Unit Test XML. |
| `WOO_XML_URL` | WooCommerce repo on GitHub | Fallback source of the WooCommerce sample XML. |

Example config for plugin development:

```bash
SITES_DIR="$HOME/Sites"
DB_USER="root"
DB_PASS=""
DEFAULT_PLUGINS="query-monitor"
LINK_PLUGINS="$HOME/Code/my-plugin $HOME/Code/my-other-plugin"
DEMO_IMAGES=0
```

## Commands

```
wpsite create <name> [options]   Create a new site
wpsite demo <name> <type>        Import demo data into an existing site
wpsite delete <name> [-y]        Delete a site
wpsite list                      List sites created by wpsite
wpsite open <name>               Open a site's wp-admin
wpsite config                    Create/edit the config file
wpsite help                      Show help
wpsite --version                 Show version
```

Short aliases: `new`/`c` = create, `rm`/`d` = delete, `ls`/`l` = list, `o` = open.

### create

```bash
wpsite create <name> [options]
```

Site names use lowercase letters, digits and hyphens (`my-shop`, `test2`). Run without a name to be prompted.

| Option | Description |
| --- | --- |
| `--woo` | Install and activate WooCommerce (wizard skipped) plus `WOO_PLUGINS`. |
| `--demo=<type>` | Import demo data: `unittest`, `woo`, `all`, or a comma list like `unittest,woo`. `woo` implies `--woo`. `--demo` alone = `all`. |
| `--no-demo` | Ignore `DEFAULT_DEMO` for this site. |
| `--no-images` | Skip demo images (much faster, works offline). |
| `--theme=<x>` | Install and activate a theme. `x` can be a wordpress.org slug, a `.zip` path/URL, or a local folder (symlinked). |
| `--plugins="a b c"` | Extra wordpress.org plugins to install and activate. |
| `--php=<version>` | Run this site on a specific PHP version (`valet isolate php@8.2`). The version must be installed via Homebrew. |
| `--wp=<version>` | Install a specific WordPress version, e.g. `--wp=6.6`. |
| `--secure` / `--no-secure` | Enable/disable HTTPS (overrides `SECURE`). |
| `--no-open` | Don't open the browser afterwards. |

What happens, in order:

1. Validate the name; refuse if the folder or database already exists.
2. Download WordPress and write `wp-config.php`.
3. Create the database and install WordPress.
4. Set `/%postname%/` permalinks and discourage search engines.
5. Install WooCommerce, plugins, linked plugins/themes and the chosen theme.
6. Remove default content (if `REMOVE_DEFAULTS=1`).
7. Import demo data (if requested).
8. Isolate PHP and secure with HTTPS (if requested).
9. Print the URL and credentials, open wp-admin.

If any step before demo import fails, the site is rolled back completely. A failed demo import keeps the site and prints the command to retry.

### demo

```bash
wpsite demo <name> <unittest|woo|all> [--no-images]
```

Imports demo data into an existing site (it doesn't need to have been created by wpsite, it just needs a `wp-config.php`). WooCommerce is installed or activated automatically for `woo`. See [Demo data](#demo-data).

### delete

```bash
wpsite delete <name> [-y]
```

Shows what will be removed and asks for confirmation (`-y` skips it). Then:

1. Removes the HTTPS certificate if the site is secured.
2. Removes the PHP isolation and any Valet link.
3. Drops the database.
4. Deletes the site folder. Symlinked plugins/themes are removed as links only. Your source folders are untouched.

Run without a name to pick from the list.

### list

Lists every site in `SITES_DIR` created by wpsite, with its URL (`https://` if secured).

### open

Opens the site's wp-admin in your default browser.

### config

Creates `~/.config/wpsite/config` on first run, then opens it in `$EDITOR` (default `nano`).

## Demo data

Two datasets, usable on their own or together:

| Type | What you get | Good for |
| --- | --- | --- |
| `unittest` | The official [WordPress Theme Unit Test](https://github.com/WPTT/theme-unit-test) data: ~58 posts, ~22 pages, 6 menus, ~68 categories, 2 extra authors. Includes post formats, long titles, nested pages, comments, sticky posts, image alignment, non-Latin text, and many markup edge cases. | Theme development and review, block/markup styling, menus. |
| `woo` | WooCommerce's own [sample products](https://github.com/woocommerce/woocommerce/tree/trunk/plugins/woocommerce/sample-data): 18 products (14 simple, 2 variable, 1 grouped, 1 external) with categories, prices, sale prices, SKUs and global Color/Size attributes. | Shop pages, product templates, variation/swatch UIs, cart and checkout. |

```bash
wpsite create shop --demo=woo                  # new WooCommerce site with products
wpsite create theme-dev --demo=unittest        # new site with Theme Unit Test data
wpsite create full --demo=all --no-images      # both, fast, no images
wpsite demo existing-site woo                  # add products to an existing site
```

How it works:

- The [WordPress Importer](https://wordpress.org/plugins/wordpress-importer/) plugin is installed automatically and used via `wp import --authors=create`.
- The unit-test XML is downloaded once and cached in `CACHE_DIR`. The WooCommerce XML is read from the plugin's own `sample-data/` folder, falling back to GitHub.
- **Global attributes:** the WooCommerce sample file uses global attributes (`pa_color`, `pa_size`). WooCommerce only registers them during an import started from wp-admin, so a plain `wp import` silently drops them. wpsite creates the Color and Size attributes before importing, so the variable products keep their variations.
- After a Woo import, product lookup tables and transients are refreshed.
- The first imported menu is assigned to the theme's first menu location (classic themes only).

**Images.** Demo images are downloaded from their original hosts (wordpress.com and woocommerce.com) during import. That is the slow part. Use `--no-images` (or `DEMO_IMAGES=0`) to skip them: the import takes seconds and works offline, but products have no photos. Images that fail to download are reported as a warning with the log path, `/tmp/wpsite-import.log`.

Re-running an import on the same site is safe: the importer skips posts that already exist, so content isn't duplicated.

## Developing plugins and themes

Point `LINK_PLUGINS` / `LINK_THEMES` at your working copies:

```bash
LINK_PLUGINS="$HOME/Code/my-plugin"
LINK_THEMES="$HOME/Code/my-theme"
```

Every new site gets a symlink, for example `wp-content/plugins/my-plugin -> ~/Code/my-plugin`, and linked plugins are activated. Edit once and test everywhere: different PHP versions, WordPress versions, with and without WooCommerce, with different demo data.

For a one-off theme, pass a folder to `--theme`:

```bash
wpsite create test-theme --theme=~/Code/my-theme --demo=unittest
```

> If your plugin needs a build step (Composer/npm), build it in the source folder. Every site sees the result immediately.

## What a new site looks like

- `wp-config.php` with `WP_ENVIRONMENT_TYPE=local`, `FS_METHOD=direct`, `WP_DEBUG`, `WP_DEBUG_LOG` (to `wp-content/debug.log`), `WP_DEBUG_DISPLAY=false` and `SCRIPT_DEBUG`.
- Admin user from your config. Install emails are skipped.
- Pretty permalinks (`/%postname%/`), search engines discouraged.
- Latest bundled default theme active, other Twenty* themes removed.
- No Hello Dolly, no Akismet, no "Hello world!" post or "Sample Page".
- A `.wpsite` marker file in the site root (see [Safety](#safety)).

## Safety

Deleting things is the risky part of a script like this, so wpsite is deliberately strict:

- **Marker file:** `create` writes `.wpsite` into the site folder. `delete` refuses any folder without it, so it can't remove a project you set up by hand.
- **Strict names:** only `a-z`, `0-9` and `-`. Empty names, `..` and slashes are rejected before anything touches the disk.
- **No overwrite:** `create` stops if the folder or database already exists, instead of wiping it.
- **Safe rollback:** if `create` fails, it removes the new folder and drops the database only if this run created it.
- **Symlinks:** deleting a site removes the links, never your plugin or theme source.

## Recipes

```bash
# Test a plugin on old PHP + old WordPress
wpsite create legacy --php=7.4 --wp=6.2 --woo

# WooCommerce store with HTTPS (needed for some payment gateways / Apple Pay testing)
wpsite create pay --demo=woo --secure

# Theme review setup
wpsite create review --demo=unittest --plugins="theme-check query-monitor"

# Throwaway site, no browser, delete when done
wpsite create tmp --no-open && wpsite delete tmp -y

# Install a premium theme from a zip
wpsite create client --theme=~/Downloads/theme.zip --woo

# Debug the script itself
bash -x wpsite create debug-site
```

## Troubleshooting

**`Error establishing a database connection` / `Can't connect to server`.**
MySQL isn't running or the credentials are wrong. Run `brew services start mysql`, then check `DB_USER` / `DB_PASS` / `DB_HOST`. Try `127.0.0.1` vs `localhost`.

**Site shows "404 - Not Found" or doesn't resolve.**
`SITES_DIR` must be a folder you ran `valet park` in. Check with `valet paths`. Confirm the TLD with `valet tld`.

**`--php=8.x` fails.**
Install that version first: `brew install php@8.2`.

**`valet secure` asks for a password.**
Valet needs `sudo` to trust certificates. That's expected.

**Demo import is slow.**
That's image downloads. Use `--no-images`.

**Variable products have no attributes after a manual `wp import`.**
Use `wpsite demo <site> woo`, which creates the global attributes first (see [Demo data](#demo-data)).

**"wasn't created by wpsite (no .wpsite file). Refusing to delete."**
The folder wasn't made by wpsite. If you're sure, add the marker (`touch ~/Sites/name/.wpsite`) and run delete again, or remove it by hand.

**Running as root (Docker/CI).**
Export `WP_CLI_ALLOW_ROOT=1`.

## Uninstall

```bash
rm ~/bin/wpsite                     # wherever you linked it
rm -rf ~/.valet-wpsite ~/.config/wpsite ~/.cache/wpsite
```

Your sites and databases are left untouched. Delete them first with `wpsite delete` if you want them gone.

## Credits

This project wouldn't exist without **ValetPress**:

- **[ValetPress](https://github.com/sdenike/valetpress) by [Shelby DeNike](https://github.com/sdenike)**: the direct inspiration. Its `vp create` / `vp delete` workflow, config-driven defaults and WooCommerce/theme install options shaped how wpsite works.
- **[ValetPress](https://github.com/AaronRutley/valetpress) by [Aaron Rutley](https://github.com/AaronRutley)**: the original experiment that showed a full WordPress site could be created with Valet and WP-CLI in a few seconds.

wpsite is an independent rewrite; no code was copied. Thank you both for sharing your work.

Also built on:

- [Laravel Valet](https://github.com/laravel/valet) and [WP-CLI](https://github.com/wp-cli/wp-cli)
- [Theme Unit Test data](https://github.com/WPTT/theme-unit-test) by the WordPress Themes Team
- [WooCommerce sample data](https://github.com/woocommerce/woocommerce/tree/trunk/plugins/woocommerce/sample-data)
- [WordPress Importer](https://github.com/WordPress/wordpress-importer)

## License

Released under the [MIT License](LICENSE).
