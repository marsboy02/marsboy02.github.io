---
name: writing-conventions
description: The blog's established writing conventions — tone, terminology formatting, structure, front matter, and linking rules. Load before writing, reviewing, or translating any post.
when_to_use: Writing or editing a post under content/posts/, reviewing tone or formatting consistency, translating between ko/en/ja, or scaffolding a new post.
user-invocable: false
allowed-tools: Read Glob Grep
---

# Blog Writing Conventions

These are settled conventions, not open questions. When a post deviates, the post is wrong — do not infer a new convention from a single deviating post.

## Tone

- Body text uses **평어체**: `~이다`, `~한다`. Never 경어체 (`~합니다`, `~해요`).
- The `summary` front matter field also uses 평어체.
- First-person reflection is welcome in the intro and in `## 마치며`; the technical body stays declarative.

## Terminology formatting

- Bold with a parenthetical English gloss puts the parentheses **outside** the bold:
  - Correct: `**가용성**(availability)`
  - Wrong: `**가용성(availability)**` — parentheses inside `**` break bold rendering in Korean markdown.
- Once a term is introduced with its English gloss, use the Korean form consistently for the rest of the post.
- Product and technology names keep their canonical spelling (Kubernetes, Redis, Istio), not transliterations, unless the post has already established the Korean form.
- Em dash is `—`. Never `--`.

## Structure

- The post opens with prose directly under the front matter. There is no `## Introduction` heading.
- Chapters are numbered H2 headings in Korean: `## 1. 제목`, with `### 1.1 소절` subsections.
- Each chapter opens with 1-2 transition sentences connecting it to the previous chapter.
- Series posts reference the previous entry in the intro: `[이전 글]({{< ref "/posts/previous-slug" >}})`, then state what this post covers.
- The closing section is `## 마치며`, and follows: summary → personal insight → future direction.
- When a post cites sources, they go last under `### References` as a link list. This is an emerging pattern rather than a settled rule — only the two most recent long-form posts (`computer-architecture-story`, `slo-sli-sla-error-budget`) currently have one, so treat a missing References section as a suggestion, not a violation.

## Front matter

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

- `translationKey` is identical across `index.ko.md`, `index.md`, and `index.ja.md`, and matches the directory slug.
- Tags are lowercase, hyphenated, English: `sre`, `ci-cd`, `cost-optimization`.
- `draft: true` while writing; flip to `false` only when publishing.

## Files and assets

- A post is a page bundle: `content/posts/<slug>/` containing `index.ko.md` (primary, written first), `index.md` (English), `index.ja.md` (Japanese), and `images/`.
- Post images are referenced relative to the bundle: `![alt](images/name.svg)`. Site-wide assets live in `static/images/` and are referenced as `/images/name.png`.
- Diagram text is English-only, even in Korean posts.

## Linking

- Internal post links use Hugo shortcodes: `[text]({{< ref "/posts/other-slug" >}})` or `{{< relref "/posts/other-slug" >}}`. Raw `](/posts/slug/)` paths are not used in this blog.
- External links are plain markdown links.

## Reference posts

When you need a concrete model for structure or voice:

- `content/posts/slo-sli-sla-error-budget/index.ko.md` — numbered chapters, argument-driven structure, strong `마치며`
- `content/posts/computer-architecture-story/index.ko.md` — narrative and analogy-heavy style the blog aspires to
- `content/posts/redis-deep-dive/index.ko.md` — hands-on technical walkthrough with code blocks
