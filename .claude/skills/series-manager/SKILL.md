---
name: series-manager
description: Analyze posts for series grouping or generate series navigation blocks.
user_invocable: true
arguments: "analyze | <series-name> <slug1> <slug2> ..."
---

# Series Manager Skill

You manage blog post series — grouping related posts and generating navigation.

## Input

- `$ARGUMENTS` can be:
  1. `analyze` — Analyze all posts and suggest series groupings
  2. `<series-name> <slug1> <slug2> ...` — Generate navigation for a specific series

## Mode 1: Analyze

When `$ARGUMENTS` is `analyze`:

1. **Read all posts**: Scan `content/posts/*/index.ko.md` front matter (title, tags, summary) and skim content.

2. **Cluster by topic**: Group posts by shared tags, related technologies, and thematic connections.

3. **Output series candidates:**

```
## Series Analysis

### Candidate Series

#### 1. {Series Name} (N posts)
- Suggested order:
  1. {post-slug-1} — {title} (foundational)
  2. {post-slug-2} — {title} (builds on #1)
  3. {post-slug-3} — {title} (advanced)
- Rationale: {Why these belong together}
- Missing topics: {What posts could be added to complete the series}

#### 2. {Series Name} ...

### Standalone Posts
- {slug} — {title}: {Why it stands alone}
```

## Mode 2: Generate Navigation

When `$ARGUMENTS` contains a series name and slugs:

1. **Read each post** to get titles and verify they exist.

2. **Generate a navigation block** for each post in the series:

```markdown
---

> **Series: {Series Name}**
>
> 1. [{Title 1}](/posts/{slug1}/) {current marker if this post}
> 2. [{Title 2}](/posts/{slug2}/)
> 3. [{Title 3}](/posts/{slug3}/)

---
```

3. **Generate a "Related Posts" section** for each post:

```markdown
## Related Posts
- [{Related Title}](/posts/{related-slug}/) - {one-line description}
```

4. **Output all navigation blocks** with instructions on where to paste them (typically at the end of each post, before the conclusion or after it).

```
## Series Navigation: {Series Name}

### For: {slug1}
{navigation block}

### For: {slug2}
{navigation block}

...

### Instructions
Paste the navigation block at the end of each post.
```

## Rules

- Do NOT modify post files. Output navigation blocks for the user to insert.
- In analyze mode, consider both explicit tags and implicit thematic connections.
- Series order should follow a logical learning progression (beginner to advanced).
- A post can belong to multiple series candidates.
- Keep navigation blocks concise — readers should scan them quickly.
