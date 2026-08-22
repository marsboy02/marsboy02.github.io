---
name: draft-manager
description: Dashboard of draft posts and translation status, or pre-publish checklist for a specific post.
argument-hint: "[post-slug]"
allowed-tools: Read Glob Grep Bash(grep *) Bash(ls *)
---

# Draft Manager Skill

You manage draft posts and run pre-publish checklists.

## Input

- `$ARGUMENTS` = empty (dashboard mode) or a post slug (checklist mode)

## Pre-fetched context

- All language files: !`ls content/posts/*/index*.md`
- Drafts (any language): !`grep -l "^draft: true" content/posts/*/index*.md`
- Empty summary: !`grep -l '^summary: ""' content/posts/*/index*.md`
- translationKey values: !`grep -H "^translationKey:" content/posts/*/index.ko.md`

Derive the KO/EN/JA matrix and the "missing translation" lists from the file listing above.
Use the data above as your starting point and read files only where you need more detail.

## Mode 1: Dashboard (no arguments)

1. **Scan all posts**: Read front matter from all `content/posts/*/index.ko.md` files.

2. **Check translation status**: For each post, check existence of, and draft state in:
   - `index.ko.md` (Korean — the primary source, written first)
   - `index.md` (English)
   - `index.ja.md` (Japanese)

   All three are configured languages in `hugo.yaml`, so a post is only fully translated when all three exist. A post whose Korean version is published while a translation is still `draft: true` is a real gap worth flagging.

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
- Fully translated (KO+EN+JA): N
- Missing English translation: N
- Missing Japanese translation: N
- Translation exists but still draft: N
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
- [ ] summary is filled (not empty) and written in 평어체
- [ ] tags are present (3-5 recommended), lowercase-hyphenated English
- [ ] translationKey matches slug and is identical across all language versions

### Content Quality
- [ ] Opens with an intro section body (no "## Introduction" heading — the blog starts with prose)
- [ ] Closes with a `## 마치며` section (summary → personal insight → future direction)
- [ ] Heading hierarchy is correct (H2 > H3, no skips)
- [ ] Body text uses 평어체 (~이다/~한다), not 경어체
- [ ] No TODO/FIXME/placeholder comments remain
- [ ] Code blocks have language annotations

### Media
- [ ] All referenced images exist in the post's `images/` directory
- [ ] Images have alt text
- [ ] SVG diagrams are well-formed and contain English text only

### Translations
- [ ] English version (index.md) exists and its front matter is translated
- [ ] Japanese version (index.ja.md) exists and its front matter is translated
- [ ] Translations are not left at `draft: true` while the Korean version publishes

### Links
- [ ] Internal links use the `{{< ref "/posts/slug" >}}` shortcode form
- [ ] Internal links point to existing posts
- [ ] No broken image references

### Ready to Publish?
{YES — all critical checks pass / NO — N issues to fix}

To publish, set `draft: false` in the front matter of every language version.
```

3. **For each check**, actually verify it (read files, check Glob results, parse content).

## Rules

- Do NOT modify any files. This is read-only.
- In dashboard mode, sort posts by draft status (drafts first), then by date.
- Use checkmarks and X marks for clear visual status.
- Be honest about missing items — don't skip checks.
- Treat Japanese as a first-class language, not an optional extra.
