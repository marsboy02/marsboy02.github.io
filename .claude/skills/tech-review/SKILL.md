---
name: tech-review
description: Review a blog post for technical accuracy — code correctness, terminology, deprecated APIs, and reference validity.
user_invocable: true
arguments: "<post-slug>"
---

# Tech Review Skill

You are reviewing a Hugo blog post for technical accuracy.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Post path: `content/posts/$ARGUMENTS/index.ko.md`

## Steps

1. **Read the post** at `content/posts/$ARGUMENTS/index.ko.md`. If not found, check `content/posts/$ARGUMENTS/index.md`.

2. **Review the following categories:**

### A. Code Correctness
- Syntax errors in code blocks (check language annotation)
- Code that would not compile or run as written
- Missing imports, incorrect function signatures, wrong return types
- Misleading variable names or logic errors

### B. Technical Terminology
- Incorrect or imprecise use of technical terms
- Conflation of related but distinct concepts
- Terms used inconsistently throughout the post

### C. Currency & Deprecation
- Deprecated APIs, libraries, or approaches
- Outdated version references
- Practices that have been superseded by better alternatives
- Check against your knowledge of current best practices

### D. Accuracy of Explanations
- Oversimplifications that could mislead readers
- Factual errors in how technologies work
- Missing important caveats or edge cases

### E. References & Links
- Extract all URLs from the post
- For internal links (other posts, images), verify they exist using Glob
- Note any claims that should have a reference but don't

3. **Output a structured report:**

```
## Tech Review: {post title}

### Critical Issues
(Must fix before publishing — factual errors, broken code)
- [ ] ...

### Warnings
(Should fix — deprecated content, imprecise terms)
- [ ] ...

### Suggestions
(Nice to have — additional context, better examples)
- [ ] ...

### Summary
- Code blocks reviewed: N
- Issues found: N critical, N warnings, N suggestions
```

## Rules

- Do NOT modify the post file. This is a read-only review.
- Be specific: quote the problematic text and explain why it's wrong.
- Provide the correct version when suggesting a fix.
- If the post is technically sound, say so — don't invent issues.
