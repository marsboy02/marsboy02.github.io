---
name: link-checker
description: Verify all links in a post (internal, external, images) and report broken ones.
user_invocable: true
arguments: "<post-slug> | all"
---

# Link Checker Skill

You verify all links in blog posts and report broken or problematic ones.

## Input

- `$ARGUMENTS` = a post slug (e.g., `docker-complete-guide`) or `all`

## Steps

### 1. Collect Posts

- If `$ARGUMENTS` is a slug: read `content/posts/$ARGUMENTS/index.ko.md` (and `index.md` if exists)
- If `$ARGUMENTS` is `all`: read all `content/posts/*/index.ko.md` files

### 2. Extract Links

From each post, extract all:
- **Internal post links**: `](/posts/other-slug/)` or relative links
- **Image references**: `![alt](image.png)` or `![alt](/images/...)`
- **External URLs**: `https://...`, `http://...`
- **Anchor links**: `#section-heading`

### 3. Verify Each Link Type

#### Internal Post Links
- Use Glob to verify the target post directory exists
- Check: `content/posts/{linked-slug}/index.ko.md` exists

#### Image References
- For relative images (e.g., `image.png`): check file exists in `content/posts/{slug}/`
- For absolute images (e.g., `/images/...`): check file exists in `static/images/`
- Use Glob for verification

#### External URLs
- Use WebFetch to check each URL (HEAD request or fetch)
- Note: rate-limit external checks, do max 10 external URLs per run
- If more than 10 external URLs, check the first 10 and note the rest as unchecked

#### Anchor Links
- Verify the heading exists in the same document by checking markdown headings

### 4. Output Report

```
## Link Check: {post title or "All Posts"}

### Broken Links
| Post | Link | Type | Issue |
|------|------|------|-------|
| {slug} | {url} | {internal/image/external} | {not found / 404 / timeout} |

### Warnings
| Post | Link | Type | Issue |
|------|------|------|-------|
| {slug} | {url} | {type} | {redirect / slow / unchecked} |

### OK
- Internal links: N checked, N ok
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
- If WebFetch fails or times out, classify as "warning" not "broken".
- Group results by post when checking `all`.
- Report the exact line or context where broken links appear.
