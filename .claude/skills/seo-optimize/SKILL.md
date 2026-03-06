---
name: seo-optimize
description: Audit a blog post for SEO — title, summary, headings, images, tags, and content length.
user_invocable: true
arguments: "<post-slug>"
---

# SEO Optimization Skill

You are auditing a Hugo blog post for search engine optimization.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Post path: `content/posts/$ARGUMENTS/index.ko.md` (also check `index.md` if it exists)

## Steps

1. **Read the post** at `content/posts/$ARGUMENTS/index.ko.md`. Also read `index.md` if it exists.

2. **Evaluate 8 SEO criteria:**

### 1. Title (score /10)
- Length: 50-60 characters recommended
- Contains primary keyword?
- Clear and compelling?

### 2. Summary / Meta Description (score /10)
- Length: 150-160 characters recommended
- Contains primary keyword?
- Compelling call-to-read?
- If empty: FAIL

### 3. Heading Hierarchy (score /10)
- Proper H2 > H3 > H4 nesting (no skipped levels)
- Headings contain relevant keywords?
- Not too many or too few headings for content length

### 4. Image Alt Text (score /10)
- All images have alt text?
- Alt text is descriptive (not just "image1")?
- If no images: N/A

### 5. Links (score /10)
- Has internal links to other posts?
- Has external links to authoritative sources?
- Reasonable link density

### 6. Tags (score /10)
- 3-5 tags present?
- Tags are relevant to content?
- Tags match common search terms?

### 7. Content Length (score /10)
- Word count (Korean characters counted appropriately)
- Sufficient depth for the topic?
- Not too thin (< 500 words) or excessively long without structure

### 8. URL Slug (score /10)
- Descriptive and keyword-rich?
- Uses hyphens (not underscores)?
- Not too long (3-5 words ideal)

3. **Output a scorecard:**

```
## SEO Audit: {post title}

| Criterion | Score | Status | Notes |
|-----------|-------|--------|-------|
| Title | /10 | ... | ... |
| Summary | /10 | ... | ... |
| Heading Hierarchy | /10 | ... | ... |
| Image Alt Text | /10 | ... | ... |
| Links | /10 | ... | ... |
| Tags | /10 | ... | ... |
| Content Length | /10 | ... | ... |
| URL Slug | /10 | ... | ... |
| **Overall** | **/80** | | |

### Priority Fixes
1. ...
2. ...
3. ...

### Quick Wins
- ...
```

## Rules

- Do NOT modify the post file. This is a read-only audit.
- Be actionable: for each issue, explain exactly what to change.
- If both KO and EN versions exist, audit both and note discrepancies.
- Score honestly — a perfect score should be rare.
