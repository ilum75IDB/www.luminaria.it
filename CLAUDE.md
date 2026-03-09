# CLAUDE.md — ivanluminaria.com

## Project Overview

Personal and professional website of **Ivan Luminaria** — Oracle DBA, DWH Architect & Project Manager with ~30 years of experience across Oracle, PostgreSQL and MySQL environments. The site is a multilingual Hugo static site deployed to GitHub Pages.

Live URL: `https://ivanluminaria.com/`

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

### Writing articles

When writing a new blog article, **always** follow these steps:

1. **Read the content guidelines first** — before drafting any text, read the files in `DOCS/`, in particular:
   - `DOCS/AI_CONTENT_GUIDELINES.md` (English version)
   - `DOCS/AI_CONTENT_GUIDELINES_IT.md` (Italian version)
   - `DOCS/prompt-master.md` (master prompt with tone & style rules)
   - `DOCS/DESCRIZIONE_PROGETTO_DATABASE_STRATEGY_BLOG.md` and `DOCS/database_strategy_blog_project_description_FULL.md` (project context)
2. **Write like a human** — the text **must not** sound AI-generated. Avoid generic filler, motivational closings, bullet-point-heavy structures, and overly polished phrasing. Use Ivan's voice: direct, experienced, occasionally ironic, grounded in real-world project stories. Vary sentence length, use colloquial turns where appropriate, and let opinions show.
3. **No AI tells** — never use patterns like "In conclusion…", "It's worth noting that…", "Let's dive in…", "In today's fast-paced world…", or similar clichés typical of LLM output.

## Homepage Layout

The homepage (`layouts/_partials/home/custom.html`) uses an editorial magazine-style layout. Sections are displayed in a **fixed order** defined by the `$sections` slice in the template.

### Section display rules

- **Latest** section: 1 hero (full-width image above title) + 5 grid articles (total 6, controlled by `homepage.recentLimit` param)
- **Each category section**: 1 hero (2-column: image left, text right) + up to 4 grid articles (total max 5)
- **All images** must use **3:2 aspect ratio** — never change this

### Image sizes (3:2 ratio)

| Element             | srcset                          | CSS class                        |
|---------------------|---------------------------------|----------------------------------|
| Latest hero         | `800x533` / `1200x800`         | `.editorial-hero-img`            |
| Latest grid         | `160x107` / `320x213`          | `.editorial-grid-img`            |
| Category hero       | `300x200` / `450x300`          | `.editorial-category-hero-img`   |
| Category grid       | `160x107` / `320x213`          | `.editorial-grid-img`            |

### Category hero layout

Each category section hero uses a **2-column layout** (CSS class `.editorial-category-hero`):
- **Left column**: cover image (3:2, linked to article)
- **Right column** (`.editorial-category-hero-text`): 3 rows — title, date + reading time, description
- Below 480px: collapses to single column (image on top)

### Section order

| #  | Section                  | Slug                 | Articles displayed       |
|----|--------------------------|----------------------|--------------------------|
| 1  | Ultimi articoli (Latest) | *(all sections)*     | 1 hero + 5 grid = 6     |
| 2  | Data Warehouse Architect | `data-warehouse`     | 1 hero + up to 4 grid   |
| 3  | Project Management       | `project-management` | 1 hero + up to 4 grid   |
| 4  | Oracle                   | `oracle`             | 1 hero + up to 4 grid   |
| 5  | PostgreSQL               | `postgresql`         | 1 hero + up to 4 grid   |
| 6  | MySQL                    | `mysql`              | 1 hero + up to 4 grid   |

**Important**: When adding new sections, update both this table and the `$sections` slice in `layouts/_partials/home/custom.html`. Never modify the image aspect ratios or the category hero 2-column structure.

## Custom Layouts & Overrides

- **`layouts/partials/translations.html`**: Custom language switcher with flag dropdown (replaces Congo default)
- **`layouts/shortcodes/staticurl.html`**: Resolves static asset URLs with correct base path
- **`layouts/_default/single.html`**: Article template with cover image support
- **`layouts/simple.html`**: Used by the About page for a clean profile layout
- **Navbar**: CSS-forced to `position: fixed` with `z-index: 9999` (see custom.css)

## Publication Schedule

Articles are published **one per week, every Tuesday at 10:00 CET**, starting from 16 December 2025.

### Rules

1. **Cadence**: one article per Tuesday, no gaps
2. **Time**: always `T10:00:00+01:00`
3. **First article**: 2025-11-18 (Tuesday)
4. **New articles**: take the next available Tuesday slot after the last published article
5. **All 4 language versions** of each article share the same date
6. **Schedule table format**: always show the columns below. When reorganising dates, fill the "New Date" column; use `—` if no change is needed

### Current Schedule

| #  | Date       | New Date | Day | Article                                                    | Section            | Issue | Status    |
|----|------------|----------|-----|------------------------------------------------------------|--------------------|-------|-----------|
| 1  | 2025-11-18 | —        | Tue | Da single instance a Data Guard                            | oracle             | #20   | published |
| 2  | 2025-11-25 | —        | Tue | 4 milioni e nessun software                                | project-management | #28   | published |
| 3  | 2025-12-02 | —        | Tue | LIKE optimization PostgreSQL                               | postgresql         | #29   | published |
| 4  | 2025-12-09 | —        | Tue | Oracle Partitioning (2 miliardi di righe)                  | oracle             | #19   | published |
| 5  | 2025-12-16 | —        | Tue | MySQL users and hosts                                      | mysql              | #30   | published |
| 6  | 2025-12-23 | —        | Tue | AI e GitHub per gestire un progetto                        | project-management | #31   | published |
| 7  | 2025-12-30 | —        | Tue | Ragged Hierarchies (self-parenting)                        | data-warehouse     | #24   | published |
| 8  | 2026-01-06 | —        | Tue | Oracle ruoli e privilegi (GRANT ALL)                       | oracle             | #22   | published |
| 9  | 2026-01-13 | —        | Tue | La tecnica del Sì-E (Yes-And)                              | project-management | #25   | planned   |
| 10 | 2026-01-20 | —        | Tue | PostgreSQL roles and users                                 | postgresql         | #32   | published |
| 11 | 2026-01-27 | —        | Tue | Standup meeting (max 15 minuti)                            | project-management | #27   | planned   |
| 12 | 2026-02-03 | —        | Tue | Galera Cluster a 3 nodi                                    | mysql              | #33   | published |
| 13 | 2026-02-10 | —        | Tue | AWR, ASH e i 10 minuti che hanno salvato un go-live        | oracle             | #21   | planned   |
| 14 | 2026-02-17 | —        | Tue | Smart working nella consulenza IT                          | project-management | #34   | published |
| 15 | 2026-02-24 | —        | Tue | Oracle su Linux - i parametri kernel che nessuno configura  | oracle             | #23   | planned   |
| 16 | 2026-03-03 | —        | Tue | Bici vs Auto a Roma                                        | project-management | #35   | published |
| 17 | 2026-03-10 | —        | Tue | Pagamenti a 60-90-120 giorni                               | project-management | #26   | published |

**Next available slot**: 2026-03-17 (Tuesday)

When adding a new article, update this table, assign the next available Tuesday slot, and set the "Next available slot" line accordingly.

## Important Notes

- The Congo theme is a **Git submodule** — always clone/checkout with `--recurse-submodules` or run `git submodule update --init`
- The `baseURL` in `hugo.toml` is set for the custom domain (`https://ivanluminaria.com/`)
- PDF resumes are stored in `static/downloads/` and linked from the resumes content pages
- The `.gitignore` excludes `public/`, `resources/_gen/`, and `.hugo_build.lock`

## GitHub Issues

In questo ambiente **non è disponibile `gh` CLI** e non è possibile accedere direttamente alle API di GitHub. Quando l'utente chiede di "creare una issue", in realtà sta chiedendo di **generare il comando `gh issue create` pronto per il copia-incolla**, in modo che possa eseguirlo lui stesso dal proprio terminale. Non tentare di installare `gh`, non provare workaround con `curl` o API proxy: fornisci direttamente i comandi `gh issue create` completi e pronti all'uso.

### Consultare le issue aperte

Poiché `gh` CLI non è disponibile, il file **`DOCS/GITHUB_ISSUES.md`** contiene i link diretti alle issue aperte del repository. Quando l'utente chiede di consultare, ricordare o lavorare sulle issue:

1. **Leggere `DOCS/GITHUB_ISSUES.md`** per ottenere i link delle issue
2. **Usare `WebFetch`** sui link per leggere il contenuto completo di ciascuna issue da GitHub
3. Quando una issue viene chiusa o ne vengono create di nuove, **aggiornare `DOCS/GITHUB_ISSUES.md`** di conseguenza
4. Quando l'utente chiede di **creare una nuova issue** e poi la crea dal suo terminale, **chiedere sempre il link** della issue appena creata per poterlo inserire in `DOCS/GITHUB_ISSUES.md`. Non procedere senza aver aggiornato il file

### Regole di formattazione per i comandi gh issue create

**CRITICO — il comando deve funzionare con copia-incolla diretto nel terminale macOS (zsh):**

1. **NON usare backtick `` ` `` o triple backtick `` ``` `` nel body** — la shell li interpreta come command substitution e il comando fallisce. Per i blocchi di codice nel body, usare indentazione con 4 spazi oppure scrivere la parola `Codice:` seguita dal contenuto su righe successive.
2. **NON usare `$(cat <<'EOF' ... EOF)`** per il body — è fragile con contenuti complessi. Usare il flag `--body-file` con un file temporaneo.
3. **Approccio obbligatorio — blocco unico copia-incolla**: genera un **unico blocco di codice** che contiene sia il `cat` per creare il file temporaneo sia il comando `gh issue create`, separati da `&&`. In questo modo l'utente fa un solo copia-incolla e la issue viene creata immediatamente. Formato:
   ```
   cat > /tmp/issue-body.md << 'ISSUE_EOF'
   ... contenuto body ...
   ISSUE_EOF
   gh issue create --repo ilum75IDB/ivanluminaria.com --title "..." --label "..." --body-file /tmp/issue-body.md
   ```
4. Nel body della issue, i blocchi di codice vanno scritti con triple backtick normalmente — saranno interpretati correttamente da GitHub perché il file viene letto da `--body-file`, non parsato dalla shell.
5. **NON usare caratteri accentati nei titoli** del comando `gh issue create` (nei `--title`). Sostituire à→a, è→e, é→e, ù→u, ò→o, ì→i, ecc. I titoli delle issue devono essere ASCII-safe.
6. **Testare mentalmente**: prima di fornire il comando, verificare che non ci siano apici, virgolette o backtick che possano confondere la shell zsh.

### Label GitHub create

Le seguenti label sono già state create nel repository `ilum75IDB/ivanluminaria.com`:

| Label                | Colore    | Descrizione                          |
|----------------------|-----------|--------------------------------------|
| `oracle`             | `#F80000` | Articoli sezione Oracle              |
| `blog-article`       | `#336791` | Nuovo articolo del blog              |
| `postgresql`         | `#336791` | Articoli sezione PostgreSQL          |
| `mysql`              | `#00758F` | Articoli sezione MySQL               |
| `project-management` | `#4a4a4a` | Articoli sezione Project Management  |
| `data-warehouse`     | `#1E3A5F` | Articoli sezione Data Warehouse Architect |
| `ai-manager`         | `#8B5CF6` | Articoli sezione AI Manager          |

Quando si crea una nuova issue per un articolo, usare sempre la label `blog-article` insieme alla label della sezione corrispondente (es. `--label "oracle,blog-article"`).
