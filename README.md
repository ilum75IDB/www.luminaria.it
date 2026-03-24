# ivanluminaria.com

Personal and professional website of **Ivan Luminaria** — Oracle DBA, Data Warehouse Architect & Project Manager with ~30 years of experience in database environments.

🌐 **[ivanluminaria.com](https://ivanluminaria.com/)**

## What's inside

A multilingual technical blog about databases, data warehousing, and project management. Articles are written in **Italian, English, Spanish, and Romanian** — all human-crafted, drawn from real-world project experience.

### Sections

- **Oracle** — tuning, AWR analysis, migration stories, daily DBA life
- **PostgreSQL** — adoption strategies, performance, ecosystem tools
- **MySQL** — practical tips and operational insights
- **Data Warehouse** — architecture, ETL patterns, SCD strategies, modeling
- **Project Management** — lessons learned from decades of IT projects

## Tech stack

- [Hugo](https://gohugo.io/) (extended) — static site generator
- [Congo](https://github.com/jpanther/congo) theme (Git submodule)
- GitHub Actions → GitHub Pages (automatic deploy on push + weekly scheduled build)
- Custom CSS with brand colors (Oracle red, Postgres blue)

## About AI usage

This project uses **Claude Code** (Anthropic) as a development and writing tool. To be clear about how:

- **Every article idea, technical angle, and editorial decision is mine.** Claude doesn't decide what to write — I do, based on 30 years of hands-on experience with databases and projects.
- **I use AI as an accelerator, not as a ghostwriter.** Claude helps me draft, translate into 4 languages, manage the Hugo codebase, and handle the repetitive parts of a multilingual site. The voice, the opinions, the war stories — those are mine.
- **The code is AI-assisted too.** Custom layouts, CSS, GitHub Actions workflows, and site structure are built in collaboration with Claude Code. I review, test, and approve everything.
- **Why be transparent?** Because I think hiding AI usage is dishonest, and because using AI well is a skill worth showing — not something to be embarrassed about. The puppeteer is human. The puppet is artificial. The show is real.

## Local development

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/ilum75IDB/ivanluminaria.com.git

# Run dev server
hugo server -D

# Production build
hugo --minify
```

Requires Hugo **extended** version.

## License

Content and code © Ivan Luminaria. All rights reserved.
