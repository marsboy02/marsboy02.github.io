---
name: svg-diagram
description: Generate SVG diagrams for blog posts — architecture, flow, comparison, layer, and sequence diagrams with consistent styling.
user_invocable: true
arguments: "<post-slug> [description]"
---

# SVG Diagram Skill

You are generating publication-ready SVG diagrams for a Hugo blog post.

## Input

- `$ARGUMENTS` = `<post-slug>` and optionally `[description]`
  - Example: `docker-complete-guide "multi-stage build flow diagram"`
  - If description is not provided, scan the post for `<!-- TODO:` comments or sections that would benefit from a diagram, then ask which one to create.

## Steps

1. **Locate the post** at `content/posts/$ARGUMENTS/index.ko.md`. Read it to understand the context.

2. **Determine what to diagram**:
   - If a description was provided, use it.
   - Otherwise, scan for `<!-- TODO:` comments that mention diagrams, or identify sections with dense text that would benefit from visual explanation.
   - Ask the user to confirm before generating.

3. **Choose diagram type** from the catalog below and generate the SVG.

4. **Save the file** to `content/posts/$ARGUMENTS/images/{descriptive-name}.svg`.

5. **Insert into the post** by replacing the `<!-- TODO: ... -->` comment (if one exists) or suggesting where to place the image reference:
   ```markdown
   ![Alt text](images/{descriptive-name}.svg)
   ```

6. **Report** the file path and suggest the user open it in a browser to verify.

## Diagram Type Catalog

### A. Architecture / Comparison (side-by-side)
- Two systems shown side by side with a `vs` divider
- Stacked layers (hardware → OS → runtime → app)
- Use for: VM vs Container, monolith vs microservice, before/after

### B. Flow / Pipeline
- Left-to-right flow with arrows between stages
- Stages as rounded rectangles, artifacts as highlighted boxes
- Use for: CI/CD pipelines, build processes, request flows, data pipelines

### C. Layer / Priority (nested rectangles)
- Concentric rectangles showing hierarchy or priority
- Outer = higher priority or broader scope
- Use for: variable precedence, network layers, abstraction levels

### D. Sequence (vertical timeline)
- Vertical lifelines with horizontal arrows between participants
- Use for: API calls, handshakes, event sequences

### E. Component (boxes and arrows)
- Boxes representing services/components with directional arrows
- Use for: system architecture, service mesh, data flow between components

## SVG Style Guide

### Fonts
```xml
font-family="'Inter', 'Segoe UI', -apple-system, sans-serif"
<!-- For code snippets inside diagrams -->
font-family="'SF Mono', 'Fira Code', monospace"
```

### Color Palette
| Purpose         | Color   | Hex       |
|-----------------|---------|-----------|
| Primary Blue    | Info    | `#2980b9` |
| Green           | Success | `#27ae60` |
| Orange          | Warning | `#f39c12` |
| Red             | Danger  | `#e74c3c` |
| Purple          | Accent  | `#8e44ad` |
| Dark Gray       | Text    | `#1a1a2e` |
| Medium Gray     | Subtle  | `#555`    |
| Light Gray      | Note    | `#888`    |
| Dark BG (code)  | Code BG | `#2d2d2d` |

### Background Fills (light tints for containers)
| Color       | Hex       |
|-------------|-----------|
| Blue tint   | `#e8f4f8` |
| Green tint  | `#eaf7ee` |
| Orange tint | `#fff3e0` |
| Red tint    | `#fde8e8` |
| Yellow tint | `#fef9e7` |

### Typography Sizes
| Element        | Size   | Weight |
|----------------|--------|--------|
| Title          | 18px   | 700    |
| Section title  | 15px   | 700    |
| Label          | 14px   | 600    |
| Sub-label      | 11px   | 400    |
| Note / caption | 11px   | 400    |
| Code text      | 10-11px| 400    |
| Badge          | 10px   | 700    |

### Layout Rules
- `viewBox` width: 600-860px (adjust to content)
- `viewBox` height: varies by content — do NOT pad excessively
- **viewBox와 배경 rect의 width/height는 반드시 일치시킨다.** 불일치 시 모서리가 잘리거나(viewBox < rect), 한쪽에 빈 공간이 생긴다(viewBox > rect).
- Corner radius: `rx="6"` for small boxes, `rx="8-12"` for large containers
- Stroke width: `2-2.5` for main borders, `1-1.5` for inner elements
- Use `stroke-dasharray="4,3"` for optional/placeholder elements or cables/connections
- Arrow markers defined in `<defs>` with appropriate colors
- Use `<filter id="s"><feDropShadow .../></filter>` for consistent shadow on key boxes

### SVG Template Structure
```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {W} {H}"
     font-family="'Inter', 'Segoe UI', -apple-system, sans-serif">
  <defs>
    <filter id="s"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-opacity="0.12"/></filter>
    <!-- Arrow markers, gradients -->
  </defs>

  <rect width="{W}" height="{H}" fill="#f8fafc" rx="12"/>
  <!-- Title -->
  <!-- Main content -->
</svg>
```

## Rules

### Content Principles
- **하나의 다이어그램에 하나의 개념만 담는다.** 여러 개념(예: 테이블 + 매핑 + 규칙 구조)을 하나의 정사각형 SVG에 욱여넣지 말고, 섹션별로 쪼개서 각각의 SVG로 만든다. 가로로 넓고 세로로 짧은 다이어그램이 블로그 가독성에 유리하다.
- **설명 박스(legend, caption, insight)를 다이어그램 하단에 넣지 않는다.** 색상 범례, 요약 문장, "Key:" 박스 등은 불필요하다. 본문 텍스트에서 설명하는 것이 훨씬 낫다.
- **All text in diagrams MUST be in English.** No Korean, Japanese, or other languages.
- Keep diagrams clean and readable — avoid clutter. 그림은 시각적 구조만 전달하고, 세부 설명은 본문에 맡긴다.

### Sizing
- **요소를 제거하면 viewBox와 배경 rect 높이도 함께 줄인다.** 하단에 빈 공간이 남지 않도록 한다.
- 내부 콘텐츠의 실제 영역에 맞춰 viewBox를 설정한다. 콘텐츠 최하단 + 20~30px 여백 정도가 적절하다.

### Technical
- File names should be lowercase kebab-case (e.g., `multi-stage-build-flow.svg`).
- Verify the SVG is well-formed XML before saving.
- Do NOT use external resources (images, fonts) — everything must be inline.
- Prefer CSS classes in `<style>` over inline styles for repeated elements.
- 민감한 정보(실제 MAC 주소, IP 주소 등)는 마스킹한다 (`aa:bb:cc:dd:ee:01`, `192.168.0.x` 등).
