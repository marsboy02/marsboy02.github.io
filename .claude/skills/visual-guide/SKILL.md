---
name: visual-guide
description: Suggest visual aids for a blog post — SVG diagrams, tables, and ASCII art — and generate the ones this site can actually render.
argument-hint: "<post-slug>"
allowed-tools: Read Glob Grep
---

# Visual Guide Skill

You are analyzing a blog post to suggest and generate visual aids that improve comprehension.

## What this site can render

Check this before suggesting a format — it decides everything below.

- **Inline SVG files** — the blog's actual diagram format. Files live in the post bundle's `images/` directory and are referenced as `![alt](images/name.svg)`. Use `/svg-diagram <slug> "<description>"` to author them; that skill holds the palette, sizing, and content rules.
- **Markdown tables** — fully supported.
- **ASCII / box-drawing art inside a fenced code block** — supported, renders in the site's monospace font.
- **Raw HTML** — `goldmark` runs with `unsafe: true`, so inline HTML works when a table cannot express the layout.
- **Mermaid — NOT rendered.** There is no Mermaid JS in `layouts/` or the `gandanham` theme, no codeblock render hook, and no post uses a ```mermaid fence. A Mermaid block would ship to readers as a plain code block. Do not suggest Mermaid as a finished visual. If a diagram is genuinely best expressed as Mermaid, say so explicitly and note that it requires adding `layouts/_default/_markup/render-codeblock-mermaid.html` plus the Mermaid script to the head partial first — then let the author decide.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Post path: `content/posts/$ARGUMENTS/index.ko.md`

## Steps

1. **Read the post** at `content/posts/$ARGUMENTS/index.ko.md`.

2. **Inventory what the post already has**: list the files in `content/posts/$ARGUMENTS/images/` and note which sections already carry a diagram. Do not propose a visual for a section that already has one.

3. **Scan for visual opportunities** in these categories:

### A. Architecture / Components
- System components and their relationships, infrastructure layouts, layered architectures
- **Format**: SVG (`/svg-diagram` type A or E)

### B. Process / Flow
- Step-by-step procedures, request/response flows, build/deploy pipelines
- **Format**: SVG (`/svg-diagram` type B)

### C. Comparisons
- Before/after states, technology comparisons, trade-off analysis
- **Format**: Markdown table — or SVG side-by-side when the comparison is structural rather than tabular

### D. Hierarchies / Precedence
- Nesting, precedence order, abstraction levels
- **Format**: SVG (`/svg-diagram` type C) or a nested list

### E. Sequences / Timelines
- Event ordering, protocol handshakes, lifecycle stages
- **Format**: SVG (`/svg-diagram` type D)

### F. Dense reference data
- Command flags, config keys, size/latency numbers
- **Format**: Markdown table

4. **Generate what you can here, and hand off what you can't.**

Tables and ASCII art: produce the finished block, ready to paste. SVG suggestions: describe the diagram precisely enough that `/svg-diagram $ARGUMENTS "<description>"` can produce it in one shot, and include the exact `![alt](images/name.svg)` line and insertion point.

5. **Output:**

```
## Visual Guide: {post title}

### Existing visuals
- images/{file} — used in "{section}"

### Suggested Visuals

#### 1. {Description}
- **Type**: {SVG via /svg-diagram | Table | ASCII art | HTML}
- **Where to insert**: After "{section heading}"
- **Why**: {How this helps the reader}
- **Next step**: {paste the block below | run /svg-diagram {slug} "{description}"}

{Complete table / ASCII block, or the SVG brief + the image reference line}

#### 2. {Description}
...

---

### Summary
- Visuals suggested: N
- Types: N SVG briefs, N tables, N ASCII art
- Rendering caveats: {none | Mermaid requested but unsupported}
```

## Rules

- Do NOT modify the post file. Output suggestions only.
- Never suggest a format this site cannot render without saying so in the same breath.
- Tables should use proper markdown formatting.
- Text inside diagrams is English-only, even in Korean posts. Table and ASCII content follows the post's language.
- Keep diagrams simple and focused — one concept per diagram. Wide-and-short beats tall-and-square for blog readability.
- Suggest placement locations (after which heading/paragraph).
- Prioritize visuals that replace or complement dense text explanations, not decoration.
