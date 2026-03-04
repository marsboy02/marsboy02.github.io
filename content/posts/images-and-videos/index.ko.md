---
title: "이미지와 동영상 삽입하기"
date: 2026-03-04T08:00:00+09:00
draft: false
tags: ["media", "guide"]
translationKey: "images-and-videos"
summary: "이미지, 캡션이 있는 figure, YouTube/Vimeo 동영상을 삽입하는 방법을 소개합니다."
---

이 글에서는 블로그 포스트에 시각적 미디어를 삽입하는 모든 방법을 다룹니다.

## 이미지

### 기본 마크다운 이미지

이미지를 추가하는 가장 간단한 방법:

```markdown
![산 풍경](/images/example.jpg)
```

이미지 파일은 `static/images/`에 넣고 절대 경로로 참조합니다.

### 제목이 있는 이미지

```markdown
![대체 텍스트](/images/photo.jpg "마우스를 올리면 보이는 제목")
```

### Hugo Figure 숏코드

캡션, 링크, 크기 조절 등 더 많은 제어가 필요할 때 Hugo의 내장 `figure` 숏코드를 사용합니다:

```
{{</* figure src="/images/photo.jpg" alt="설명" caption="Unsplash의 Someone 촬영" */>}}
```

이것은 HTML `<figure>`와 `<figcaption>`으로 렌더링되어 의미론적으로 올바르고 보기에도 좋습니다.

#### Figure 옵션

| 파라미터 | 설명 |
|----------|------|
| `src` | 이미지 경로 (필수) |
| `alt` | 접근성을 위한 대체 텍스트 |
| `caption` | 이미지 아래에 표시되는 캡션 |
| `title` | 마우스를 올렸을 때 나타나는 툴팁 |
| `width` | 너비 (예: `"300px"` 또는 `"50%"`) |
| `height` | 높이 |
| `link` | 이미지를 클릭할 때 이동할 URL |
| `class` | CSS 클래스 이름 |

여러 옵션을 함께 사용하는 예시:

```
{{</* figure
  src="/images/screenshot.png"
  alt="블로그 홈페이지 스크린샷"
  caption="다크 모드에서의 블로그 홈페이지"
  width="600px"
*/>}}
```

### HTML로 이미지 크기 조절

정확한 크기 제어가 필요하면 마크다운 안에서 HTML을 직접 사용할 수 있습니다:

```html
<img src="/images/photo.jpg" alt="설명" width="400">
```

> **참고**: Hugo의 기본 마크다운 렌더러(Goldmark)는 기본적으로 HTML을 렌더링합니다. 작동하지 않으면 `hugo.yaml`의 `markup.goldmark.renderer`에서 `unsafe: true`를 설정하세요.

---

## 동영상

### YouTube

Hugo에는 YouTube용 내장 숏코드가 있습니다:

```
{{</* youtube dQw4w9WgXcQ */>}}
```

YouTube URL에서 `v=` 뒤의 동영상 ID만 붙여넣으면 됩니다. 반응형 iframe이 생성됩니다.

{{< youtube dQw4w9WgXcQ >}}

### Vimeo

Vimeo도 마찬가지입니다:

```
{{</* vimeo 146022717 */>}}
```

### 직접 호스팅하는 동영상

`static/` 디렉토리에 저장된 동영상:

```html
<video controls width="100%">
  <source src="/videos/demo.mp4" type="video/mp4">
  브라우저가 video 태그를 지원하지 않습니다.
</video>
```

동영상 파일을 `static/videos/demo.mp4`에 배치합니다.

### 포스터 이미지가 있는 동영상

```html
<video controls width="100%" poster="/images/video-thumbnail.jpg">
  <source src="/videos/demo.mp4" type="video/mp4">
  <source src="/videos/demo.webm" type="video/webm">
</video>
```

---

## 미디어 파일 구성

권장 디렉토리 구조:

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

또는 **페이지 번들**을 사용하여 이미지를 마크다운 파일 옆에 배치할 수 있습니다:

```
content/
└── posts/
    └── my-post/
        ├── index.md
        ├── featured.jpg
        └── diagram.png
```

페이지 번들에서는 상대 경로로 이미지를 참조합니다:

```markdown
![다이어그램](diagram.png)
```

---

## 팁

1. **항상 대체 텍스트를 추가하세요** — 접근성과 SEO에 도움이 됩니다.
2. **이미지 크기를 최적화하세요** — 업로드 전에 [Squoosh](https://squoosh.app/)나 ImageOptim 같은 도구를 사용하세요.
3. **WebP나 AVIF를 사용하세요** — 최신 포맷은 JPEG/PNG보다 훨씬 작습니다.
4. **지연 로딩** — 스크롤해야 보이는 이미지에 `loading="lazy"`를 추가하세요: `<img src="..." loading="lazy">`.
5. **다이어그램은 SVG로** — 다이어그램과 아이콘에는 SVG 포맷을 사용하세요. 완벽하게 확대되며 파일 크기도 매우 작습니다.
