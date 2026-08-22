---
name: link-checker
description: Verify all links in a post (internal, external, images) and report broken ones.
argument-hint: "<post-slug> | all"
allowed-tools: Read Glob Grep Bash(curl -sI *)
context: fork
agent: Explore
background: false
---

# Link Checker Skill

You verify all links in blog posts and report broken or problematic ones.

## Input

- `$ARGUMENTS` = a post slug (e.g., `docker-complete-guide`) or `all`

## Steps

### 1. Collect Posts

- If `$ARGUMENTS` is a slug: read `content/posts/$ARGUMENTS/index.ko.md`, plus `index.md` and `index.ja.md` if they exist
- If `$ARGUMENTS` is `all`: read all `content/posts/*/index*.md` files

### 2. Extract Links

This blog links internally with Hugo shortcodes, not raw paths. Extract all of:

- **Internal post links (shortcode form — the convention here)**:
  - `[text]({{< ref "/posts/other-slug" >}})`
  - `[text]({{< relref "/posts/other-slug" >}})`
  - Both may carry a fragment: `{{< ref "/posts/other-slug#heading" >}}`
- **Internal post links (raw path form)**: `](/posts/other-slug/)` — rare/legacy; flag it as a convention warning and suggest the shortcode form
- **Post-bundle images**: `![alt](images/name.png)` — the standard form; files live in the post's own `images/` directory
- **Site-wide images**: `![alt](/images/name.png)` — files live in `static/images/`
- **External URLs**: `https://...`, `http://...`
- **Anchor links**: `#section-heading`

Useful starting sweep:

```bash
grep -o '{{< *relref\? *"[^"]*"' content/posts/*/index*.md
grep -o '!\[[^]]*\]([^)]*)' content/posts/*/index*.md
```

### 3. Verify Each Link Type

#### Internal Post Links
- Extract the slug from the shortcode path (strip a leading `/posts/` and any `#fragment`)
- Verify `content/posts/{slug}/` exists and contains at least one `index*.md`
- Language note: `ref`/`relref` resolve within the current language first. If the linking file is `index.ja.md` but the target has no `index.ja.md`, note it as a warning — Hugo falls back, but the reader lands on another language
- If a fragment is present, verify the heading exists in the target post

#### Images
- `](images/foo.png)`: verify `content/posts/{slug}/images/foo.png` exists
- `](/images/foo.png)`: verify `static/images/foo.png` exists
- For `.svg` files, also confirm the file is non-empty and well-formed XML

#### External URLs
- Check with `curl -sI -m 10 -o /dev/null -w '%{http_code}' <url>` (HEAD request)
- If the HEAD request returns 405 or 403, retry once with `curl -sI -m 10 -L`
- Cap at 20 external URLs per run; report the rest as unchecked
- Space out requests to the same host

#### Anchor Links
- Verify the heading exists in the same document by checking markdown headings
- Remember Hugo's anchor rules: lowercase, spaces to hyphens, punctuation dropped. Korean headings keep their characters

### 4. Output Report

```
## Link Check: {post title or "All Posts"}

### Broken Links
| Post | Link | Type | Issue |
|------|------|------|-------|
| {slug} | {url} | {shortcode/image/external/anchor} | {not found / 404 / timeout} |

### Warnings
| Post | Link | Type | Issue |
|------|------|------|-------|
| {slug} | {url} | {type} | {redirect / raw path instead of shortcode / missing target language / unchecked} |

### OK
- Internal shortcode links: N checked, N ok
- Images: N checked, N ok
- External URLs: N checked, N ok, N unchecked
- Anchors: N checked, N ok

### Summary
- Total links found: N
- Broken: N
- Warnings: N
- OK: N
```

## Rules

- Do NOT modify any files. This is read-only.
- For external URLs, be respectful: don't make excessive requests.
- If a request times out or the tool fails, classify as "warning" not "broken".
- Group results by post when checking `all`.
- Report the exact line or context where broken links appear.
- A raw `](/posts/...)` link is not broken if the target exists, but it deviates from the blog's shortcode convention — report it as a warning.
