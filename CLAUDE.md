# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## Project Overview

This is a personal tech blog built with **Hugo** (v0.112.0), deployed to GitHub Pages at https://payne4handsome.github.io. The site uses the **PaperMod** theme (`themes/PaperMod/`). Config is in `config.yml`, default language is Chinese.

## Content Structure

All blog posts live under `content/`:

```
content/
├── posts/
│   ├── basic/            # DevOps, Linux, DB notes (~10 posts)
│   ├── java/             # Java concurrency, generics (~6 posts)
│   ├── machine learning/ # ML/DL fundamentals, transformers, VLMs (~19 posts)
│   └── compute network/  # HTTP/2, TLS, protocols (~3 posts)
├── papers/               # Paper analysis series (~22 posts)
├── tech-news/            # Daily tech progress (new section)
├── search.md
└── archive.md
```

Directory names use mixed Chinese and English — be aware of this when creating new content paths.

## Key Commands

```bash
# Build the site (output goes to public/)
hugo -d public/

# Local dev server with drafts
hugo server -D

# Create a new post
hugo new posts/category/title.md
hugo new tech-news/2026-05-05-title.md

# Create a new paper analysis post (uses archetypes/papers.md)
hugo new papers/title.md
```

## Architecture Notes

- **Theme**: PaperMod is vendored in `themes/PaperMod/`. Custom layouts go in `layouts/` (currently empty).
- **Archetypes**: `archetypes/default.md` and `archetypes/papers.md` — paper posts use date-prefixed naming convention.
- **Output formats**: HTML + JSON (for Fuse.js search). RSS is disabled.
- **Static assets**: `static/` contains cover images (cover001~003).
- **Taxonomies**: categories and tags. Papers section also uses series taxonomy.
- **Menu items** defined in `config.yml` under `languages.zh.menu.main`. Add new tabs there.
- **`public/` is gitignored** — do not commit build output.
