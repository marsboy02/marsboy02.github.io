---
title: "Welcome to My Blog"
date: 2026-03-01
draft: false
tags: ["blog", "design"]
translationKey: "welcome-to-my-blog"
summary: "A brief introduction to this blog — its design philosophy, tech stack, and typography choices."
---

## About This Blog

Welcome! This is a minimal personal blog built with simplicity and readability in mind. No heavy frameworks, no JavaScript bundles — just clean HTML, CSS, and a fast static site generator.

## Tech Stack

This blog is powered by:

- **[Hugo](https://gohugo.io/)** — One of the fastest static site generators, written in Go. Pages are generated at build time, so there's no server-side rendering or database to worry about.
- **GitHub Pages** — Free hosting for static sites, deployed automatically on every push.
- **CSS Custom Properties** — Used for the light/dark theme toggle. No CSS preprocessor needed.
- **No JavaScript frameworks** — Just a few lines of vanilla JS for the theme toggle and nothing else.

## Typography

The blog uses a **system font stack** for maximum performance and native feel:

```
-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif
```

This means:

- **macOS / iOS**: San Francisco
- **Windows**: Segoe UI
- **Android**: Roboto
- **Linux**: System default sans-serif

No web fonts are loaded, which keeps the page weight minimal and eliminates font-loading flicker.

## Design Philosophy

1. **Content first** — The layout is a single centered column (720px max-width) with generous line height (1.8) for comfortable reading.
2. **Minimal chrome** — No sidebar, no ads, no tracking scripts. Just the content.
3. **Dark mode** — Respects your OS preference by default, with a manual toggle in the header.
4. **Multilingual** — Full support for English and Korean, with per-page translation linking.
5. **Responsive** — Works on any screen size, from phones to ultrawide monitors.

## Colors

| Element | Light | Dark |
|---------|-------|------|
| Background | `#fff` | `#1a1a1a` |
| Text | `#222` | `#e0e0e0` |
| Links | `#0066cc` | `#6db3f2` |
| Borders | `#e0e0e0` | `#333` |

That's it. A simple blog for writing and sharing.
