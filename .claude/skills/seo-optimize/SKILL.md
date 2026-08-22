---
name: seo-optimize
description: Audit a blog post for SEO — title, summary, headings, images, tags, and content length.
argument-hint: "<post-slug>"
allowed-tools: Read Glob Grep
---

# SEO Optimization Skill

You are auditing a Hugo blog post for search engine optimization.

## Input

- `$ARGUMENTS` = post slug (e.g., `docker-complete-guide`)
- Post path: `content/posts/$ARGUMENTS/index.ko.md` (also check `index.md` if it exists)

## How this site actually emits metadata

Audit against the real templates, not generic SEO advice:

- `layouts/_partials/head/og.html` builds OpenGraph and Twitter tags. The description resolves as `.Description → .Summary → site.Params.description`, plainified and **truncated at 200 characters**. Posts here set `summary:` in front matter, so `summary` is the meta description in practice.
- OG image resolves as `.Params.images[0]` (looked up as a page resource first, so `images: ["images/diagram.svg"]` works from the bundle) and falls back to `site.Params.images[0]` = `/images/og-default.jpg`. Only one post currently sets a per-post image, so this is usually the highest-leverage fix available.
- `layouts/_partials/head.html` emits `<title>` as `{Title} | {site.Title}`, favicons, and the OG partial. It does **not** emit `<meta name="description">`, `<link rel="canonical">`, or `hreflang` alternates. Those are site-level gaps — report them once under "Site-level findings", never as a per-post defect the author can fix in front matter.
- Hugo generates per-language sitemaps (`/sitemap.xml`, `/ko/sitemap.xml`, `/ja/sitemap.xml`). There is no `static/robots.txt`.

## Steps

1. **Read the post** at `content/posts/$ARGUMENTS/index.ko.md`. Also read `index.md` if it exists.

2. **Evaluate 8 SEO criteria:**

### 1. Title (score /10)
- Length: 50-60 characters recommended
- Contains primary keyword?
- Clear and compelling?

### 2. Summary / Meta Description (score /10)
- This field is the meta description on this site, and it is truncated at 200 characters — aim for 120-200 and make sure the first 120 stand alone
- Written in 평어체, like the body
- Contains primary keyword?
- Compelling call-to-read?
- If empty: FAIL

### 3. Heading Hierarchy (score /10)
- Proper H2 > H3 > H4 nesting (no skipped levels)
- Headings contain relevant keywords?
- Not too many or too few headings for content length

### 4. Images & Social Card (score /10)
- All images have descriptive alt text (not "image1")?
- Bundle images resolve (`![alt](images/name.svg)` → `content/posts/{slug}/images/`)?
- Does the post set `images:` in front matter for its social card? If not, it falls back to the generic `/images/og-default.jpg` — recommend a concrete candidate from the post's own `images/` directory, preferring a wide raster or SVG that reads at card size
- If no images: N/A

### 5. Links (score /10)
- Has internal links to other posts, in the `{{< ref "/posts/slug" >}}` shortcode form?
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
- `translationKey` matches the slug across all language versions, so the language switcher lands on the right page

3. **Output a scorecard:**

```
## SEO Audit: {post title}

| Criterion | Score | Status | Notes |
|-----------|-------|--------|-------|
| Title | /10 | ... | ... |
| Summary | /10 | ... | ... |
| Heading Hierarchy | /10 | ... | ... |
| Images & Social Card | /10 | ... | ... |
| Links | /10 | ... | ... |
| Tags | /10 | ... | ... |
| Content Length | /10 | ... | ... |
| URL Slug | /10 | ... | ... |
| **Overall** | **/80** | | |

### Site-level findings
(Not fixable in this post's front matter — report only when relevant, and don't count them in the score)
- ...

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
- If KO, EN, and JA versions exist, audit all of them and note discrepancies — a missing or untranslated `summary` in one language means that language ships a fallback meta description.
- Score honestly — a perfect score should be rare.
