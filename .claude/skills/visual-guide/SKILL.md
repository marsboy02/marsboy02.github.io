---
name: visual-guide
description: Suggest and generate visual aids (Mermaid diagrams, tables, ASCII art) for a blog post.
user_invocable: true
arguments: "<post-slug>"
---

# Visual Guide Skill

You are analyzing a blog post to suggest and generate visual aids that improve comprehension.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Post path: `content/posts/$ARGUMENTS/index.ko.md`

## Steps

1. **Read the post** at `content/posts/$ARGUMENTS/index.ko.md`.

2. **Scan for visual opportunities** in these categories:

### A. Architecture Diagrams
- System components and their relationships
- Infrastructure layouts (servers, services, databases)
- Layered architectures
- **Format**: Mermaid `graph TD` or `graph LR`

### B. Process / Flow
- Step-by-step procedures
- Request/response flows
- Build/deploy pipelines
- Decision trees
- **Format**: Mermaid `flowchart` or `sequenceDiagram`

### C. Comparisons
- Before/after states
- Technology comparisons
- Pros/cons analysis
- **Format**: Markdown table

### D. Data Structures
- Object relationships
- Class hierarchies
- Data models
- **Format**: Mermaid `classDiagram` or `erDiagram`

### E. Sequences / Timelines
- Event ordering
- Protocol handshakes
- Lifecycle stages
- **Format**: Mermaid `sequenceDiagram` or `timeline`

3. **Generate ready-to-use visual aids:**

For each opportunity found, produce the complete code that can be directly pasted into the post.

4. **Output:**

```
## Visual Guide: {post title}

### Suggested Visuals

#### 1. {Description}
- **Type**: {Mermaid diagram / Table / ASCII art}
- **Where to insert**: After "{section heading}" section
- **Why**: {How this helps the reader}

{Complete Mermaid/table/ASCII code block}

#### 2. {Description}
...

---

### Summary
- Visuals suggested: N
- Types: N diagrams, N tables, N ASCII art
```

## Rules

- Do NOT modify the post file. Output suggestions only.
- All Mermaid code must be valid and render correctly.
- Tables should use proper markdown formatting.
- Keep diagrams simple and focused — don't try to show everything.
- Suggest placement locations (after which heading/paragraph).
- Prioritize visuals that replace or complement dense text explanations.
