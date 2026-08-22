# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hugo static blog (marsboy02.github.io) — a multilingual tech blog (KO/EN/JA) using custom theme `gandanham`, deployed to GitHub Pages.

## Build & Dev Commands

```bash
hugo server                    # Local dev server with live reload
hugo server -D                 # Include draft posts
hugo --minify                  # Production build (output: public/)
hugo --minify --baseURL "URL"  # Production build with custom base URL
```

Hugo extended v0.157.0+ required. No package.json or Makefile — pure Hugo project.

## Content Structure

Posts live in `content/posts/<slug>/` as page bundles:
- `index.ko.md` — Korean (primary, written first)
- `index.md` — English translation
- `index.ja.md` — Japanese translation
- `images/` — Post-specific images and SVG diagrams

### Frontmatter Template

```yaml
---
title: "제목"
date: YYYY-MM-DD
draft: true
tags: ["tag1", "tag2"]
translationKey: "post-slug"
summary: "한 줄 요약"
---
```

- `translationKey` links translations across languages — must match across all language versions
- Use `draft: true` during writing, set to `false` only when publishing
- Internal post links: `[text]({{< ref "/posts/post-slug" >}})` or `{{< relref "/posts/post-slug" >}}`

## Writing Style Conventions

- **평어체** (~이다/~한다) for body text — not 경어체 (~합니다)
- summary in frontmatter also uses 평어체
- Bold formatting with parenthetical English: `**한글용어**(English)` not `**한글용어(English)**` — parentheses inside `**` break rendering in Korean markdown
- Em dash: use `—` not `--`
- Each chapter starts with 1-2 transition sentences connecting to previous chapter
- "마치며" section follows pattern: summary → personal insight → future direction
- Intro pattern for series posts: reference previous post with `[이전 글]({{< ref "..." >}})`, then state what this post covers

## Architecture

- **Config**: `hugo.yaml` — languages, menus, params, Giscus comments
- **Theme**: `themes/gandanham/` — base theme, overridden by root `layouts/`
- **Layout overrides**: `layouts/` — `baseof.html`, `_default/single.html`, partials for head/header/footer/comments
- **i18n**: `i18n/` — UI string translations (en.yaml, ko.yaml, ja.yaml)
- **Markdown rendering**: `goldmark` with `unsafe: true` (allows raw HTML), CSS-class-based syntax highlighting

## Deployment

GitHub Actions on push to `main` → Hugo build → GitHub Pages. Workflow in `.github/workflows/`.
