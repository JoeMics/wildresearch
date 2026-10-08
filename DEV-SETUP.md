# WildResearch: Dev Setup & CI/CD

_Last updated: 2026-10-07_

How to develop the WildResearch WordPress site locally, and how changes get to
staging and production. Background on why WordPress works this way is in
[`WORDPRESS-WORKFLOW.md`](WORDPRESS-WORKFLOW.md); DNS history is in
[`staging-site-dns-issue.md`](staging-site-dns-issue.md).

## Environments

| Env | URL | Where |
|---|---|---|
| Production | `https://wildresearch.ca` | Web Hosting Hub, cPanel, server `whub59` (`209.182.197.191`) |
| Staging | `https://stagingsite.wildresearch.ca` | Same cPanel account, separate WordPress install + DB |
| Local | `https://wildresearch.ddev.site` | DDEV on your machine |

`staging.wildresearch.ca` is dead. Use `stagingsite` only.

**Stack:** WordPress 7.0.7, PHP 8.4, Apache, MySQL/MariaDB, CloudLinux shared hosting.
**Theme:** custom classic theme `wp-content/themes/wildresearch` (the main code we own).
**Plugins seen:** Google Site Kit, MetaSlider (`ml-slider`), WPForms.

## The one rule

**Code flows up, content flows down.**

```
code:     local ──git──▶ staging ──▶ production
content:  production ──▶ staging ──▶ local      (DB + uploads)
```

Never push a database upward; editors' work on production would be overwritten.

## What's in git

```
wildresearch/                      ← repo (GitHub, new org account)
├── .ddev/                         ← local env config
├── .github/workflows/             ← CI/CD
├── web/wp-content/themes/wildresearch/   ← the theme (tracked)
├── scripts/
│   ├── pull-staging.sh            ← DB + files from staging → local
│   └── deploy.sh                  ← used by CI
├── plugins.json                   ← snapshot of `wp plugin list --format=json`
└── .gitignore
```

Not committed: WordPress core, third-party plugins, `uploads/`, `wp-config.php`,
`*.sql`, `node_modules/`, secrets.

## One-time setup

### 1. Server access (SSH)

cPanel → **SSH Access** → generate or import a key → **Authorize** it.
Web Hosting Hub usually uses port **2222** (confirm on the cPanel page).

```bash
ssh -p 2222 <cpanel-user>@209.182.197.191 'wp --info'   # is WP-CLI available?
```

If SSH is disabled, ask support to enable it. Without SSH, see
[Fallbacks](#fallbacks-without-ssh).

### 2. Local tools (Fedora)

- **DDEV** (works with Podman, which is already installed; Docker isn't needed).
  See ddev.readthedocs.io for the Podman setup.
- **Node 20+** for any theme build step (`nvm install 20`; current default is 18).

### 3. Create the local site

```bash
ddev config --project-type=wordpress --php-version=8.4 --docroot=web
ddev start
ddev wp core download
```

### 4. Pull staging down (`scripts/pull-staging.sh`)

```bash
HOST="-p 2222 <cpanel-user>@209.182.197.191"
STAGING_PATH="<path to stagingsite docroot>"

ssh $HOST "wp --path=$STAGING_PATH db export -" > tmp/staging.sql
ddev import-db --file=tmp/staging.sql
ddev wp search-replace 'https://stagingsite.wildresearch.ca' 'https://wildresearch.ddev.site' --all-tables
ddev wp search-replace 'http://stagingsite.wildresearch.ca'  'https://wildresearch.ddev.site' --all-tables
rsync -az -e "ssh -p 2222" "<cpanel-user>@209.182.197.191:$STAGING_PATH/wp-content/plugins/" web/wp-content/plugins/
rsync -az -e "ssh -p 2222" "<cpanel-user>@209.182.197.191:$STAGING_PATH/wp-content/uploads/" web/wp-content/uploads/
```

Content contains hard-coded `http://` links, hence both search-replace lines.
For `uploads/`, a full rsync is simplest; if it's large, skip it and proxy
missing images from staging instead.

Without WP-CLI on the server: export the DB from cPanel → phpMyAdmin, or use
a Softaculous backup, then `ddev import-db`.

## Day-to-day

```bash
ddev start                    # https://wildresearch.ddev.site
ddev wp <anything>            # WP-CLI locally
./scripts/pull-staging.sh     # refresh content
git switch -c feature/x       # work on the theme → PR → merge
```

## CI/CD (GitHub Actions)

The repo and all secrets live in the **new GitHub org**, not a personal
account. Add a second org owner.

Secrets/Environments: `staging` and `production` Environments, each with
`SSH_HOST`, `SSH_PORT`, `SSH_USER`, `SSH_KEY`, `THEME_PATH`. `production`
requires a reviewer.

### Tier 1: the basics

| Trigger | Job |
|---|---|
| Pull request | `php -l`, PHP_CodeSniffer (WordPress Coding Standards), build assets |
| Merge to `main` | rsync theme over SSH → staging |
| Release tag / manual | rsync theme → production (Environment approval required) |

### Tier 2: safer deploys

- **Atomic deploy + rollback:** upload to `releases/<git-sha>/`, then repoint a
  `themes/wildresearch` symlink. Rollback = repoint to the previous release.
- **Smoke test after deploy:** Playwright or curl checks that key pages return
  200 and contain expected text. Send a browser User-Agent; ModSecurity
  returns 406 to bare `curl`. Fail → auto-rollback.
- **Visual regression:** BackstopJS screenshots, staging vs production, on PRs.

### Tier 3: scheduled jobs

- **Weekly:** refresh staging DB from production.
- **Nightly:** production DB + uploads backup to off-site storage.
- **Drift check:** diff `wp plugin list` between staging and production; open an
  issue on mismatch (WordPress auto-updates in place).
- **Monitoring:** uptime, SSL expiry, DNS records (incl. SPF, MX).
- **Settings as code:** config changes as WP-CLI scripts run per environment,
  instead of clicking through wp-admin.

### Fallbacks without SSH

- **cPanel → Git Version Control** with a `.cpanel.yml` that copies the theme
  into place; CI pushes to that remote.
- **FTPS deploy action** (e.g. `SamKirkland/FTP-Deploy-Action`).

## Theme: classic vs block

The current `wildresearch` theme is a **classic theme**. The site does use the
**block editor** for content, but those are two different things:

| | What it is | WildResearch today |
|---|---|---|
| **Block editor** ("Gutenberg") | Editor for **page/post content** built from blocks | **Yes.** Pages are built from blocks |
| **Block theme** ("Full Site Editing") | Theme whose **header, footer, layouts and templates** are also blocks | **No.** Classic theme |

The block editor works with any theme, so this site has blocks *inside* the
content area, wrapped in a classic theme's header and footer.

```
Classic theme                          Block theme
─────────────                          ───────────
header.php    ← PHP, developer only    parts/header.html   ← blocks, editable in Site Editor
  [ content area: blocks ]               [ content area: blocks ]
footer.php    ← PHP, developer only    parts/footer.html   ← blocks, editable in Site Editor
style.css     ← all styling            theme.json + CSS    ← design tokens + styling
```

| | Classic theme (current) | Block theme |
|---|---|---|
| Templates | **PHP files** (`header.php`, `single.php`, …) | **HTML files** with block markup (`templates/single.html`) |
| Who changes header/footer/layout | Developer, in code | Admin in **Appearance → Editor**, or developer in code |
| Design settings | Hand-written CSS | `theme.json`, which also sets the editor's palette and font sizes |
| Menus | Appearance → Menus (here: page tree) | Navigation block |
| Customising | PHP + CSS | HTML + JSON + CSS; custom blocks in React |
| How to tell | No `theme.json`, has `header.php` | Has `theme.json` + `templates/` folder |
| Status | Supported, older approach | Current default (all core themes since 2022) |

**Why block fits this project:** templates are HTML and settings are JSON
(closer to a JS/HTML background than PHP); the client can adjust the
header, footer, blog layout and sections themselves; and the existing content
is already blocks, so it renders in a new block theme without migration.

**Caveat:** Site Editor changes are saved in the **database**, not the theme
files, so git doesn't see them. Once a template is edited in the Site Editor,
the database copy wins over your theme file, and a deploy can appear to "not
work" until the customisation is cleared with **Reset**. Lock down what editors
can change (`theme.json`, template locking, user roles).

### Conversion estimate (classic → block)

The current theme is small (~1,050 lines CSS, 34 lines JS, header/footer/
wrapper templates), so there's little to port. The work is rebuilding, not
converting.

| Task | Hours |
|---|---|
| Learning block themes (`theme.json`, templates, patterns) | 8–16 |
| `theme.json` + base styles (colours, Bitter / Open Sans, spacing) | 6–10 |
| Header/footer parts + Navigation (replace page-tree nav) | 4–8 |
| Templates: front page, page, single, blog archive, search, 404 | 8–12 |
| Styling core blocks to cover existing content | 6–10 |
| Replace MetaSlider / homepage gallery with core blocks | 3–6 |
| Patterns for editors + locking | 4–8 |
| QA: ~35 pages, ~50 posts, mobile, accessibility, WPForms | 8–12 |
| Launch: activate on staging, sign-off, production, redirects | 3–5 |
| **Like-for-like total** | **~50–90 h (≈1.5–2.5 weeks full-time)** |

Not included:
- Local env + CI/CD setup (above): ~8–12 h
- New blog design + the client's other changes: +20–60 h or more, depending on
  scope. Needs a design and change list first; this is the biggest unknown.
- Content cleanup (inline colours, `http://` links, old posts): varies.

### Switching themes: what moves and what doesn't

There's no content migration. Posts, pages, media, forms and plugin data stay
in the database and render under any theme. Anything that **belongs to the
theme** does need to be rebuilt or checked:

- [ ] **`functions.php`:** read it first. Anything registered there (custom
      post types, shortcodes, image sizes, filters) disappears when the theme
      is switched. Move what's still needed into a small plugin.
- [ ] **Navigation:** the page-tree nav becomes a Navigation block.
- [ ] **Hard-coded header/footer bits:** the `google-site-verification` meta
      tag, Google Fonts, footer links, the Donate button and the tagline. The
      verification is safe (the same value is in DNS as TXT), but move the rest.
- [ ] **Customizer settings and Additional CSS:** stored per theme. Copy them
      out before switching.
- [ ] **Widgets / sidebars:** not shown by block themes; rebuild in template parts.
- [ ] **Old posts (2010–2016):** probably classic HTML, not blocks. They still
      render, but need typography styles. Converting them to blocks is optional.
- [ ] **Default colour classes** (`has-cyan-bluish-gray-color`, etc.): keep
      WordPress's default palette enabled in `theme.json`, or that content loses its colours.
- [ ] **Image sizes:** if the new theme defines new sizes, regenerate thumbnails.
- [ ] **Page templates:** none assigned today, so nothing to remap.

Rehearse the switch on **local → staging** before production; switching back
to the old theme is one click as long as it stays installed.

## Housekeeping

- Staging hidden from search engines: Settings → Reading → "Discourage search
  engines" (already on: staging serves `noindex`) **and** cPanel → Directory
  Privacy (password).
- Disable the wp-admin theme/plugin file editor: `define('DISALLOW_FILE_EDIT', true);`
  in `wp-config.php`, so nobody edits live code outside git.
- **Flagged:** production has no SPF record (see DNS notes).

## Open items

- [ ] Create GitHub org + repo; add second owner
- [ ] Confirm SSH access + port, and whether WP-CLI exists on the server
- [ ] Find staging + production docroot paths in cPanel
- [ ] Install DDEV (Podman) and Node 20
- [ ] Download the theme, `git init`, first commit
- [ ] Write `pull-staging.sh` and the Tier 1 workflows
