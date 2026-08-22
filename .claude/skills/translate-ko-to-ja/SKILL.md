---
name: translate-ko-to-ja
description: Translate a Korean blog post (index.ko.md) to Japanese and save as index.ja.md.
argument-hint: "<post-slug>"
disable-model-invocation: true
allowed-tools: Read Glob Write Bash(grep *)
---

# Korean to Japanese Translation Skill

You are translating a Hugo blog post from Korean to Japanese.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Source: `content/posts/$ARGUMENTS/index.ko.md`
- Target: `content/posts/$ARGUMENTS/index.ja.md`

## Pre-flight Check

Japanese is already a configured language: `languages.ja` exists in `hugo.yaml` (weight 3) and `i18n/ja.yaml` is present. Confirm both with a single check and move on:

!`grep -A2 "^  ja:" hugo.yaml`

Only if that check comes back empty should you stop and report that the site config regressed — do not re-add the language block on your own.

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
- The Korean source is written in 평어체 (plain declarative). The Japanese target uses **です/ます体** — this is an intentional register change, not an inconsistency.
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

### Markup That Must Survive Translation
- Hugo shortcodes are copied verbatim, including the path inside them: `[text]({{< ref "/posts/slug" >}})` stays `{{< ref "/posts/slug" >}}` — `ref` resolves to the Japanese version automatically when one exists. Translate only the link text.
- Image references are relative to the page bundle (`![alt](images/name.svg)`) and must not be rewritten. Translate the alt text only.
- Diagram text inside SVG files stays English — never edit the SVGs.
- The Korean source glosses terms as `**한글용어**(English)`. In Japanese, use the katakana or Japanese term followed by the English gloss only where it aids comprehension: `**コンテナ**(container)`. Keep the parentheses outside the `**` markers.
- Em dash `—` stays `—`.

3. **Write** the translated content to `content/posts/$ARGUMENTS/index.ja.md`.

4. **Report** what was done:
```
## Translation Complete: {post title}

- Source: content/posts/{slug}/index.ko.md
- Target: content/posts/{slug}/index.ja.md
- Draft status: {true/false}
- Hugo JA config: verified
```

## Rules

- If `index.ja.md` already exists, read it first and warn the user before overwriting.
- The output should read naturally to a Japanese software engineer.
- Preserve paragraph breaks and formatting exactly.
- Do not add or remove content — translate what exists.
- Section headings translate to Japanese, including `마치며` → `おわりに` and `References` → `References` (kept as-is).
- `draft` must match the Korean version. If the Korean version is published and this translation is new, ask before publishing it in the same commit.
