# CLAUDE.it.md — www.luminaria.it

## Panoramica del Progetto

Sito web personale e professionale di **Ivan Luminaria** — Oracle DBA, DWH Architect & Project Manager con circa 30 anni di esperienza su ambienti Oracle, PostgreSQL e MySQL. Il sito e un sito statico Hugo multilingue pubblicato su GitHub Pages.

URL live: `https://ilum75IDB.github.io/www.luminaria.it/`
(Target produzione: `https://www.luminaria.it/`)

## Stack Tecnologico

- **Generatore di siti statici**: [Hugo](https://gohugo.io/) (versione extended, ultima disponibile)
- **Tema**: [Congo](https://github.com/jpanther/congo) — installato come sottomodulo Git in `themes/congo/`
- **Deploy**: GitHub Actions → GitHub Pages (workflow in `.github/workflows/hugo.yml`)
- **CSS**: Override personalizzati in `assets/css/custom.css` (font Inter, colori brand, navbar fissa, syntax highlighting)
- **Lingue**: Hugo multilingue — Italiano (default, peso 1), Inglese (peso 2), Spagnolo (peso 3), Rumeno (peso 4)

## Struttura delle Directory

```
.
├── archetypes/          # Archetypes Hugo (default.md)
├── assets/
│   ├── css/custom.css   # Tutto il CSS personalizzato (colori brand, fix layout)
│   └── img/             # Immagine avatar
├── config/
│   └── _default/
│       ├── languages.{it,en,es,ro}.toml   # Configurazione per lingua
│       ├── menus.{it,en,es,ro}.toml       # Menu di navigazione
│       ├── markup.toml                     # Goldmark / syntax highlighting
│       ├── module.toml                     # Configurazione moduli Hugo
│       └── params.toml                     # Parametri tema e info autore
├── content/
│   ├── about.{it,en,es,ro}.md             # Pagina "Chi Sono" (tutte le lingue)
│   ├── posts/
│   │   ├── _index.{it,en,es,ro}.md        # Landing sezione articoli
│   │   └── database-strategy.cover.jpg    # Immagine di copertina sezione
│   └── resumes/
│       ├── index.{it,en,es,ro}.md         # Pagina CV / Esperienze
│       └── resumes.cover.jpg              # Immagine di copertina sezione
├── data/
│   └── taxonomy_images.toml               # Mappatura immagini tassonomie
├── layouts/
│   ├── baseof.html                        # Template base (override)
│   ├── simple.html                        # Layout semplice (usato da Chi Sono)
│   ├── list.html                          # Override layout lista
│   ├── _default/
│   │   ├── single.html                    # Template articolo singolo
│   │   └── term.html                      # Template termine tassonomia
│   ├── _partials/
│   │   ├── article-link.html
│   │   └── article-meta.html
│   ├── partials/
│   │   └── translations.html              # Dropdown selettore lingua
│   └── shortcodes/
│       └── staticurl.html                 # Shortcode {{</* staticurl "percorso" */>}}
├── static/
│   ├── img/                               # Immagini statiche (avatar, bandiere)
│   └── downloads/                         # CV in formato PDF
├── themes/congo/                          # Tema Congo (sottomodulo Git)
├── hugo.toml                              # Configurazione principale Hugo
└── .github/workflows/hugo.yml             # CI/CD: build e deploy
```

## Build e Sviluppo

```bash
# Server di sviluppo locale (con bozze)
hugo server -D

# Build di produzione
hugo --minify

# L'output va in public/ (nel gitignore)
```

E richiesta la versione **extended** di Hugo (per l'elaborazione SCSS/PostCSS del tema Congo).

## Deploy

Il push sul branch `main` attiva il workflow GitHub Actions (`.github/workflows/hugo.yml`) che:

1. Clona il repository con i sottomoduli
2. Configura Hugo (ultima versione extended)
3. Esegue `hugo --minify`
4. Carica `public/` come artefatto Pages
5. Pubblica su GitHub Pages

## Architettura Multilingue

- **Lingua predefinita**: Italiano (`it`), con `defaultContentLanguageInSubdir = true` (i contenuti risiedono sotto `/it/`)
- I file di contenuto usano la convenzione del suffisso: `about.it.md`, `about.en.md`, ecc.
- Gli indici di sezione usano: `_index.it.md`, `_index.en.md`, ecc.
- Le pagine bundle usano: `index.it.md`, `index.en.md`, ecc.
- Ogni lingua ha la propria configurazione menu in `config/_default/menus.{lingua}.toml`
- Il selettore lingua e un partial personalizzato in `layouts/partials/translations.html` che usa immagini bandiera da `static/img/flags/`

### Aggiungere una Nuova Lingua

1. Creare `config/_default/languages.{codice}.toml` con il peso appropriato
2. Creare `config/_default/menus.{codice}.toml` con le voci di menu tradotte
3. Aggiungere i file di contenuto tradotti con il suffisso `.{codice}.md`
4. Aggiungere l'immagine della bandiera in `static/img/flags/{codice}.png`

## Brand e Design System

Definiti in `assets/css/custom.css`:

| Token                | Valore      | Utilizzo                                              |
|----------------------|-------------|-------------------------------------------------------|
| `--ivan-blue`        | `#336791`   | Blu Postgres — nome brand, testo strong, breadcrumbs  |
| `--ivan-red`         | `#F80000`   | Rosso Oracle — link, menu nav, bordi blockquote       |
| `--ivan-red-dark`    | `#b30000`   | Stato hover dei link                                  |
| `--ivan-gray-title`  | `#4a4a4a`   | Titoli (h1–h4)                                        |
| `--ivan-gray-obj`    | `#6b7280`   | Identificatori SQL nei blocchi di codice              |
| `--ivan-sql-green`   | `#2e8b57`   | Keyword SQL nei blocchi di codice                     |

Font: **Inter** (caricato da Google Fonts), dimensione base 20px.

## Linee Guida per i Contenuti

- Il sito ha una voce autoriale distintiva — riflessiva, diretta, professionale ma personale
- I contenuti sono scritti in Markdown con shortcode Hugo
- Usare lo shortcode `{{</* staticurl "percorso" */>}}` quando si referenziano asset statici (necessario per la corretta risoluzione dei percorsi su GitHub Pages)
- Goldmark e configurato con `unsafe = true` per permettere HTML grezzo nel Markdown
- Le immagini di copertina delle sezioni sono posizionate accanto ai file di contenuto (es. `database-strategy.cover.jpg`)

## Layout e Override Personalizzati

- **`layouts/partials/translations.html`**: Selettore lingua personalizzato con dropdown a bandiere (sostituisce il default di Congo)
- **`layouts/shortcodes/staticurl.html`**: Risolve gli URL degli asset statici con il percorso base corretto
- **`layouts/_default/single.html`**: Template articolo con supporto immagine di copertina
- **`layouts/simple.html`**: Usato dalla pagina Chi Sono per un layout profilo pulito
- **Navbar**: Forzata via CSS a `position: fixed` con `z-index: 9999` (vedi custom.css)

## Note Importanti

- Il tema Congo e un **sottomodulo Git** — clonare/fare checkout sempre con `--recurse-submodules` oppure eseguire `git submodule update --init`
- Il `baseURL` in `hugo.toml` e impostato per il sito progetto GitHub Pages (`/www.luminaria.it/`); questo influenza tutti i percorsi degli asset
- I CV in PDF sono archiviati in `static/downloads/` e collegati dalle pagine di contenuto resumes
- Il `.gitignore` esclude `public/`, `resources/_gen/` e `.hugo_build.lock`
