# Claude Code로 기술 블로그 운영하기 — 가이드

이 문서는 Claude Code를 활용하여 Hugo 기술 블로그를 운영하기 위한 가이드다.
최초 작성 시점(2026-03)에는 "무엇을 만들까"에 대한 제안서였고, 지금은 `.claude/`에 구현된 구성을 설명하는 현황 문서다. (최종 갱신: 2026-08-22)

---

## 1. 블로그 구조 요약

- **프레임워크:** Hugo extended v0.157.0+
- **테마:** gandanham (커스텀 테마) + 루트 `layouts/` 오버라이드
- **언어:** 한국어(ko), 영어(en), 일본어(ja) — 3개 국어. `hugo.yaml`의 `languages`와 `i18n/{en,ko,ja}.yaml`에 모두 설정되어 있다
- **포스트 구조:** page bundle — `content/posts/{slug}/`
  - `index.ko.md` (한국어, 먼저 작성하는 원본)
  - `index.md` (영어 번역)
  - `index.ja.md` (일본어 번역)
  - `images/` (포스트 전용 이미지·SVG)
- **Front matter 필수 필드:** title, date, draft, tags, translationKey, summary
- **내부 링크:** `[text]({{< ref "/posts/slug" >}})` 숏코드. 평문 `](/posts/slug/)` 경로는 쓰지 않는다

---

## 2. 구성된 Skills & Sub-agents

### 2.1. Skills (`.claude/skills/`) — `/이름`으로 직접 호출

| 스킬 | 단계 | 역할 |
|------|------|------|
| `new-post` | 아웃라인 | 디렉터리·front matter·한국어 아웃라인 스캐폴딩 (3개 언어 파일 생성) |
| `tech-review` | 리뷰 | 기술적 정확성, deprecated API, 참조 유효성 |
| `content-enhance` | 리뷰 | 스토리텔링·가독성 보강 제안 |
| `seo-optimize` | 리뷰 | 메타데이터 감사 (이 사이트의 실제 OG 템플릿 기준) |
| `visual-guide` | 리뷰 | 시각 자료 제안 — SVG/표/ASCII (Mermaid는 렌더링되지 않음) |
| `svg-diagram` | 리뷰 | 발행 가능한 SVG 다이어그램 생성 |
| `translate-ko-to-en` | 발행 준비 | 한→영 번역 |
| `translate-ko-to-ja` | 발행 준비 | 한→일 번역 |
| `draft-manager` | 발행 준비 | 초안 현황 대시보드 / 발행 전 체크리스트 |
| `link-checker` | 유지보수 | 링크·이미지·앵커 검증 (`context: fork`로 서브에이전트에서 실행) |
| `series-manager` | 유지보수 | 시리즈 그룹핑 및 내비게이션 블록 생성 |
| `writing-conventions` | 참조 | 블로그 문체·용어·구조 규약의 단일 출처 (모델 전용, 직접 호출 불가) |

파일을 쓰는 스킬(`new-post`, `svg-diagram`, `translate-*`)은 `disable-model-invocation: true`가 걸려 있어 Claude가 임의로 실행하지 않는다. 직접 `/스킬명`으로 호출해야 한다.

### 2.2. Sub-agents (`.claude/agents/`) — 위임 실행

| 에이전트 | 모델 | 역할 |
|----------|------|------|
| `topic-researcher` | sonnet | 주제 사전 조사, 기존 글과의 중복 확인 |
| `outline-expander` | opus | 스캐폴딩 아웃라인을 절별 집필 계획으로 확장 |
| `code-example-gen` | sonnet | 한국어 주석이 달린 실행 가능한 코드 예제 |
| `style-checker` | sonnet | 문체·용어·포맷 일관성 검사 (규약 위반 판정) |
| `cross-linker` | sonnet | 포스트 간 내부 링크 추천 |
| `publish-helper` | inherit | draft 해제 및 공유 문구 생성 |

`style-checker`, `outline-expander`, `code-example-gen`, `publish-helper`는 `skills: writing-conventions`로 문체 규약을 컨텍스트에 미리 적재한다.

---

## 3. 블로그 작성 워크플로우

### 3.1. 새 글 작성 시 권장 순서

```
1.  topic-researcher    : 주제 조사, 기존 글 중복 확인
2.  /new-post           : 스캐폴딩 생성 (ko/en/ja + 아웃라인)
3.  outline-expander    : 절별 집필 계획 확장
4.  (직접 작성)          : 한국어로 초안 작성 — 평어체
5.  code-example-gen    : 코드 예제 보강
6.  /tech-review        : 기술적 정확성 검토
7.  /content-enhance    : 스토리텔링·가독성 보강
8.  style-checker       : 문체·용어·포맷 규약 검사
9.  /visual-guide       : 시각 자료 계획
10. /svg-diagram        : 다이어그램 생성
11. /seo-optimize       : 메타데이터 감사
12. cross-linker        : 내부 링크 추천
13. /translate-ko-to-en, /translate-ko-to-ja
14. /draft-manager {slug} : 발행 전 체크리스트
15. hugo server         : 로컬 프리뷰
16. publish-helper      : draft 해제 + 공유 문구
```

### 3.2. 기존 글 관리

```
/draft-manager      : 미발행 글·번역 누락 현황 파악
/link-checker all   : 전체 글 링크 검증
/series-manager     : 시리즈 구성 및 연결
cross-linker        : 새 글 발행 후 기존 글에서 걸어줄 링크 찾기
```

---

## 4. Hugo 운영 팁

### 4.1. 로컬 프리뷰
```bash
hugo server -D    # draft 포함하여 로컬 서버 실행
hugo server       # 발행 상태 글만 프리뷰
hugo --minify     # 프로덕션 빌드 (public/)
```

### 4.2. 언어 구성

3개 국어(en/ko/ja)는 이미 구성되어 있다. `defaultContentLanguage: en`, `defaultContentLanguageInSubdir: false`이므로 영어가 루트 경로(`/posts/slug/`)를 쓰고 한국어·일본어는 `/ko/`, `/ja/` 하위로 간다.

언어를 추가할 때 손봐야 하는 곳:
- `hugo.yaml`의 `languages.<코드>` (languageName, weight, menus)
- `i18n/<코드>.yaml` (UI 문자열)
- `layouts/_partials/head/og.html`의 로케일 매핑 (`en_US` / `ko_KR` / `ja_JP`)

### 4.3. Front Matter 템플릿
```yaml
---
title: "포스트 제목"
date: 2026-08-22
draft: true
tags: ["tag1", "tag2"]
translationKey: "post-slug"
summary: "이 포스트에서 다루는 핵심 내용 요약 (평어체 한 줄)"
---
```

`translationKey`는 3개 언어 파일에서 동일해야 하고, 디렉터리 슬러그와 일치시킨다.

### 4.4. 이미지 관리
- 포스트별 이미지는 page bundle 안의 `images/`에 저장: `content/posts/{slug}/images/name.svg`
- 마크다운에서 참조: `![alt text](images/name.svg)`
- 공통 이미지는 `static/images/`에 저장하고 `/images/name.png`으로 참조
- 다이어그램 안의 텍스트는 한국어 글에서도 영어로 쓴다
- 소셜 카드 이미지는 front matter의 `images: ["images/name.png"]`로 포스트별 지정 가능 (미지정 시 `/images/og-default.jpg` 폴백)

### 4.5. 다크모드
`assets/css/main.css`에 `[data-theme="dark"]` 토글이 있고, `.post-content img`에는 별도 처리가 없다. SVG는 `<img>`로 로드되어 페이지 테마를 알 수 없으므로, 다이어그램은 밝은 배경 카드(`#f8fafc`, `rx="12"`)를 유지하는 것을 전제로 만든다.

---

## 5. 콘텐츠 품질 체크리스트

### 기술적 정확성
- [ ] 코드 예제가 실제로 동작하는가?
- [ ] 기술 용어를 정확하게 사용했는가?
- [ ] 버전/날짜에 민감한 정보가 최신인가?
- [ ] 참고 자료/출처를 명시했는가?

### 문체 규약
- [ ] 본문이 평어체(~이다/~한다)인가?
- [ ] `**한글용어**(English)` — 괄호가 볼드 바깥에 있는가?
- [ ] em dash가 `—`인가? (`--` 아님)
- [ ] 각 장이 이전 장과 연결되는 도입 1~2문장으로 시작하는가?
- [ ] `## 마치며`가 요약 → 개인적 통찰 → 향후 방향 순인가?

### 가독성
- [ ] 도입부가 독자의 관심을 끄는가?
- [ ] 섹션 간 논리적 흐름이 있는가?
- [ ] 적절한 시각 자료(다이어그램, 표, 코드)가 포함되어 있는가?
- [ ] 결론에서 핵심 메시지가 명확한가?

### SEO & 메타데이터
- [ ] title이 명확하고 검색 키워드를 포함하는가?
- [ ] summary가 채워져 있는가? 이 사이트는 summary를 meta description으로 쓰고 200자에서 자른다
- [ ] tags가 3-5개이며 소문자-하이픈 영문인가?
- [ ] translationKey가 3개 언어에서 동일한가?

### 다국어
- [ ] 한국어 원본이 완성되어 있는가?
- [ ] 영어 번역이 자연스러운가? (직역 아닌 의역)
- [ ] 일본어 번역이 です/ます体인가?
- [ ] 모든 언어 버전의 front matter가 일관되는가?
- [ ] 한국어만 발행되고 번역이 draft로 남아 있지 않은가?

---

## 6. 기존 포스트 현황 참고

현재 26개 포스트(한국어 26 / 영어 26 / 일본어 11), 주요 카테고리:

- **DevOps/GitOps:** about-gitops, gitops-secret-management, kustomize-multi-environment, jenkins-cicd-pipeline, docker-complete-guide
- **Kubernetes/Service Mesh:** kubernetes-cni-networking, kubernetes-networking-ingress-to-istio, istio-deep-dive, raspberry-pi-k8s-homeserver
- **Observability/SRE:** prometheus-grafana-monitoring, slo-sli-sla-error-budget
- **Cloud/AWS:** aws-cost-optimization, aws-production-deployment, aws-serverless-chat-server, aws-cli-credentials-identity-center
- **네트워크:** network-fundamentals-ssh-https-rest, linux-network-internals, tailscale-mtu-troubleshooting
- **Backend:** java-spring-multithreading, javascript-nodejs-internals, redis-deep-dive
- **보안/인증:** auth-concepts-to-keycloak
- **AI/ML:** embeddings-vector-search
- **기타:** computer-architecture-story, stable-marriage-matching-algorithm, web-crawler-scraping-guide
