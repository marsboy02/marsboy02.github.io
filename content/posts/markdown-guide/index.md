---
title: "Markdown Guide"
date: 2026-03-02
draft: false
tags: ["markdown", "guide"]
translationKey: "markdown-guide"
summary: "A quick reference for all the Markdown syntax you can use in this blog."
---

This blog uses Hugo's built-in Markdown renderer ([Goldmark](https://github.com/yuin/goldmark)), which supports the full CommonMark spec plus some extras. Here's what you can do.

## Headings

```markdown
## Heading 2
### Heading 3
#### Heading 4
```

## Heading 2
### Heading 3
#### Heading 4

---

## Text Formatting

```markdown
This is **bold**, this is *italic*, and this is ***bold italic***.

This is ~~strikethrough~~.
```

This is **bold**, this is *italic*, and this is ***bold italic***.

This is ~~strikethrough~~.

---

## Links and Images

```markdown
[Link text](https://example.com)

![Alt text](/images/example.jpg)
```

[Hugo Documentation](https://gohugo.io/documentation/)

---

## Lists

### Unordered

```markdown
- First item
- Second item
  - Nested item
  - Another nested item
- Third item
```

- First item
- Second item
  - Nested item
  - Another nested item
- Third item

### Ordered

```markdown
1. First
2. Second
3. Third
```

1. First
2. Second
3. Third

### Task List

```markdown
- [x] Completed task
- [ ] Incomplete task
- [ ] Another task
```

- [x] Completed task
- [ ] Incomplete task
- [ ] Another task

---

## Blockquotes

```markdown
> This is a blockquote.
>
> It can span multiple paragraphs.
```

> This is a blockquote.
>
> It can span multiple paragraphs.

Nested blockquotes:

> First level
>
> > Second level
> >
> > > Third level

---

## Tables

```markdown
| Left | Center | Right |
|:-----|:------:|------:|
| A    |   B    |     C |
| D    |   E    |     F |
```

| Left | Center | Right |
|:-----|:------:|------:|
| A    |   B    |     C |
| D    |   E    |     F |

---

## Horizontal Rule

Three dashes make a horizontal line:

```markdown
---
```

---

## Inline Code

Use backticks for `inline code` like variable names or commands:

```markdown
Run `hugo server` to start the dev server.
```

Run `hugo server` to start the dev server.

---

## Footnotes

```markdown
This has a footnote[^1].

[^1]: Here is the footnote content.
```

This has a footnote[^1].

[^1]: Here is the footnote content.

---

## Emoji

Hugo supports emoji shortcodes if enabled:

```markdown
:wave: :rocket: :thumbsup:
```

You can enable this by adding `enableEmoji: true` in `hugo.yaml`.

---

## Summary

That covers most of what you'll need for everyday writing. For the complete CommonMark spec, visit [commonmark.org](https://commonmark.org/). For Hugo-specific features like shortcodes, check the [Hugo documentation](https://gohugo.io/content-management/shortcodes/).
