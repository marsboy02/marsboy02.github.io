---
name: content-enhance
description: Suggest storytelling and readability improvements for a blog post, inspired by the computer-architecture-story style.
user_invocable: true
arguments: "<post-slug>"
---

# Content Enhancement Skill

You are suggesting content improvements to make a blog post more engaging and readable.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Post path: `content/posts/$ARGUMENTS/index.ko.md`

## Steps

1. **Read the target post** at `content/posts/$ARGUMENTS/index.ko.md`.

2. **Read the reference post** at `content/posts/computer-architecture-story/index.ko.md` to understand the storytelling style this blog aspires to (narrative-driven, analogy-rich, reader-friendly).

3. **Evaluate the following dimensions:**

### A. Opening / Hook
- Does the introduction grab attention?
- Is there a relatable problem, question, or story?
- Could a personal experience or scenario improve it?

### B. Analogies & Examples
- Are abstract concepts paired with concrete analogies?
- Are there real-world examples that make ideas stick?
- Suggest specific analogies where they'd help.

### C. Flow & Transitions
- Do sections connect logically?
- Are there abrupt jumps between topics?
- Suggest transition sentences where needed.

### D. The "Why" Factor
- Does the post explain WHY, not just HOW?
- Are motivations and trade-offs discussed?
- Where could deeper reasoning be added?

### E. Visual Opportunities
- Where would a diagram, table, or comparison chart help?
- Are there walls of text that could be broken up?
- Suggest specific visual aids (Mermaid diagrams, tables, etc.)

### F. Conclusion & Takeaway
- Does the conclusion summarize key insights?
- Is there a clear takeaway for the reader?
- Could a "what's next" or call-to-action improve it?

4. **Output a structured report:**

```
## Content Enhancement: {post title}

### Opening
(Current state + specific suggestion)

### Analogies & Examples
(Opportunities for concrete examples)

### Flow & Transitions
(Gaps in logical progression)

### "Why" Factor
(Where to deepen reasoning)

### Visual Opportunities
(Specific diagram/table suggestions)

### Conclusion
(How to strengthen the ending)

---

### Priority Actions (Top 3)
1. ...
2. ...
3. ...
```

## Rules

- Do NOT modify the post file. This is a read-only review.
- Be specific: quote sections and provide concrete rewrite suggestions.
- Reference the `computer-architecture-story` style where relevant.
- Respect the author's voice — suggest enhancements, not rewrites.
- Focus on the 3 highest-impact improvements.
