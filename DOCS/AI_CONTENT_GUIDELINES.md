# AI_CONTENT_GUIDELINES.md

## Editorial and Visual Rules for Generating Content

This document defines the rules that AI systems should follow when
generating content for the **Ivan Luminaria Database Strategy blog**.

The goal is to maintain consistency in:

-   technical depth
-   editorial tone
-   visual style
-   conceptual clarity

------------------------------------------------------------------------

# 1. Editorial Tone

Content must be written in a **technical, analytical and professional
tone**.

Avoid:

-   marketing language
-   exaggerated claims
-   motivational writing
-   superficial tutorials

Prefer:

-   analytical explanations
-   engineering reasoning
-   real‑world scenarios
-   architectural thinking

Articles should feel like **engineering essays**, not blog marketing
content.

------------------------------------------------------------------------

# 2. Writing Style

Preferred characteristics:

-   clear technical explanations
-   structured reasoning
-   concise paragraphs
-   practical examples
-   deep understanding of database internals

Articles should emphasize:

-   cause and effect
-   performance implications
-   system behavior

Avoid generic phrases like:

"databases are important"

Instead explain **why** and **how**.

------------------------------------------------------------------------

# 3. Article Structure

When generating a technical article, use this structure:

1.  Problem introduction
2.  Technical explanation
3.  Real‑world example
4.  Implementation strategy
5.  Performance or architectural implications
6.  Practical takeaway

Articles should feel like **knowledge transfer from an experienced
engineer**.

------------------------------------------------------------------------

# 4. Target Audience

Readers are:

-   database engineers
-   backend engineers
-   system architects
-   CTOs
-   technical decision makers

Assume readers:

-   understand SQL
-   understand production environments
-   care about reliability and performance

Avoid beginner explanations unless necessary.

------------------------------------------------------------------------

# 5. Preferred Topics

Content may include:

-   query optimization
-   indexing strategies
-   execution plan analysis
-   database architecture
-   access control models
-   security practices
-   performance tuning
-   system reliability

Avoid beginner tutorials like:

"what is a database"

Focus instead on **how systems behave under real load**.

------------------------------------------------------------------------

# 6. Image Generation Rules

Images must follow the blog's retro technical illustration style.

Use prompts similar to:

retro 1950s cartoon illustration, minimal vector art, mid‑century
modern, art deco inspiration, geometric shapes, elegant silhouettes,
limited color palette (black, red, beige, brown, white), textured paper
background, dim lighting, vintage editorial illustration style, 16:9
aspect ratio

Images should represent **technical concepts through visual metaphors**.

Examples:

database control room\
engineers monitoring data systems\
security guards protecting data vaults\
operators analyzing dashboards

Avoid:

-   photorealistic imagery
-   modern stock photography
-   bright modern UI‑style graphics

------------------------------------------------------------------------

# 7. Conceptual Consistency

Each article should maintain conceptual coherence between:

-   title
-   technical explanation
-   illustration

Images should reinforce the core idea of the article.

Example:

Article about access control → illustration showing guards or security
checkpoints controlling entry to a data facility.

------------------------------------------------------------------------

# 8. Glossary Section

Every article must end with a `## Glossary` section listing **up to 10
key technical terms or acronyms** used in the article.

Format for each entry:

**Term** — Short, clear description (1–2 sentences).

Selection criteria — prioritize:

-   acronyms (e.g. AWR, SCD, ETL)
-   specific technical concepts (e.g. buffer pool, execution plan)
-   tools or technologies mentioned in the article

Avoid overly generic terms (e.g. "database", "SQL") unless they are
central to the article's topic.

The glossary must appear in **all 4 language versions** of the article,
with descriptions translated into each language.

After writing the glossary, **always update** the file
`DOCS/GLOSSARIO_TERMINI.md` — add new terms or update the "Contained in"
column for terms that already exist.

------------------------------------------------------------------------

# 9. Glossary Term Pages

Each glossary term is a standalone page under `content/glossary/{slug}/`
with 4 language versions (`index.it.md`, `index.en.md`, `index.es.md`,
`index.ro.md`).

## 9.1 Front Matter (mandatory fields)

```yaml
---
title: "TERM_NAME"
description: "Full expansion — one-sentence definition in the page language."
translationKey: "glossary_slug_name"
aka: "Full Name / Expansion of Acronym"
articles:
  - "/posts/section/article-slug"
---
```

- **title**: the term as commonly referenced (acronym or technical name)
- **description**: short definition suitable for the glossary index page
- **translationKey**: must be identical across all 4 language files
- **aka**: full expansion or alternative name; rendered as subtitle by the template
- **articles**: list of paths to related blog articles (resolved by Hugo `GetPage`);
  do **NOT** use hardcoded localized URLs — the template generates the correct
  language-specific link automatically
- Do **NOT** use `tags` — use `articles` instead

## 9.2 Content Structure

Every glossary term page must have:

1. **Opening paragraph**: bold term + expansion in parentheses, followed by
   a concise definition (2–3 sentences max)
2. **3–4 sections** with `##` headings, each focused on one aspect:
   - How it works / mechanism
   - Why it matters / when it's useful
   - Practical considerations / common pitfalls
   - Comparison or context (if relevant)
3. **No "Articoli correlati" section in the content** — related articles are
   handled automatically by the template via the `articles` front matter field

## 9.3 Section Heading Style

Use clear, short headings in the page language. Good examples:

| Italian              | English             | Spanish              | Romanian              |
|----------------------|---------------------|----------------------|-----------------------|
| Come funziona        | How it works        | Cómo funciona        | Cum functioneaza      |
| A cosa serve         | What it's for       | Para qué sirve       | La ce serveste        |
| Quando si usa        | When to use it      | Cuándo se usa        | Când se foloseste     |
| Perché è critico     | Why it matters      | Por qué es crítico   | De ce conteaza        |
| Cosa può andare storto | What can go wrong | Qué puede salir mal  | Ce poate merge prost  |

## 9.4 Writing Style for Terms

- Keep it technical but readable — same voice as the blog articles
- Use bullet lists only for enumerations (types, phases, included items)
- Paragraphs should be 2–4 sentences, not walls of text
- Include concrete numbers or examples where possible (e.g. "100 outer rows",
  "default value is 100")
- Avoid generic filler; every sentence should add information

## 9.5 Slug Naming — DB-specific vs Universal Terms

Glossary term directory names (slugs) follow these rules:

- **Universal or unambiguous terms** — use the term name directly, without any
  database prefix: `etl/`, `scd/`, `awr/`, `hash-join/`, `nested-loop/`,
  `execution-plan/`
- **DB-specific terms** — when a term exists on multiple databases with different
  behavior or meaning (e.g. ANALYZE, EXPLAIN, VACUUM), or when a term is a
  parameter/feature exclusive to one database (e.g. `default_statistics_target`),
  prefix the slug with the database name:
  `postgresql-analyze/`, `oracle-analyze/`, `mysql-analyze/`,
  `postgresql-default-statistics-target/`

This prevents slug collisions and makes it immediately clear which database
context the term refers to. The `translationKey` must follow the same convention
(e.g. `glossary_postgresql_analyze`).

**Rule of thumb**: if the term could reasonably have a separate glossary page
for another database, use the prefix. If it's a universal concept (algorithms,
design patterns, architectural terms), don't.

## 9.6 Consistency Rules

- All 4 language versions must have the **same structure** (same number of
  sections, same heading intent, same level of detail)
- The `translationKey` must match across all 4 files
- The `articles` list must be identical across all 4 files (paths are
  language-agnostic)

------------------------------------------------------------------------

# 10. Final Objective

The blog should evolve into a **technical knowledge base documenting
real engineering practice in database systems**.

Content should prioritize:

-   clarity
-   depth
-   accuracy
-   practical relevance

The ultimate goal is to build a **reference resource for database
strategy and engineering thinking**.
