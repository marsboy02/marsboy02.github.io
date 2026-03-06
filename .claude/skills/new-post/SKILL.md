---
name: new-post
description: Scaffold a new blog post with directory, front matter, and topic outline.
user_invocable: true
arguments: "<slug> [title]"
---

# New Post Scaffolding Skill

You are creating the scaffolding for a new Hugo blog post.

## Input

- `$ARGUMENTS` = `<slug>` and optionally `[title]`
  - Example: `my-new-post "My New Post Title"`
  - If title is not provided, derive a readable title from the slug

## Steps

1. **Parse arguments**: Extract slug and optional title from `$ARGUMENTS`.

2. **Check for conflicts**: Verify `content/posts/{slug}/` does not already exist using Glob. If it exists, warn the user and stop.

3. **Scan existing tags**: Read front matter from several existing posts in `content/posts/*/index.ko.md` to collect the set of tags already used in the blog. This ensures tag consistency.

4. **Create directory and files**:

### `content/posts/{slug}/index.ko.md`
```markdown
---
title: "{title}"
date: {today's date in YYYY-MM-DD format}
draft: true
tags: [{suggest 3-5 relevant tags from existing tags + new ones if needed}]
translationKey: "{slug}"
summary: ""
---

## Introduction
<!-- Why this topic matters, what problem it solves -->

## {Main Section 1}
<!-- Core concept or explanation -->

## {Main Section 2}
<!-- Deep dive, examples, or implementation -->

## {Main Section 3}
<!-- Advanced topics, best practices, or comparisons -->

## Conclusion
<!-- Key takeaways and next steps -->
```

### `content/posts/{slug}/index.md`
```markdown
---
title: "{title in English}"
date: {today's date}
draft: true
tags: [{same tags}]
translationKey: "{slug}"
summary: ""
---

(English translation placeholder)
```

5. **Generate a topic outline**: Based on the slug/title, suggest a more detailed outline with specific section headings relevant to the topic. Look at similar existing posts for structural inspiration.

6. **Report**:
```
## New Post Created: {title}

- Directory: content/posts/{slug}/
- Files created:
  - index.ko.md (Korean, draft)
  - index.md (English placeholder, draft)
- Suggested tags: [...]
- Suggested outline provided above

Next steps:
1. Fill in the summary field
2. Write the content in index.ko.md
3. Run /tech-review {slug} when ready
```

## Rules

- Always set `draft: true` for new posts.
- Use today's actual date.
- Tags should reuse existing blog tags where applicable.
- The outline should be specific to the topic, not generic.
- Do NOT generate actual content — only scaffolding and suggestions.
