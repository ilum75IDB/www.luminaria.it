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
5. **Backdated articles**: if the user asks to publish an article in the past, assign it to the **Previous available slot** date. Then update the Previous available slot to the Tuesday before the new oldest article
6. **All 4 language versions** of each article share the same date
7. **Schedule table format**: always show the columns below. When reorganising dates, fill the "New Date" column; use `—` if no change is needed
8. **Slot markers** (at the bottom of `DOCS/HUGO_PUBLICATIONS_TABLE.md`):
   - **Previous available slot**: the first Tuesday before the oldest article's date. Used when inserting backdated articles
   - **Next available slot**: the first Tuesday after the most recent article's date. Used for new articles
   - Both must be updated after every schedule change

### Current Schedule

The full publication table is maintained in **`DOCS/HUGO_PUBLICATIONS_TABLE.md`**. Always read and update that file when working with the schedule.

When adding a new article, update the table in `DOCS/HUGO_PUBLICATIONS_TABLE.md`, assign the next available Tuesday slot, and update both slot markers accordingly.

### Publication dashboard rule

When the user asks for the "tabella delle pubblicazioni" (publication table), **always show 3 tables**:

1. **Tabella pubblicazioni** — the Current Schedule table above (all published + planned articles with dates)
2. **Tabella issue aperte** — all open issues grouped by section (from `DOCS/GITHUB_ISSUES.md`)
3. **Riepilogo per sezione** — a summary table with columns: Sezione, Pubblicati, Pianificati (issue aperte), Totale

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

### Chiusura issue di articoli pubblicati

Quando si chiude una issue relativa a un articolo pubblicato, il commento di chiusura deve contenere:

1. **Data di pubblicazione** — la data prevista dal calendario editoriale (es. 2025-11-11)
2. **URL dell'articolo** — link alla versione italiana sul sito live (es. `https://ivanluminaria.com/it/posts/data-warehouse/scd-tipo-2/`)
3. **Conferma tutte le lingue** — specificare che l'articolo è stato pubblicato in tutte e 4 le lingue (IT, EN, ES, RO)
4. **Breve riassunto** — 2-3 frasi che descrivono il contenuto dell'articolo

Formato del comando:

```
gh issue close <numero> --repo ilum75IDB/ivanluminaria.com --comment "Articolo pubblicato il <data>.

URL: https://ivanluminaria.com/it/posts/<sezione>/<slug>/

Pubblicato in tutte le lingue: IT, EN, ES, RO.

<breve riassunto del contenuto>"
```

Dopo la chiusura, **aggiornare `DOCS/GITHUB_ISSUES.md`**: spostare la issue dalla sezione aperte alla sezione chiuse con la data di chiusura.

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

## Sincronizzazione repo con l'utente

L'utente lavora in locale sul suo Mac, Claude lavora su una branch separata. Ogni volta che si fanno modifiche o si devono sincronizzare i repo, seguire queste regole.

### Dopo ogni commit/push di Claude

Dopo aver fatto commit e push sulla branch di sviluppo, fornire **sempre** all'utente i comandi per:

1. Posizionarsi sulla branch corretta (se non ci è già)
2. Aggiornare il repo locale con le modifiche

Formato:

```bash
git fetch origin <branch-name>
git checkout <branch-name>
git pull origin <branch-name>
```

### Quando l'utente dice di aver fatto modifiche in locale

Fornire i comandi per aggiornare il repo remoto in modo che Claude possa poi aggiornare il suo repo locale:

```bash
git add <files>
git commit -m "descrizione modifiche"
git push -u origin <branch-name>
```

Poi, dopo che l'utente conferma il push, Claude esegue `git pull origin <branch-name>` per allinearsi.

### Quando l'utente dice "ora possiamo portare tutto online"

Significa: merge nella branch main, push al remoto, e ritorno sulla branch di sviluppo. Fornire i comandi nell'ordine:

```bash
git checkout main
git pull origin main
git merge <branch-name>
git push -u origin main
git checkout <branch-name>
```

Dove `<branch-name>` è la branch di sviluppo corrente.
