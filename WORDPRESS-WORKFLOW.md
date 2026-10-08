# Working on the Inherited WordPress Site

Notes on how this site actually works and how to set up a local dev + versioning
workflow. Written for a developer coming from a JS/framework background
(Next.js, Vercel, Azure), so it's framed against those conventions.

## TL;DR

- Only the **theme** (and any custom plugin) is yours to version. Core,
  third-party plugins, and `wp-content/uploads` are **not** committed.
- The **database is half the application** — content, menus, settings live in
  MySQL, not in files. You cannot version the site through code alone.
- There is **no deployment pipeline**. The live server *is* the source of truth.
  Your job is to introduce the discipline (git repo + local env + a deploy step)
  that this site has never had.

---

## How "deployment" works here (vs. Vercel / Netlify / Azure)

### The model you know

```
git push → build pipeline → immutable artifact → atomic deploy → CDN/serverless
```

- Code is the source of truth; the running site is a *product* of the repo.
- Compute is ephemeral and stateless; state lives in a managed DB elsewhere.
- You never touch a server. "Deploy" = platform rebuilds from latest commit and
  atomically swaps it in. Rollback = re-point to a previous build.
- "Cattle" model — servers are disposable, rebuilt from definition.

### The model this WordPress site uses

```
A single Linux box runs Apache + PHP + MySQL, 24/7, forever.
The "application" = a folder of .php files on its disk + a MySQL database on the same box.
```

- **No build, no artifact.** PHP is interpreted at request time. A visitor hits
  the site → Apache runs the PHP on the spot → it queries MySQL → HTML is
  rendered for that request. Nothing to compile or bundle.
- **The server is persistent and stateful** — a "pet", one long-lived machine
  kept alive, not rebuilt. It holds live code *and* the database.
- **"Changing code" = overwriting a `.php` file on that disk.** The next request
  picks it up instantly; no build to invalidate, nothing to restart.

### Who deployed it?

Nobody, in the modern sense. Someone once uploaded WordPress via FTP/cPanel and
walked away. Since then:

- WordPress **updates itself in place** (core/plugins/themes auto-update by
  writing to the server's own disk).
- Content editors change the **database** through wp-admin.
- Any code change was someone editing files directly on the live server.

No CI, no staging, no git, no artifact, no deployer. That's the WordPress default.

### What we're adding

| | Today (inherited) | What we add |
|---|---|---|
| Source of truth | The live server's disk | A git repo |
| Testing changes | Edit live, hope | Local WordPress |
| "Deploy" | Overwrite files in place | `git push` → sync theme to server |
| History / rollback | None | Git history |

When we wire up "git push → files land on the server," **we** become the
deployment system this site never had.

---

## What goes in git

The repo root **is** the theme directory (`wp-content/themes/<theme>/`):

```
my-site-theme/              ← the git repo IS the theme directory
├── .git/
├── style.css               ← theme header lives here
├── functions.php
├── *.php / templates/
├── assets/                 ← src + built CSS/JS
├── package.json            ← if a JS/CSS build step is added
└── .gitignore
```

Do **not** commit: WordPress core, third-party plugins, `wp-content/uploads`,
`node_modules/`, local config/secrets.

---

## Hosting: what we know

From wp-admin → Tools → Site Health → Info → Server:

```
Server architecture: Linux 4.18.0-...lve.1.el8.x86_64   ← CloudLinux LVE + RHEL8
Web server:          Apache
PHP:                 8.4.21 (cgi-fcgi)
```

- `lve` = **CloudLinux LVE** → almost exclusively used by **shared-hosting** providers.
- `el8` = RHEL 8 base.

**Conclusion: this is a cPanel shared host** (high confidence). That means:

1. There's a **cPanel login** separate from wp-admin (usually
   `yourdomain.com/cpanel` or `:2083`).
2. cPanel includes: **File Manager** (download the theme as a zip, no SSH),
   **SSH Access** (enable + add a key), and often **Git Version Control**
   (deploy by pulling on the server).
3. SSH is often disabled by default on cheap plans and gated behind a support
   ticket — so it's a nice-to-have, not a blocker.

### Access reality

- **wp-admin ≠ SSH.** wp-admin logs into the *application*; SSH/cPanel logs into
  the *server account*. They're different credentials.
- **Open blocker:** we only have wp-admin. Need to locate the **cPanel account
  login** (ask the owner, follow hosting invoices, whois the domain, check
  transactional email headers for the host's mail servers).

---

## Do we need SSH?

| Job | Needs server access? | Alternative |
|---|---|---|
| Develop the theme locally | No | — |
| Get the theme files down initially | Helpful | wp-admin file-manager / migration plugin, or cPanel File Manager |
| Deploy changes back up | Effectively yes | wp-admin Theme Editor (bad) or a deploy plugin |

SSH/SFTP is just the clean version of file access. You can bootstrap entirely
through wp-admin/cPanel, but you'll want real file access for a sane
`git push → deploy` loop.

---

## Getting the theme down (wp-admin only, for now)

1. **Migration/backup plugin** (cleanest — gets files **+ database** so the local
   site mirrors prod): *Duplicator*, *All-in-One WP Migration*, or *WP Migrate*.
2. **File-manager plugin** (theme files only): *Advanced File Manager* → browse
   `wp-content/themes/` → download the theme zip.

Once cPanel is found, its **File Manager** does the same thing more directly.

---

## Local environment options

- **LocalWP (Local by Flywheel)** — free GUI app, lowest friction; import a
  Duplicator/AIO archive and you're running in minutes. Good for the initial
  content bootstrap.
- **`@wordpress/env` (wp-env)** — Docker-based, config in `.wp-env.json` committed
  to the repo. Most "JS-native" (`npx wp-env start`), reproducible.
- **`docker-compose.yml`** (WordPress + MySQL images) — most control, fully
  committed to repo, closest to running things on Azure.

Recommended: bootstrap content once via LocalWP/Duplicator, then run dev on
**wp-env or docker-compose** committed to the repo.

---

## Deploying changes back (once file access exists)

- **SFTP deploy on push** — a GitHub Action (e.g. `SamKirkland/FTP-Deploy-Action`)
  rsyncs the theme dir to the server on merge to `main`. Simple, common.
- **cPanel Git Version Control** — host a repo on the server, deploy by pulling.
- **Git push-to-deploy** — native on managed hosts (WP Engine, Kinsta, Pantheon)
  if the site is ever migrated. Closest to the Vercel experience.
- **Plugin-based** (*WP Pusher*) — pulls the theme from GitHub into wp-admin;
  works even with wp-admin-only access.

---

## Worth considering: headless WordPress

Given a JS/Next/Vercel background, the pattern that plays to your strengths:
keep WordPress as a content API (**WPGraphQL** or the REST API) and build the
frontend in **Next.js deployed to Vercel**. Editors keep wp-admin; you stop
writing PHP themes. More upfront work — not always worth it for an inherited
site, but it's the direction that matches your tooling.

---

## Next steps

- [ ] Locate the cPanel / hosting account login (the real blocker).
- [ ] Pull the theme (and ideally a DB export) down via plugin or File Manager.
- [ ] `git init` the theme directory; add `.gitignore` + this README.
- [ ] Stand up a local WordPress (LocalWP to bootstrap, wp-env/docker for dev).
- [ ] Wire up a deploy path (SFTP Action / cPanel Git) once file access exists.
