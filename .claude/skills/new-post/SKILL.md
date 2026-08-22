---
name: new-post
description: Scaffold a new blog post with directory, front matter, and topic outline.
argument-hint: "<slug> [title]"
disable-model-invocation: true
allowed-tools: Read Glob Grep Write
---

# New Post Scaffolding Skill

You are creating the scaffolding for a new Hugo blog post.

## Input

- `$ARGUMENTS` = `<slug>` and optionally `[title]`
  - Example: `my-new-post "My New Post Title"`
  - If title is not provided, derive a readable Korean title from the slug

## Steps

1. **Parse arguments**: Extract slug and optional title from `$ARGUMENTS`.

2. **Check for conflicts**: Verify `content/posts/{slug}/` does not already exist using Glob. If it exists, warn the user and stop.

3. **Scan existing tags**: Read front matter from several existing posts in `content/posts/*/index.ko.md` to collect the set of tags already used in the blog. Tags are lowercase, hyphenated, English (e.g. `sre`, `cost-optimization`, `ci-cd`). Reuse existing tags where they fit.

4. **Create directory and files.** The Korean version is the primary one and gets the outline; the other two languages are placeholders that the translate skills fill in later.

### `content/posts/{slug}/index.ko.md`
```markdown
---
title: "{한국어 제목}"
date: {today's date in YYYY-MM-DD format}
draft: true
tags: [{suggest 3-5 relevant tags from existing tags + new ones if needed}]
translationKey: "{slug}"
summary: ""
---

<!-- 도입부: 제목 바로 아래 본문으로 시작한다. '## Introduction' 같은 제목을 달지 않는다.
     이 주제를 다루게 된 계기나 마주한 문제를 2~4문단으로 쓴다.
     시리즈 글이라면 [이전 글]({{< ref "/posts/이전-슬러그" >}})을 먼저 언급한다. -->

## 1. {첫 번째 장 제목}

<!-- 각 장은 이전 장과 연결되는 도입 1~2문장으로 시작한다. -->

### 1.1 {소절 제목}

## 2. {두 번째 장 제목}

### 2.1 {소절 제목}

## 3. {세 번째 장 제목}

## 마치며

<!-- 요약 → 개인적 통찰 → 앞으로의 방향 순서로 쓴다. -->

### References

- [{제목}]({url})
```

### `content/posts/{slug}/index.md` (English placeholder)
```markdown
---
title: "{title in English}"
date: {today's date}
draft: true
tags: [{same tags}]
translationKey: "{slug}"
summary: ""
---

(English translation placeholder — run /translate-ko-to-en {slug})
```

### `content/posts/{slug}/index.ja.md` (Japanese placeholder)
```markdown
---
title: "{title in Japanese}"
date: {today's date}
draft: true
tags: [{same tags}]
translationKey: "{slug}"
summary: ""
---

(Japanese translation placeholder — run /translate-ko-to-ja {slug})
```

Also create the `content/posts/{slug}/images/` directory placeholder expectation: post images are referenced as `![alt](images/name.svg)` and live in that directory. Do not create empty directories — just note it in the report.

5. **Generate a topic outline**: Based on the slug/title, suggest concrete numbered chapter titles in Korean, specific to the topic. Look at similar existing posts for structural inspiration (`slo-sli-sla-error-budget`, `redis-deep-dive` are good models: `## N. 제목` chapters with `### N.M` subsections, closing with `## 마치며` and `### References`).

6. **Report**:
```
## New Post Created: {title}

- Directory: content/posts/{slug}/
- Files created:
  - index.ko.md (Korean, draft, outlined)
  - index.md (English placeholder, draft)
  - index.ja.md (Japanese placeholder, draft)
- Suggested tags: [...]
- Suggested outline provided above

Next steps:
1. Fill in the summary field (평어체)
2. Write the content in index.ko.md
3. Run /tech-review {slug} and /style-checker when ready
4. Run /translate-ko-to-en {slug} and /translate-ko-to-ja {slug}
```

## Rules

- Always set `draft: true` for new posts, in all three languages.
- Use today's actual date.
- Tags should reuse existing blog tags where applicable, and stay lowercase-hyphenated English.
- The outline should be specific to the topic, not generic.
- Do NOT generate actual content — only scaffolding and suggestions.

## Writing conventions to bake into the scaffold

These are the blog's standing conventions (see `CLAUDE.md`). Reflect them in every comment and placeholder you write:

- **평어체** (~이다 / ~한다) for body text and for the `summary` field — never 경어체 (~합니다).
- Bold with parenthetical English goes **outside** the bold: `**한글용어**(English)`, not `**한글용어(English)**` — parentheses inside `**` break rendering in Korean markdown.
- Em dash is `—`, not `--`.
- Each chapter opens with 1-2 transition sentences connecting it to the previous chapter.
- The closing section is `## 마치며`, following summary → personal insight → future direction.
- Internal post links use the shortcode form: `[text]({{< ref "/posts/other-slug" >}})`.
