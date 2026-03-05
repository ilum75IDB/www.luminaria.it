# CLAUDE.md — www.luminaria.it

## Project Overview

Personal and professional website of **Ivan Luminaria** — Oracle DBA, DWH Architect & Project Manager with ~30 years of experience across Oracle, PostgreSQL and MySQL environments. The site is a multilingual Hugo static site deployed to GitHub Pages.

Live URL: `https://ilum75IDB.github.io/www.luminaria.it/`
(Production target: `https://www.luminaria.it/`)

## Tech Stack

- **Static Site Generator**: [Hugo](https://gohugo.io/) (extended version, latest)
- **Theme**: [Congo](https://github.com/jpanther/congo) — installed as a Git submodule under `themes/congo/`
- **Deployment**: GitHub Actions → GitHub Pages (workflow at `.github/workflows/hugo.yml`)
- **CSS**: Custom overrides in `assets/css/custom.css` (Inter font, brand colors, fixed navbar, syntax highlighting)
- **Languages**: Hugo multilingual — Italian (default, weight 1), English (weight 2), Spanish (weight 3), Romanian (weight 4)

## Directory Structure

```
.
├── archetypes/          # Hugo archetypes (default.md)
├── assets/
│   ├── css/custom.css   # All custom CSS (brand colors, layout fixes)
│   └── img/             # Avatar image
├── config/
│   └── _default/
│       ├── languages.{it,en,es,ro}.toml   # Per-language config
│       ├── menus.{it,en,es,ro}.toml       # Navigation menus
│       ├── markup.toml                     # Goldmark / syntax highlighting
│       ├── module.toml                     # Hugo module config
│       └── params.toml                     # Theme parameters & author info
├── content/
│   ├── about.{it,en,es,ro}.md             # About page (all languages)
│   ├── posts/
│   │   ├── _index.{it,en,es,ro}.md        # Posts section landing
│   │   └── database-strategy.cover.jpg    # Section cover image
│   └── resumes/
│       ├── index.{it,en,es,ro}.md         # Resumes / CV page
│       └── resumes.cover.jpg              # Section cover image
├── data/
│   └── taxonomy_images.toml               # Taxonomy cover images mapping
├── layouts/
│   ├── baseof.html                        # Base template override
│   ├── simple.html                        # Simple layout (used by About)
│   ├── list.html                          # List layout override
│   ├── _default/
│   │   ├── single.html                    # Single article template
│   │   └── term.html                      # Taxonomy term template
│   ├── _partials/
│   │   ├── article-link.html
│   │   └── article-meta.html
│   ├── partials/
│   │   └── translations.html              # Language switcher dropdown
│   └── shortcodes/
│       └── staticurl.html                 # {{</* staticurl "path" */>}} shortcode
├── static/
│   ├── img/                               # Static images (avatar, flags)
│   └── downloads/                         # PDF resumes
├── themes/congo/                          # Congo theme (Git submodule)
├── hugo.toml                              # Main Hugo config
└── .github/workflows/hugo.yml             # CI/CD: build & deploy
```

## Build & Development

```bash
# Local development server (with drafts)
hugo server -D

# Production build
hugo --minify

# Output goes to public/ (gitignored)
```

Hugo **extended** version is required (for SCSS/PostCSS processing in the Congo theme).

## Deployment

Pushing to the `main` branch triggers the GitHub Actions workflow (`.github/workflows/hugo.yml`) which:

1. Checks out the repo with submodules
2. Sets up Hugo (latest extended)
3. Runs `hugo --minify`
4. Uploads `public/` as a Pages artifact
5. Deploys to GitHub Pages

## Multilingual Architecture

- **Default language**: Italian (`it`), with `defaultContentLanguageInSubdir = true` (content lives under `/it/`)
- Content files use the suffix convention: `about.it.md`, `about.en.md`, etc.
- Section indexes use: `_index.it.md`, `_index.en.md`, etc.
- Bundle pages use: `index.it.md`, `index.en.md`, etc.
- Each language has its own menu config in `config/_default/menus.{lang}.toml`
- Language switcher is a custom partial at `layouts/partials/translations.html` using flag images from `static/img/flags/`

### Adding a New Language

1. Create `config/_default/languages.{code}.toml` with appropriate weight
2. Create `config/_default/menus.{code}.toml` with translated menu items
3. Add translated content files with the `.{code}.md` suffix
4. Add the flag image to `static/img/flags/{code}.png`

## Brand & Design System

Defined in `assets/css/custom.css`:

| Token               | Value       | Usage                            |
|----------------------|-------------|----------------------------------|
| `--ivan-blue`        | `#336791`   | Postgres blue — brand name, strong text, breadcrumbs |
| `--ivan-red`         | `#F80000`   | Oracle red — links, nav menu, blockquote borders     |
| `--ivan-red-dark`    | `#b30000`   | Hover states for links                               |
| `--ivan-gray-title`  | `#4a4a4a`   | Headings (h1–h4)                                     |
| `--ivan-gray-obj`    | `#6b7280`   | SQL identifiers in code blocks                       |
| `--ivan-sql-green`   | `#2e8b57`   | SQL keywords in code blocks                          |

Font: **Inter** (loaded from Google Fonts), base size 20px.

## Content Guidelines

- The site has a distinctive authorial voice — thoughtful, direct, professional but personal
- Content is written in Markdown with Hugo shortcodes
- Use `{{</* staticurl "path" */>}}` shortcode when referencing static assets (needed for correct GitHub Pages subpath resolution)
- Goldmark is configured with `unsafe = true` to allow raw HTML in Markdown
- Cover images for sections are placed alongside content files (e.g., `database-strategy.cover.jpg`)

## Custom Layouts & Overrides

- **`layouts/partials/translations.html`**: Custom language switcher with flag dropdown (replaces Congo default)
- **`layouts/shortcodes/staticurl.html`**: Resolves static asset URLs with correct base path
- **`layouts/_default/single.html`**: Article template with cover image support
- **`layouts/simple.html`**: Used by the About page for a clean profile layout
- **Navbar**: CSS-forced to `position: fixed` with `z-index: 9999` (see custom.css)

## Important Notes

- The Congo theme is a **Git submodule** — always clone/checkout with `--recurse-submodules` or run `git submodule update --init`
- The `baseURL` in `hugo.toml` is set for GitHub Pages project site (`/www.luminaria.it/`); this affects all asset paths
- PDF resumes are stored in `static/downloads/` and linked from the resumes content pages
- The `.gitignore` excludes `public/`, `resources/_gen/`, and `.hugo_build.lock`
