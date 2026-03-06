---
name: draft-manager
description: Dashboard of draft posts and translation status, or pre-publish checklist for a specific post.
user_invocable: true
arguments: "[post-slug]"
---

# Draft Manager Skill

You manage draft posts and run pre-publish checklists.

## Input

- `$ARGUMENTS` = empty (dashboard mode) or a post slug (checklist mode)

## Mode 1: Dashboard (no arguments)

1. **Scan all posts**: Read front matter from all `content/posts/*/index.ko.md` files.

2. **Check translation status**: For each post, check existence of:
   - `index.ko.md` (Korean)
   - `index.md` (English)
   - `index.ja.md` (Japanese)

3. **Output dashboard:**

```
## Draft Manager Dashboard

### Draft Posts
| Post | Draft | KO | EN | JA | Summary | Tags |
|------|-------|----|----|----|---------|------|
| {slug} | {yes/no} | {check/x} | {check/x} | {check/x} | {yes/no} | {count} |

### Statistics
- Total posts: N
- Published: N
- Drafts: N
- Fully translated (KO+EN): N
- Missing English translation: N
- Missing summary: N

### Action Items
1. {Most impactful action, e.g., "Publish docker-complete-guide — it's complete"}
2. ...
```

## Mode 2: Pre-publish Checklist (with post-slug)

1. **Read the post** at `content/posts/$ARGUMENTS/index.ko.md`.

2. **Run checklist:**

```
## Pre-publish Checklist: {post title}

### Front Matter
- [ ] title is set and descriptive
- [ ] date is set
- [ ] summary is filled (not empty)
- [ ] tags are present (3-5 recommended)
- [ ] translationKey matches slug

### Content Quality
- [ ] Post has an introduction section
- [ ] Post has a conclusion/summary section
- [ ] Heading hierarchy is correct (H2 > H3, no skips)
- [ ] No TODO/FIXME/placeholder comments remain
- [ ] Code blocks have language annotations

### Media
- [ ] All referenced images exist in the post directory
- [ ] Images have alt text

### Translations
- [ ] English version (index.md) exists
- [ ] English front matter (title, summary) is translated
- [ ] Japanese version (index.ja.md) — optional

### Links
- [ ] Internal links point to existing posts
- [ ] No broken image references

### Ready to Publish?
{YES — all critical checks pass / NO — N issues to fix}

To publish, set `draft: false` in the front matter.
```

3. **For each check**, actually verify it (read files, check Glob results, parse content).

## Rules

- Do NOT modify any files. This is read-only.
- In dashboard mode, sort posts by draft status (drafts first), then by date.
- Use checkmarks and X marks for clear visual status.
- Be honest about missing items — don't skip checks.
