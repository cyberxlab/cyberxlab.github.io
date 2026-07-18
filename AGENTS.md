# AGENTS.md — Cyber·X·Lab Website

> Guidance for AI coding agents (Hermes, Claude Code, Codex, etc.) working on this repository.

## Project Overview

Cyber·X·Lab (`Cyber·X·Lab`) is a multilingual (English + 简体中文) documentation and community website built with **Hugo Extended** and the **Hextra** theme. The site is deployed to GitHub Pages via GitHub Actions and served through Cloudflare at **https://cyberxlab.3344198.xyz/**.

- **Repo:** `github.com/cyberxlab/cyberxlab.github.io`
- **Branch:** `main` (push triggers automatic deployment)
- **Domain:** `cyberxlab.3344198.xyz` (configured via Cloudflare CNAME → GitHub Pages)
- **Theme:** `github.com/imfing/hextra` (v0.12.3+) via Hugo Modules
- **Languages:** English (en, default) + 简体中文 (zh-cn)

## Tech Stack & Key Dependencies

| Dependency        | Version    | Purpose                          |
|-------------------|------------|----------------------------------|
| Hugo Extended     | ≥ 0.146.0  | Static site generator            |
| Go                | ≥ 1.21     | Hugo Module system dependency    |
| Node.js           | 20         | PostCSS/Tailwind CSS compilation |
| Hextra theme      | v0.12.3+   | Hugo module via `go.mod`         |
| Tailwind CSS v4   | ^4.3.0     | Styling framework (via Hextra)   |
| PostCSS           | ^8.5.18    | CSS post-processing               |
| Playwright        | ^1.60.0    | E2E + accessibility tests         |

## Project Structure

```
.
├── .github/workflows/
│   └── pages.yaml              # CI/CD: build & deploy to GitHub Pages
├── assets/                     # Hugo asset resources (processed at build)
│   ├── css/                     # Custom CSS overrides
│   └── images/                 # Images referenced as Hugo resources
├── content/                    # All site content (Markdown)
│   ├── _index.md              # Homepage (English)
│   ├── _index.zh-cn.md         # Homepage (Chinese)
│   ├── about/                  # About page
│   ├── articles/               # Articles/showcase section
│   ├── blog/                   # Blog posts
│   ├── docs/                   # Documentation
│   │   ├── getting-started/
│   │   ├── guide/
│   │   └── advanced/
│   ├── glossary/                # Glossary terms
│   └── wechat/                 # WeChat community page
├── data/                        # Hugo data files (termbase, etc.)
├── i18n/                        # Custom i18n overrides (en, zh-cn)
│   ├── en.yaml
│   └── zh-cn.yaml
├── layouts/                    # Layout overrides (Hugo template lookup)
│   ├── _partials/              # Overridden partials
│   └── _shortcodes/            # Custom shortcodes
├── static/                     # Static assets served as-is
│   ├── images/                 # Logos, screenshots, banners
│   └── CNAME                   # Custom domain (cyberxlab.3344198.xyz)
├── hugo.yaml                   # Main Hugo configuration
├── go.mod                       # Go module definition (Hextra dependency)
├── go.sum
├── package.json                 # Node.js dev dependencies (PostCSS, Tailwind, Playwright)
└── hugo_stats.json              # Hugo build stats (auto-generated, committed for Tailwind)
```

## Build & Development

### Prerequisites

1. **Hugo Extended** ≥ 0.146.0 (must be the extended version for SCSS support)
   ```bash
   hugo version  # Verify: "extended" must appear in output
   ```
2. **Go** ≥ 1.21 (required by Hugo Modules)
   ```bash
   go version
   ```
3. **Node.js** 20 (required for PostCSS/Tailwind CSS compilation)
   ```bash
   node --version
   npm ci  # Install dependencies (use ci, not install, for reproducible builds)
   ```

### Local Development

```bash
# 1. Ensure Hugo module dependencies are resolved
hugo mod tidy

# 2. Start local dev server
hugo server

# 3. For a production-like local build:
hugo --gc --minify
```

- Local dev server runs at `http://localhost:1313/`
- Hugo server auto-reloads on file changes.
- **Do NOT** use `--themesDir=../..` or `--source=docs`; those are for the Hextra theme's own repo, not our site.

### Build Verification

```bash
# Production build (same as CI)
hugo --gc --minify

# Verify build exit code
echo "Exit: $?"
# Expected output: ~416 pages (en + zh-cn), 0 errors
```

### Running Tests

```bash
# All Playwright tests
npm test

# Accessibility tests only
npm run test:a11y

# Mobile menu tests only
npm run test:mobile-menu
```

## Deployment

Deployment is fully automated via GitHub Actions:

1. **Trigger:** Any push to `main` branch
2. **Pipeline:** `.github/workflows/pages.yaml`
3. **Steps:**
   - Install Hugo Extended (v0.160.1)
   - Checkout code (with submodules)
   - Setup Node.js 20 + `npm ci`
   - Configure GitHub Pages
   - Build with Hugo (`--gc --minify`)
   - Deploy to GitHub Pages artifact
4. **Live URL:** `https://cyberxlab.3344198.xyz/` (via Cloudflare DNS → GitHub Pages)

**Manual deployment:** Go to Actions tab → "Deploy Hugo site to Pages" → "Run workflow"

## Configuration Guide (`hugo.yaml`)

### Key Configuration Decisions

| Setting                     | Value                              | Reason                          |
|-----------------------------|------------------------------------|---------------------------------|
| `baseURL`                   | `https://cyberxlab.3344198.xyz/`   | Production domain              |
| `title`                     | `Cyber·X·Lab`                      | Brand name (with middle dots)  |
| `defaultContentLanguage`    | `en`                               | English is primary language    |
| `params.navbar.displayLogo` | `true`                             | Show logo in navbar            |
| `params.navbar.displayTitle`| `true`                             | Show brand text next to logo   |
| `params.navbar.logo.path`   | `images/logo.svg`                  | Light-mode logo (in `static/`) |
| `params.navbar.logo.dark`   | `images/logo-dark.svg`             | Dark-mode logo                  |
| `params.navbar.logo.width`  | `36`                               | Navbar logo dimensions         |
| `params.navbar.logo.height` | `36`                               |                                 |
| `params.editURL.enable`     | `false`                            | No "edit this page" links      |
| `params.theme.default`      | `system`                           | Auto-detect dark/light mode    |
| `params.footer.displayPoweredBy` | `true`                        | "Powered by Hextra" in footer  |
| `comments.enable`           | `false`                            | Giscus disabled (no active repo)|

### Navigation Menu Structure

The navbar has the following structure (defined in `hugo.yaml` → `menu.main`):

```
Showcase (→ /articles)
Blog (→ /blog)
Community ▾                          # Dropdown menu
  ├── X (Twitter)  → external
  ├── YouTube      → external
  ├── Discord      → external
  ├── Telegram    → external
  └── WeChat      → /wechat
More ▾                               # Dropdown menu
  ├── Documentation → /docs
  └── Glossary      → /glossary
[Search] [GitHub icon]
```

**When modifying menus:** Always update both the `menu.main` section in `hugo.yaml` AND the corresponding i18n translation keys in `i18n/en.yaml` and `i18n/zh-cn.yaml`.

### i18n Custom Overrides

This project has custom i18n files in `i18n/en.yaml` and `i18n/zh-cn.yaml`. These files only contain **overrides** for menu labels and copyright text that differ from Hextra's built-in translations.

- **Do NOT** replace these files with Hextra's full i18n files — they contain only the subset of keys we need to override.
- **Do NOT** delete these files — their absence will cause menu items to fall back to Hextra's default labels (which won't match our menu structure).
- Keys currently overridden: `documentation`, `showcase`, `blog`, `glossary`, `about`, `more`, `hugoDocs`, `versions`, `development`, `community`, `copyright`.

## Layout Overrides

Custom layout templates live in `layouts/`:

- `layouts/_partials/navbar-title.html` — Navbar logo + title rendering
- `layouts/_partials/footer.html` — Footer customization
- `layouts/_partials/shortcodes/card.html` — Card shortcode override
- `layouts/_shortcodes/new-feature.html` — Badge shortcode
- And others in `layouts/_partialisons/` and `layouts/_shortcodes/`

**When modifying layouts:** Hugo's template lookup prioritizes files in `layouts/` over the theme module's `layouts/`. Always check if a partial is overridden before editing — overriding the wrong copy will have no effect.

## Content Conventions

### Markdown Files

- English content: `content/<section>/<file>.md`
- Chinese content: `content/<section>/<file>.zh-cn.md`
- Use `{{% relref "path/to/file" %}}` for internal links (language-aware)
- Use `{{< shortcode >}}` syntax for shortcodes

### Page Front Matter

```yaml
---
title: "Page Title"
date: 2026-07-13
draft: false
weight: 1  # For ordering in sidebar/nav
---
```

## Important Lessons Learned

### 1. Hugo Modules: Do NOT Use `hugo.work` or `replace` directives

The upstream Hextra `docs/` directory uses `hugo.work` workspace files and `replace github.com/imfing/hextra => ../` in `go.mod` — those only work inside the monorepo clone. Our standalone site must:
- NOT have a `hugo.work` file (delete it if syncing from upstream)
- NOT have a `replace` directive in `go.mod` (only `require github.com/imfing/hextra vX.Y.Z`)
- Use `hugo mod tidy` to resolve the module from the Go proxy

### 2. i18n Files: Add User Overrides, Don't Copy Full Theme Files

Hextra ships its own complete i18n files (40+ keys) inside the theme module. Our `i18n/*.yaml` files should only contain the **subset** of keys we need to overwrite (menu labels, copyright text). Hugo merges project-level i18n with theme-level i18n, so a partial file is correct. Copying the full theme file and then letting it become stale will cause missing translations.

### 3. Logo Files Must Exist in `static/images/`

The `hugo.yaml` references `images/logo.svg` and `images/logo-dark.svg` via `params.navbar.logo.path` / `.dark`. These paths resolve to `static/images/logo.svg` and `static/images/logo-dark.svg`. If either file is missing, Hugo build does NOT fail, but the rendered HTML references broken image URLs, causing navbar rendering issues (including the language switcher text disappearing).

### 4. GitHub Actions: Node + npm ci Are Required

The CI workflow includes `actions/setup-node@v4` and `npm ci` steps before the Hugo build. These are required because Hextra v0.12+ uses Tailwind CSS v4 which requires PostCSS processing at build time. Without Node.js and `npm ci`, the build fails with `POSTCSS: failed to transform` errors.

### 5. Middle Dot Character: Use U+00B7 (·)

The brand name uses the standard middle dot character `·` (U+00B7), NOT the fullwidth katakana middle dot `・` (U+30FB). Mixing these causes inconsistent rendering across fonts and platforms.

## Common Tasks

### Blog Post / Article Maintenance

Blog and article content is the most frequently updated part of this site. Every post must exist as a **bilingual pair** — if you create or edit one language, create or edit the other in the same commit.

#### Add a New Blog Post

1. Create `content/blog/<slug>.md` (English) and `content/blog/<slug>.zh-cn.md` (Chinese)
2. Front matter: `title`, `date`, `draft: false`, optional `weight`
3. Use `{{% relref "blog/<slug>" %}}` for internal cross-links (language-aware)
4. Submit to `main` — CI auto-deploys

#### Update an Existing Blog Post

1. Edit both `content/blog/<slug>.md` and `content/blog/<slug>.zh-cn.md`
2. Bump `date` in front matter only if the change is substantive (new sections, rewritten content); skip for typo fixes
3. Do NOT change the filename/slug — it would break existing URLs and `relref` links

#### Delete a Blog Post

1. Remove both `content/blog/<slug>.md` and `content/blog/<slug>.zh-cn.md`
2. Search the entire `content/` tree for any `{{% relref "blog/<slug>" %}}` references pointing to it and remove/replace them
3. Commit and push; the old URL will return 404 after deployment

### Add a New Documentation Page

1. Create `content/docs/<section>/<page>.md` (+ `.zh-cn.md`)
2. The page auto-appears in the docs sidebar based on `content/docs/` directory structure

### Update the Logo

1. Replace both `static/images/logo.svg` (light mode) and `static/images/logo-dark.svg` (dark mode)
2. Verify with `hugo server` that the navbar renders correctly
3. Ensure the SVG has no white/colored background rect (must be transparent)
4. Run `hugo --gc --minify` to verify build success

### Add a New Language

1. Add a new entry under `languages:` in `hugo.yaml` (copy the `en` block)
2. Create `i18n/<lang>.yaml` with custom overrides
3. Create `content/_index.<lang>.md` and section-specific `.<lang>.md` files
4. Set `weight` to control language order in the switcher

### Sync from Upstream Hextra

1. Clone `github.com/imfing/hextra` temporarily
2. Copy `docs/` content selectively — only the files you need
3. **Delete** any `hugo.work` file and remove `replace` from `go.mod`
4. **Delete** any `i18n/*.yaml` files that are full copies of theme i18n (keep only our overrides)
5. Run `hugo mod tidy` then `hugo --gc --minify` to verify
6. Commit and push

## Verification Checklist

Before pushing to `main` (which auto-deploys), verify:

- [ ] `hugo --gc --minify` succeeds with exit code 0
- [ ] No broken logo references in `public/` (check `public/index.html` for correct `<img src=...>` paths)
- [ ] Language switcher shows current language label (not empty) in generated HTML
- [ ] All i18n menu labels resolve correctly (no missing/wrong text in navbar)
- [ ] `static/CNAME` exists with content `cyberxlab.3344198.xyz`
- [ ] No `hugo.work` file in repo root
- [ ] No `replace` directive in `go.mod`
- [ ] `npm test` passes (if Playwright tests exist)

## Do NOT

- **Do NOT** commit `public/` or `resources/` directories (they are in `.gitignore`)
- **Do NOT** force-push or rewrite history on `main`
- **Do NOT** modify files under `themes/` — the theme is managed as a Hugo Module
- **Do NOT** mix `hugo.work` workspace files from the upstream Hextra monorepo
- **Do NOT** copy Hextra's full i18n files wholesale — use only our override keys
- **Do NOT** add `replace` directives in `go.mod` — use the published module version
- **Do NOT** change `static/CNAME` without coordinating DNS changes
