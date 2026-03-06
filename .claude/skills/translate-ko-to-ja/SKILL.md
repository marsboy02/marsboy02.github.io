---
name: translate-ko-to-ja
description: Translate a Korean blog post (index.ko.md) to Japanese and save as index.ja.md, with Hugo config pre-flight check.
user_invocable: true
arguments: "<post-slug>"
---

# Korean to Japanese Translation Skill

You are translating a Hugo blog post from Korean to Japanese.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Source: `content/posts/$ARGUMENTS/index.ko.md`
- Target: `content/posts/$ARGUMENTS/index.ja.md`

## Pre-flight Check

Before translating, verify Hugo is configured for Japanese:

1. **Read `hugo.yaml`** and check if `languages.ja` exists.
2. **Check if `i18n/ja.yaml`** exists using Glob.

If either is missing, output the following and ask the user before proceeding:

```
## Pre-flight: Japanese language not configured

The following setup is needed before Japanese translation:

### 1. Add to hugo.yaml under `languages`:
languages:
  ja:
    languageName: JA
    weight: 3
    menus:
      main:
        - name: About
          pageRef: /about
          weight: 10
        - name: Posts
          pageRef: /posts
          weight: 20

### 2. Create i18n/ja.yaml:
name: "your name in Japanese"
role: "Software Engineer"
tagline: "description in Japanese"
posted_on: "投稿日"
copyright: "All rights reserved."

Shall I apply these changes and proceed with translation?
```

## Translation Steps

1. **Read** `content/posts/$ARGUMENTS/index.ko.md`.

2. **Translate** following these rules:

### Front Matter
- `title`: Translate to Japanese
- `summary`: Translate to Japanese
- `date`: Keep as-is
- `draft`: Match the Korean version's value
- `tags`: Keep as-is (tags are already in English)
- `translationKey`: Keep as-is

### Body Content
- Use polite form (です/ます体)
- Follow Japanese IT industry conventions for technical terms:
  - Container = コンテナ, Deploy = デプロイ, Server = サーバー, etc.
  - Use katakana for established loanwords
  - Use the term most common in Japanese tech blogs/documentation
- Adapt examples to be natural for Japanese readers
- Keep the same heading structure

### Code Blocks
- Keep code blocks exactly as-is
- Only translate Korean comments to Japanese
- Do NOT modify any code logic

### Links & Images
- Keep all URLs and image references unchanged
- Translate link text and image alt text

3. **Write** the translated content to `content/posts/$ARGUMENTS/index.ja.md`.

4. **Report** what was done:
```
## Translation Complete: {post title}

- Source: content/posts/{slug}/index.ko.md
- Target: content/posts/{slug}/index.ja.md
- Draft status: {true/false}
- Hugo JA config: {already configured / newly added}
```

## Rules

- If `index.ja.md` already exists, read it first and warn the user before overwriting.
- The output should read naturally to a Japanese software engineer.
- Preserve paragraph breaks and formatting exactly.
- Do not add or remove content — translate what exists.
