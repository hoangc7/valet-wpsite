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
- [Multisite](#multisite)
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
- **Plugin/theme development:** your local source folders, single ones or a whole folder of them, are symlinked (not copied) into every new site, so one checkout is tested across many sites.
- **Multisite:** `--multisite` builds a subdirectory or subdomain network with test subsites, and sets up the Valet driver it needs.
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
| MySQL or MariaDB | Running, with a user that can create and drop databases (see [Database setup](#database-setup)). The client commands are needed too (`mysql` + `mysqlcheck`, or `mariadb` + `mariadb-check` with WP-CLI 2.12+). wpsite finds them in common keg-only Homebrew, DBngin and MySQL-installer locations; see [Troubleshooting](#troubleshooting). |
| `curl`, `bash` 3.2+ | Preinstalled on macOS and Linux. |

Quick setup on macOS with Homebrew:

```bash
brew install php wp-cli composer
composer global require laravel/valet
valet install
```

You don't need to `valet park` anything. wpsite links each site it creates (`valet link`). If you'd rather park your sites folder instead, see `VALET_LINK` under [Configuration](#configuration).

Then set up a database server, as described below.

### Database setup

Pick the server your production sites run. Matching production avoids surprises when you move databases between local and live. For example, MySQL 8's default `utf8mb4_0900_ai_ci` collation doesn't exist in MariaDB, so a dump from local MySQL fails to import on a MariaDB host. You don't need DBngin or any GUI; Homebrew is enough.

**MariaDB**

```bash
brew install mariadb
brew services start mariadb
```

Homebrew's MariaDB lets `root` (and your macOS user) log in only through the local socket, as that same system user, so wpsite's default `root` over `127.0.0.1` is rejected. Create a dev user once:

```bash
mariadb -e "CREATE USER 'wp'@'localhost' IDENTIFIED BY 'wp'; GRANT ALL PRIVILEGES ON *.* TO 'wp'@'localhost'; FLUSH PRIVILEGES;"
```

and set it in `wpsite config`:

```bash
DB_USER="wp"
DB_PASS="wp"
DB_HOST="localhost"     # use the local socket
```

**MySQL**

```bash
brew install mysql
brew services start mysql
```

`root` with no password on `127.0.0.1` works out of the box, matching wpsite's defaults, so no config changes are needed.

Check either one with `wpsite create test && wpsite delete test -y`.

## Installation

Clone the repo, then symlink the script into any folder that's already on your `PATH`:

```bash
git clone https://github.com/hoangc7/valet-wpsite.git ~/.valet-wpsite
mkdir -p ~/.local/bin
ln -s ~/.valet-wpsite/wpsite ~/.local/bin/wpsite   # or any other folder on your PATH
```

If `wpsite` isn't found, `~/.local/bin` isn't on your `PATH` yet (macOS doesn't add it by default). Add it to `~/.zshrc` or `~/.bashrc`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
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
| `SITES_DIR` | `$HOME/Sites` | Where site folders are created (created if missing). Every site is a sub-folder. |
| `VALET_LINK` | `auto` | How Valet serves new sites. `auto`: `valet link` each site, unless `SITES_DIR` is already parked. `1`: always link. `0`: never link (you serve the folder yourself). |
| `TLD` | *(auto)* | Site domain suffix. Empty = read from `valet tld`, falling back to `test`. |
| `DB_USER` | `root` | MySQL user. |
| `DB_PASS` | *(empty)* | MySQL password. |
| `DB_HOST` | `127.0.0.1` | MySQL host. |
| `DB_PREFIX` | `wp_` | Database name prefix. Site `my-shop` → database `wp_my_shop`. |
| `MYSQL_BIN_DIR` | *(auto)* | Folder containing `mysql` and `mysqlcheck`, only needed if they aren't on `PATH` and aren't auto-detected. |
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
| `MULTISITE_SITES` | `0` | Subsites created with `--multisite`: `0` = none, a number (`site1`, `site2`…) or names (`"shop,blog"`). |
| `MULTISITE_DRIVER_AUTOINSTALL` | `1` | Install the Valet driver that subdirectory multisite needs, if it's missing. `0` = only warn. |
| `LINK_PLUGINS` | *(empty)* | Space-separated local plugin folders symlinked into every site. |
| `LINK_THEMES` | *(empty)* | Space-separated local theme folders symlinked into every site. |
| `LINK_PLUGINS_DIR` | *(empty)* | Folder(s) of plugins. Every plugin inside is symlinked into every site (not activated). |
| `LINK_THEMES_DIR` | *(empty)* | Folder(s) of themes. Every theme inside is symlinked into every site. |

Advanced (rarely needed):

| Variable | Default | Description |
| --- | --- | --- |
| `CACHE_DIR` | `~/.cache/wpsite` | Where downloaded demo files are cached. |
| `UNITTEST_XML_URL` | WPTT theme-unit-test on GitHub | Source of the Theme Unit Test XML. |
| `WOO_XML_URL` | WooCommerce repo on GitHub | Fallback source of the WooCommerce sample XML. |
| `MULTISITE_DRIVER_URL` | [hoangc7/WordPressMultisiteSubdirectoryValetDriver](https://github.com/hoangc7/WordPressMultisiteSubdirectoryValetDriver) | Where the subdirectory multisite driver is downloaded from. |
| `VALET_HOME` | `~/.config/valet` | Valet's config folder (the driver goes into `Drivers/` inside it). |

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
| `--multisite[=subdomain]` | Create a WordPress multisite network. Default is subdirectory (`site.test/sub/`); `=subdomain` gives `sub.site.test`. See [Multisite](#multisite). |
| `--sites=<n\|a,b>` | Subsites to create with `--multisite`: a count (`--sites=3`) or names (`--sites=shop,blog`). Overrides `MULTISITE_SITES`. |
| `--theme=<x>` | Install and activate a theme. `x` can be a wordpress.org slug, a `.zip` path/URL, or a local folder (symlinked). |
| `--plugins="a b c"` | Extra wordpress.org plugins to install and activate. |
| `--php=<version>` | Run this site on a specific PHP version (`valet isolate php@8.2`). The version must be installed via Homebrew. |
| `--wp=<version>` | Install a specific WordPress version, e.g. `--wp=6.6`. |
| `--secure` / `--no-secure` | Enable/disable HTTPS (overrides `SECURE`). |
| `--no-open` | Don't open the browser afterwards. |

What happens, in order:

1. Validate the name; refuse if the folder or database already exists.
   (Steps 2–8 build the site; step 9 makes Valet serve it.)
2. Download WordPress and write `wp-config.php`.
3. Create the database and install WordPress (or a multisite network).
4. Set `/%postname%/` permalinks and discourage search engines.
5. Install WooCommerce, plugins, linked plugins/themes and the chosen theme.
6. Multisite only: create the subsites.
7. Remove default content (if `REMOVE_DEFAULTS=1`).
8. Import demo data (if requested).
9. `valet link` the site (unless `SITES_DIR` is parked), then isolate PHP and secure with HTTPS (if requested); install the multisite driver if needed.
10. Print the URLs and credentials, open wp-admin.

If any step before demo import fails, the site is rolled back completely. A failed demo import keeps the site and prints the command to retry.

### demo

```bash
wpsite demo <name> <unittest|woo|all> [--no-images] [--url=<subsite>]
```

`--url` targets one subsite of a multisite network. It accepts the subsite name (`shop`), its domain (`shop.network.test`) or its full URL. Without it, data goes into the main site.

Imports demo data into an existing site (it doesn't need to have been created by wpsite, it just needs a `wp-config.php`). WooCommerce is installed or activated automatically for `woo`. See [Demo data](#demo-data).

### delete

```bash
wpsite delete <name> [-y]
```

Shows what will be removed and asks for confirmation (`-y` skips it). Then:

1. Removes the HTTPS certificate if the site is secured.
2. Removes the PHP isolation and the Valet link.
3. Drops the database.
4. Deletes the site folder. Symlinked plugins/themes are removed as links only. Your source folders are untouched.

Run without a name to pick from the list.

### list

Lists every site in `SITES_DIR` created by wpsite, with its URL (`https://` if secured) and whether it's a multisite network.

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

## Multisite

```bash
wpsite create network --multisite                         # just the network (main site), no subsites
wpsite create network --multisite --sites=2               # subdirectory: network.test/site1/, network.test/site2/
wpsite create network --multisite=subdomain --sites=shop,blog   # subdomain: shop.network.test, blog.network.test
```

What you get:

- A network installed with `wp core multisite-install`, so `MULTISITE`, `SUBDOMAIN_INSTALL` and related constants are written to `wp-config.php`.
- Subsites only if you ask for them with `--sites` (or set `MULTISITE_SITES`), each with the main site's theme, pretty permalinks, search engines discouraged, and default content removed.
- Plugins (`--plugins`, `DEFAULT_PLUGINS`, `LINK_PLUGINS`, WooCommerce) are **network-activated**. The theme is network-enabled and activated on every subsite.
- `--secure` covers the whole network. Valet's certificate includes `*.network.test`, and subsite URLs are switched to `https://`.
- The summary prints the Network Admin URL and every subsite URL. `wpsite list` marks the site as `[multisite: subdirectory]` or `[multisite: subdomain]`.

**Subdomain vs subdirectory on Valet**

| | Subdomain | Subdirectory |
| --- | --- | --- |
| URLs | `shop.network.test` | `network.test/shop/` |
| Valet setup | None. Valet already routes `*.network.test` to the `network` folder. | Needs a custom Valet driver (below). |
| HTTPS | Covered by one `valet secure`. | Covered by one `valet secure`. |
| Main site permalinks | `/%postname%/` | `/blog/%postname%/`. WordPress adds `/blog/` on subdirectory networks so main-site posts can't clash with subsite paths. |

**The subdirectory driver.** Valet's built-in WordPress driver doesn't understand subdirectory multisite paths, so `/site1/wp-admin/` and subsite assets would break. When you create a subdirectory network, wpsite checks for `WordPressMultisiteSubdirectoryValetDriver.php` in `~/.config/valet/Drivers/` and downloads it if it's missing (from `MULTISITE_DRIVER_URL`). This happens once and is shared by all sites. It only changes behaviour for sites whose `wp-config.php` defines `MULTISITE` as true; single sites are served exactly as before. Set `MULTISITE_DRIVER_AUTOINSTALL=0` to install it yourself.

> The driver uses the Valet 4 driver format. On Valet Linux or older Valet versions, install a matching driver manually.

**Demo data on a network:**

```bash
wpsite create network --multisite --sites=shop,blog --demo=unittest   # demo goes into the main site
wpsite demo network woo --url=shop                                     # WooCommerce products on the shop subsite
wpsite demo network unittest --url=blog
```

With `--url`, WooCommerce is activated only on that subsite (unless it's already network-active).

## Developing plugins and themes

Point wpsite at your working copies, either one by one or a whole folder at a time:

```bash
# individual plugins/themes: linked, and plugins are activated
LINK_PLUGINS="$HOME/Code/my-plugin"
LINK_THEMES="$HOME/Code/my-theme"

# whole folders: every plugin/theme inside is linked (plugins are NOT activated)
LINK_PLUGINS_DIR="$HOME/Code/plugins"
LINK_THEMES_DIR="$HOME/Code/themes"
```

Every new site gets a symlink, for example `wp-content/plugins/my-plugin -> ~/Code/my-plugin`. Edit once and test everywhere: different PHP versions, WordPress versions, with and without WooCommerce, with different demo data.

**Folders of plugins and themes**

- Only real plugins and themes are linked. A plugin is a sub-folder with a top-level PHP file containing a `Plugin Name:` header; a theme is a sub-folder whose `style.css` has a `Theme Name:` header. Everything else (`node_modules`, `.git`, loose files, notes) is ignored.
- Plugins from a folder are **linked but not activated**, because a folder often holds many plugins, sometimes a free and a Pro version that can't be active together. Activate what you need in wp-admin, or also list the plugin you're working on in `LINK_PLUGINS` and it's linked once and activated on every new site.
- Several folders can be listed, separated by spaces. If two of your folders contain a plugin or theme with the same folder name, the first one wins and the other is skipped with a warning.
- If one of your plugins or themes has the same name as a copy WordPress ships with (for example, your fork of `twentytwentyfour` or `akismet`), the stock copy on the new site is replaced by your linked one.
- Only sites created *after* you change these settings get the links. To add a plugin to an existing site, symlink it yourself: `ln -s ~/Code/plugins/new-plugin ~/Sites/<site>/wp-content/plugins/`.

**Your source is never deleted.** wpsite never deletes a symlinked plugin or theme, including during the default-content cleanup. WordPress's own delete would follow the link and erase the files inside, so wpsite skips those. Deleting a site removes the links only.

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
- A `.wpsite` marker file in the site root (see [Safety](#safety)). It also records whether the site is a multisite network.

## Safety

Deleting things is the risky part of a script like this, so wpsite is deliberately strict:

- **Marker file:** `create` writes `.wpsite` into the site folder. `delete` refuses any folder without it, so it can't remove a project you set up by hand.
- **Strict names:** only `a-z`, `0-9` and `-`. Empty names, `..` and slashes are rejected before anything touches the disk.
- **No overwrite:** `create` stops if the folder or database already exists, instead of wiping it.
- **Safe rollback:** if `create` fails, it removes the new folder and drops the database only if this run created it.
- **Symlinks:** deleting a site removes the links, never your plugin or theme source. The default-content cleanup also skips anything symlinked.

## Recipes

```bash
# Test a plugin on old PHP + old WordPress
wpsite create legacy --php=7.4 --wp=6.2 --woo

# WooCommerce store with HTTPS (needed for some payment gateways / Apple Pay testing)
wpsite create pay --demo=woo --secure

# Theme review setup
wpsite create review --demo=unittest --plugins="theme-check query-monitor"

# Multisite network for testing a plugin per-site vs network-wide
wpsite create network --multisite --sites=3 --plugins="query-monitor"

# WooCommerce on one subsite of a subdomain network
wpsite create stores --multisite=subdomain --sites=us,eu --secure
wpsite demo stores woo --url=us

# Throwaway site, no browser, delete when done
wpsite create tmp --no-open && wpsite delete tmp -y

# Install a premium theme from a zip
wpsite create client --theme=~/Downloads/theme.zip --woo

# Debug the script itself
bash -x wpsite create debug-site
```

## Troubleshooting

**`Access denied for user 'root'@'localhost'` with MariaDB.**
Homebrew's MariaDB doesn't allow password-less `root` over TCP. Create a dev user; see [Database setup](#database-setup).

**`env: illegal option -- n` (or `env: 'mysql': No such file or directory`) at "Creating database".**
Either no database server is installed (see [Database setup](#database-setup)), or the server is running but its `mysql` client command isn't on your `PATH`. WP-CLI looks it up with `which mysql`, and when that fails it runs `/usr/bin/env --no-defaults …`, which macOS's `env` rejects. This is common with keg-only Homebrew formulas (`mysql@8.0`, `mysql@8.4`, `mysql-client`), DBngin and the official MySQL installer.

wpsite checks the usual locations automatically (`/opt/homebrew/opt/mysql*/bin`, `/usr/local/opt/mysql*/bin`, `/usr/local/mysql/bin`, DBngin, MAMP) and prints `Using MySQL client from …` when it finds one. If it still can't find one, pick one fix:

```bash
which mysql mysqlcheck                       # should print two paths
brew link --force mysql@8.4                  # Homebrew versioned formula
brew install mysql-client                    # client only, if the server comes from elsewhere
echo 'export PATH="/opt/homebrew/opt/mysql@8.4/bin:$PATH"' >> ~/.zshrc   # or put it on PATH yourself
```

Or point wpsite at it in `wpsite config`:

```bash
MYSQL_BIN_DIR="/opt/homebrew/opt/mysql@8.4/bin"
```

**`Error establishing a database connection` / `Can't connect to server`.**
MySQL isn't running or the credentials are wrong. Run `brew services start mysql`, then check `DB_USER` / `DB_PASS` / `DB_HOST`. Try `127.0.0.1` vs `localhost`.

**Site shows "404 - Not Found" or doesn't resolve.**
Valet isn't serving the folder. With the default `VALET_LINK=auto`, wpsite links each site, so check `valet links` lists it. If it doesn't (for example, a site created with version 1.1.2 or older, which only worked with parked folders), link it yourself: `cd ~/Sites/<name> && valet link <name>`. If you set `VALET_LINK=0`, `SITES_DIR` must be parked (`valet paths`). Confirm the TLD with `valet tld`.

**`line …: db…: unbound variable` on macOS.**
Fixed in 1.1.3. macOS's built-in bash 3.2 read the `…` after a variable name as part of the name. Update with `git -C ~/.valet-wpsite pull`.

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

**Subdirectory multisite: subsites or `/site1/wp-admin/` return 404 or show the main site.**
The multisite driver isn't installed. Check that `~/.config/valet/Drivers/WordPressMultisiteSubdirectoryValetDriver.php` exists; if not, download it from `MULTISITE_DRIVER_URL`. Also confirm `wp-config.php` contains `define( 'MULTISITE', true );`.

**Subdomain multisite: `shop.network.test` doesn't resolve.**
Check that Valet's DNS is running (`valet restart`) and that `ping shop.network.test` resolves to `127.0.0.1`.

**"No subsite matching …" from `wpsite demo --url`.**
The error lists the network's subsite URLs; use one of those names.

**Running as root (Docker/CI).**
Export `WP_CLI_ALLOW_ROOT=1`.

## Uninstall

```bash
rm ~/.local/bin/wpsite              # wherever you linked it
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
- [WordPress Multisite Subdirectory Valet Driver](https://github.com/Objectivco/WordPressMultisiteSubdirectoryValetDriver) by Objectiv, used for subdirectory networks ([fork used by default](https://github.com/hoangc7/WordPressMultisiteSubdirectoryValetDriver))

## License

Released under the [MIT License](LICENSE).
