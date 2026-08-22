---
name: translate-ko-to-en
description: Translate a Korean blog post (index.ko.md) to natural English and save as index.md.
argument-hint: "<post-slug>"
disable-model-invocation: true
allowed-tools: Read Glob Write
---

# Korean to English Translation Skill

You are translating a Hugo blog post from Korean to English.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Source: `content/posts/$ARGUMENTS/index.ko.md`
- Target: `content/posts/$ARGUMENTS/index.md`

## Steps

1. **Read** `content/posts/$ARGUMENTS/index.ko.md`.

2. **Translate** the content following these rules:

### Front Matter
- `title`: Translate to English
- `summary`: Translate to English
- `date`: Keep as-is
- `draft`: Match the Korean version's value
- `tags`: Keep as-is (tags are already in English)
- `translationKey`: Keep as-is

### Body Content
- Translate to **natural, idiomatic English** — NOT literal translation
- The Korean source is written in 평어체 (plain declarative). Render it as plain, confident English prose — not stiff, not chatty
- Adapt cultural references and idioms for English-speaking readers
- Technical terms should use standard English terminology
- Keep the same heading structure (H2, H3, etc.)
- Maintain the same tone (if casual in Korean, casual in English)

### Code Blocks
- Keep code blocks exactly as-is
- Only translate Korean comments within code blocks to English
- Do NOT modify any code logic

### Markup That Must Survive Translation
- Hugo shortcodes are copied verbatim, including the path inside them: `[text]({{< ref "/posts/slug" >}})` stays `{{< ref "/posts/slug" >}}` — `ref` resolves to the English version automatically when one exists. Translate only the link text.
- Image references are relative to the page bundle (`![alt](images/name.svg)`) and must not be rewritten. Translate the alt text only.
- Diagram text inside SVG files is already English — never edit the SVGs.
- The Korean source glosses terms as `**한글용어**(English)`. In English the gloss is redundant: collapse it to the bolded English term, e.g. `**가용성**(availability)` → `**availability**`.
- Em dash `—` stays `—`, never `--`.

3. **Write** the translated content to `content/posts/$ARGUMENTS/index.md`.

4. **Report** what was done:
```
## Translation Complete: {post title}

- Source: content/posts/{slug}/index.ko.md
- Target: content/posts/{slug}/index.md
- Draft status: {true/false}
- Word count (approx): {N}
```

## Rules

- If `index.md` already exists, read it first and warn the user before overwriting.
- Never produce translationese — the output should read as if originally written in English.
- Preserve paragraph breaks and formatting exactly.
- Do not add or remove content — translate what exists.
- Section headings translate to English, including `마치며` → `Closing Thoughts` and `References` → `References`.
- `draft` must match the Korean version. If the Korean version is published and this translation is new, ask before publishing it in the same commit.
