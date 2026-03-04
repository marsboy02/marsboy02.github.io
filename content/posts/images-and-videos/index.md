---
title: "Embedding Images & Videos"
date: 2026-03-04T08:00:00+09:00
draft: false
tags: ["media", "guide"]
translationKey: "images-and-videos"
summary: "How to add images, figures with captions, and embed videos from YouTube and Vimeo."
---

This post covers all the ways to embed visual media in your blog posts.

## Images

### Basic Markdown Image

The simplest way to add an image:

```markdown
![A mountain landscape](/images/example.jpg)
```

The image file goes in `static/images/` and is referenced with an absolute path.

### Image with Title

```markdown
![Alt text](/images/photo.jpg "Optional tooltip title")
```

### Hugo Figure Shortcode

For more control — captions, links, sizing — use Hugo's built-in `figure` shortcode:

```
{{</* figure src="/images/photo.jpg" alt="Description" caption="Photo by Someone on Unsplash" */>}}
```

This renders as an HTML `<figure>` with `<figcaption>`, which is semantically correct and looks great.

#### Figure Options

| Parameter | Description |
|-----------|-------------|
| `src` | Image path (required) |
| `alt` | Alt text for accessibility |
| `caption` | Caption displayed below the image |
| `title` | Tooltip on hover |
| `width` | Width (e.g., `"300px"` or `"50%"`) |
| `height` | Height |
| `link` | URL to link the image to |
| `class` | CSS class name |

Example with multiple options:

```
{{</* figure
  src="/images/screenshot.png"
  alt="Blog homepage screenshot"
  caption="The blog homepage in dark mode"
  width="600px"
*/>}}
```

### Image Sizing with HTML

If you need precise control, you can use raw HTML in Markdown:

```html
<img src="/images/photo.jpg" alt="Description" width="400">
```

> **Note**: Hugo's default Markdown renderer (Goldmark) renders raw HTML by default. If it doesn't work, ensure `unsafe: true` is set under `markup.goldmark.renderer` in `hugo.yaml`.

---

## Videos

### YouTube

Hugo has a built-in shortcode for YouTube:

```
{{</* youtube dQw4w9WgXcQ */>}}
```

Just paste the video ID (the part after `v=` in the YouTube URL). This generates a responsive iframe embed.

{{< youtube dQw4w9WgXcQ >}}

### Vimeo

Similarly for Vimeo:

```
{{</* vimeo 146022717 */>}}
```

### Self-Hosted Video

For videos stored in your `static/` directory:

```html
<video controls width="100%">
  <source src="/videos/demo.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
```

Place the video file at `static/videos/demo.mp4`.

### Video with Poster Image

```html
<video controls width="100%" poster="/images/video-thumbnail.jpg">
  <source src="/videos/demo.mp4" type="video/mp4">
  <source src="/videos/demo.webm" type="video/webm">
</video>
```

---

## Organizing Media Files

Recommended directory structure:

```
static/
├── images/
│   ├── profile.jpg
│   ├── posts/
│   │   ├── my-post/
│   │   │   ├── screenshot-1.png
│   │   │   └── screenshot-2.png
│   │   └── another-post/
│   │       └── diagram.svg
│   └── ...
└── videos/
    └── demo.mp4
```

Or use **page bundles** — put images alongside the Markdown file:

```
content/
└── posts/
    └── my-post/
        ├── index.md
        ├── featured.jpg
        └── diagram.png
```

In a page bundle, reference images with relative paths:

```markdown
![Diagram](diagram.png)
```

---

## Tips

1. **Always add alt text** — It helps with accessibility and SEO.
2. **Optimize image size** — Use tools like [Squoosh](https://squoosh.app/) or ImageOptim before uploading.
3. **Use WebP or AVIF** — Modern formats are significantly smaller than JPEG/PNG.
4. **Lazy loading** — Add `loading="lazy"` to images below the fold: `<img src="..." loading="lazy">`.
5. **SVG for diagrams** — Use SVG format for diagrams and icons; they scale perfectly and have tiny file sizes.
