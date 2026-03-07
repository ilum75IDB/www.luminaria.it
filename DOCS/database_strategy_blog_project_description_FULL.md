# Ivan Luminaria -- Database Strategy Blog

## Project Overview for AI Collaboration

## 1. Project Purpose

This project is a technical blog and professional portfolio focused on
database engineering, data architecture, and performance strategy.

The site serves three main goals:

1.  Share practical knowledge about database internals, performance
    tuning, and architecture.
2.  Demonstrate professional expertise in PostgreSQL, Oracle, MySQL, and
    Data Warehouse systems.
3.  Provide high-value technical content aimed at engineers, architects,
    and technical decision makers.

The blog is intentionally written in a clear, analytical and
method-driven tone, avoiding marketing hype or superficial explanations.

The guiding principle is:

> Databases are not just software components.\
> They are strategic infrastructure that determines performance,
> reliability and scalability of modern digital systems.

------------------------------------------------------------------------

# 2. Technical Architecture

The website is built using a static site architecture designed for
simplicity, reliability, and performance.

## Core Stack

-   Hugo -- Static site generator
-   GitHub -- Version control
-   GitHub Pages -- Hosting and deployment
-   GitHub Actions -- Automated build and deployment pipeline

This architecture guarantees:

-   zero runtime backend
-   minimal attack surface
-   extremely fast page load times
-   deterministic builds

## Repository Structure (simplified)

    project-root
    │
    ├── archetypes/
    ├── assets/
    │   └── css/
    │       └── custom.css
    │
    ├── content/
    │   ├── about.*.md
    │   ├── posts/
    │   │   ├── postgresql/
    │   │   ├── mysql/
    │   │   └── query-tuning/
    │   │
    │   └── resumes/
    │
    ├── layouts/
    │   ├── partials/
    │   ├── shortcodes/
    │   └── base templates
    │
    ├── static/
    │   ├── img/
    │   └── downloads/
    │
    ├── config/
    │   └── _default/
    │
    └── .github/workflows/

## Deployment Flow

1.  Developer pushes changes to GitHub.
2.  GitHub Actions triggers a build workflow.
3.  Hugo generates the static site into `/public`.
4.  The site is deployed automatically to GitHub Pages.

This guarantees fully automated CI/CD for the blog.

------------------------------------------------------------------------

# 3. Content Structure

The blog is organized into thematic sections focused on real database
engineering problems.

## Main Sections

### Database Strategy

High-level architectural thinking about data systems.

Topics include:

-   database architecture decisions
-   performance vs scalability trade-offs
-   data modeling strategies
-   system reliability

### Query Tuning

Deep dives into query performance.

Examples:

-   LIKE optimization
-   execution plans
-   index strategies
-   optimizer behavior

### Security & Access Control

Topics related to database security:

-   roles and privileges
-   access management
-   production security practices

### Professional Roadmap

The Resumes section presents structured skill roadmaps describing:

-   technical capabilities
-   architectural competencies
-   professional roles

This section serves as a professional capability map, not just a CV.

------------------------------------------------------------------------

# 4. Multilingual Strategy

The site is intentionally multilingual.

Languages currently supported:

-   Italian
-   English
-   Spanish
-   Romanian

Hugo handles language routing automatically using language
subdirectories:

    /it/
    /en/
    /es/
    /ro/

The language switcher is implemented via a custom template component.

------------------------------------------------------------------------

# 5. Visual Style

The site adopts a clean technical aesthetic.

Design principles:

-   readable typography
-   minimal visual noise
-   strong contrast for technical readability
-   structured hierarchy

Custom styling is implemented in:

    assets/css/custom.css

The color palette references database ecosystems:

-   PostgreSQL blue
-   Oracle red

The visual identity reinforces the theme of serious engineering and
technical depth.

------------------------------------------------------------------------

# 6. Tone and Writing Style

The blog intentionally avoids:

-   marketing language
-   exaggerated claims
-   superficial tutorials

Instead it focuses on:

-   analytical explanations
-   real-world scenarios
-   architectural reasoning

The tone is:

-   technical
-   precise
-   pragmatic
-   experience-driven

Articles are written as engineering case studies rather than SEO-driven
blog posts.

------------------------------------------------------------------------

# 7. Target Audience

The primary readers are:

-   database engineers
-   backend engineers
-   data architects
-   CTOs and technical leaders

The content assumes the reader:

-   understands SQL
-   works with production systems
-   cares about performance and scalability

This blog is not designed for beginners.

------------------------------------------------------------------------

# 8. Long Term Vision

The long-term objective is to build a reference resource for database
strategy and engineering practice.

Future directions include:

-   deeper architectural essays
-   database performance case studies
-   system design analysis
-   cross-database comparisons

The blog is intended to evolve into a technical knowledge base
documenting real-world engineering experience.

------------------------------------------------------------------------

# 9. Core Philosophy

The philosophy behind the project can be summarized simply:

> A database that merely works is not enough.\
> A database must scale, remain reliable under pressure, and support the
> business that depends on it.

This project exists to explore what it really takes to achieve that.

------------------------------------------------------------------------

# 10. Visual Style for AI Generated Illustrations

All visual illustrations used in the blog are generated using AI image
models.

The goal is not photorealism, but conceptual illustration that visually
represents technical ideas.

The visual language follows a consistent style inspired by 1950s
mid‑century editorial illustration.

## Artistic Direction

Illustrations must follow these stylistic principles:

-   retro cartoon aesthetic inspired by 1950s animation
-   minimal vector style
-   strong geometric shapes
-   smooth flowing lines
-   mid‑century modern composition
-   Art‑Deco influences
-   elegant silhouettes

The goal is to evoke the feeling of technical editorial illustrations
from vintage magazines.

## Color Palette

The palette is intentionally limited to maintain visual coherence.

Allowed colors:

-   black
-   red
-   beige
-   brown
-   white

Occasionally muted gray tones may be used for depth.

No bright modern digital palettes should be used.

## Atmosphere

Images should evoke the feeling of:

-   vintage editorial graphics
-   jazz club atmosphere
-   dim lighting
-   textured paper backgrounds

The final result should resemble an illustration printed in a 1950s
technical magazine.

## Composition Principles

Images communicate technical ideas through visual metaphors.

Examples:

Control rooms → database monitoring\
Operators analyzing dashboards → query analysis\
Engineers around a database console → architecture thinking\
Security doors or guards → database security\
Data flow diagrams → system architecture

Characters may appear slightly exaggerated in the style of UPA animation
studios.

## Image Format

All illustrations should follow these constraints:

-   aspect ratio 16:9
-   minimalistic vector‑like rendering
-   limited visual clutter
-   clear silhouettes

## Image Purpose

Images are not decorative.

They exist to:

-   visually introduce technical topics
-   reinforce conceptual understanding
-   establish a recognizable editorial identity

Each illustration should represent the conceptual theme of the article
or category.

------------------------------------------------------------------------

# 11. Bug Reporting & Issue Workflow

When the user reports a bug or a visual/functional problem on the site,
the AI assistant **must** follow this workflow automatically, without
being asked:

## Step 1 — Create the GitHub Issue

As soon as the user describes a bug, the AI **must immediately** provide
a ready-to-paste `gh issue create` command with:

-   a clear, concise title
-   a structured body containing: **Description**, **Steps to
    Reproduce**, **Expected Behavior**, **Actual Behavior**, and
    appropriate labels (e.g. `bug`)

Example format:

```bash
gh issue create \
  --title "Short description of the bug" \
  --body "$(cat <<'EOF'
## Description
...

## Steps to Reproduce
1. ...

## Expected Behavior
...

## Actual Behavior
...

## Environment
- Theme: Congo
- Mode: dark / light / both
EOF
)" \
  --label "bug"
```

## Step 2 — Fix the Bug on the Working Branch

The AI develops the fix on the designated `claude/` working branch,
commits with a clear message referencing the issue number (e.g.
`Fix #N — description`), and pushes.

## Step 3 — Provide Local Sync Commands

After pushing, the AI **must** provide the user with the exact commands
to sync the working branch locally for testing:

```bash
# Fetch the updated branch
git fetch origin <branch-name>

# Switch to the branch (first time)
git checkout <branch-name>

# Or pull if already on the branch
git pull origin <branch-name>

# Run local dev server for testing
hugo server -D
```

## Step 4 — Provide the Issue Close Comment

Once the user confirms the fix works locally, the AI **must** generate
a Markdown comment to be posted on the GitHub issue to document the
resolution. The comment must include:

-   **Problem**: what was broken and why
-   **Root Cause**: technical explanation
-   **Solution**: what was changed (files, approach)
-   **Files Modified**: list of changed files

The AI provides this as a ready-to-paste Markdown block.

## Step 5 — Close the Issue

After the comment is posted, the AI provides the command to close the
issue:

```bash
gh issue close <N> --comment "Fixed in branch <branch-name>"
```

## Summary of Automatic Behaviors

| User action              | AI must automatically provide            |
|--------------------------|------------------------------------------|
| Reports a bug            | `gh issue create` command                |
| Bug is fixed & pushed    | `git fetch` / `checkout` / `pull` commands for local sync |
| User confirms fix works  | Markdown close comment + `gh issue close` command |

This workflow is **mandatory** and must be followed for every bug
report without the user needing to request it explicitly.
