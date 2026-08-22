# Claude Sub-agent 추가 제안서

블로그 작성 워크플로우에서 기존 10개 스킬이 커버하지 못하는 영역을 분석하고, 추가 sub-agent를 제안합니다.

> **상태 (2026-08-22):** 이 문서는 제안서다. 여기서 제안한 6개 sub-agent는 모두 `.claude/agents/`에 구현되어 있고(`topic-researcher`, `outline-expander`, `code-example-gen`, `style-checker`, `cross-linker`, `publish-helper`), 스킬도 아래 표의 10개에서 12개로 늘었다(`svg-diagram`, `writing-conventions` 추가). 현재 구성은 `docs/claude_guide.md`를 본다.

---

## 1. 기존 스킬 현황 요약

| # | 스킬 | 워크플로우 단계 | 핵심 역할 |
|---|------|----------------|-----------|
| 1 | `tech-review` | 리뷰 | 기술적 정확성 검토 |
| 2 | `content-enhance` | 리뷰 | 스토리텔링/가독성 보강 제안 |
| 3 | `translate-ko-to-en` | 발행 준비 | 한→영 번역 |
| 4 | `translate-ko-to-ja` | 발행 준비 | 한→일 번역 |
| 5 | `new-post` | 아웃라인 | 디렉토리/front matter 스캐폴딩 |
| 6 | `seo-optimize` | 리뷰 | SEO 메타데이터 감사 |
| 7 | `visual-guide` | 리뷰 | 시각 자료 제안/생성 |
| 8 | `series-manager` | 유지보수 | 시리즈 네비게이션 |
| 9 | `draft-manager` | 발행 준비 | 초안 상태 대시보드/체크리스트 |
| 10 | `link-checker` | 유지보수 | 링크 유효성 검증 |

---

## 2. 워크플로우 갭 분석

```
리서치 → 아이디어 → 아웃라인 → 집필 → 리뷰 → 발행 → 유지보수
  X         X        △        X      O      △       △
```

### 잘 커버되는 단계
- **리뷰/최적화**: `tech-review`, `content-enhance`, `seo-optimize`, `visual-guide` — 완성된 글의 품질을 높이는 데 집중

### 부분적으로 커버되는 단계
- **아웃라인**: `new-post`가 디렉토리와 front matter를 생성하지만, 섹션별 내용 계획은 사용자 몫
- **발행**: `draft-manager`가 체크리스트를 제공하지만, 실제 발행 전환(draft→false, commit, 소셜 공유)은 수동
- **유지보수**: `link-checker`와 `series-manager`가 있지만, 내부 링크 전략이나 콘텐츠 최신화는 미커버

### 커버되지 않는 단계
- **리서치**: 주제에 대한 배경 조사, 레퍼런스 수집, 기존 포스트와의 차별점 분석
- **아이디어 발굴**: 트렌드 기반 주제 추천, 기존 포스트에서 파생 가능한 주제 탐색
- **집필 보조**: 아웃라인을 실제 초안으로 확장, 코드 예제 작성, 문체 일관성 유지

---

## 3. 추천 Sub-agent 상세

### 3.1. `topic-researcher` (주제 리서치) — 우선순위: 높음

**목적:** 새 포스트 주제에 대한 사전 리서치를 수행하고, 집필에 필요한 자료를 정리

**동작:**
1. 사용자가 주제 키워드를 입력 (예: "eBPF 기반 네트워크 모니터링")
2. 해당 주제의 핵심 개념, 주요 기술 요소, 최신 동향을 조사
3. 기존 18개 포스트와의 연관성 분석 — 겹치는 내용, 차별화 포인트 도출
4. 참고할 공식 문서, 블로그, GitHub 레포 등 레퍼런스 목록 생성
5. 리서치 결과를 구조화된 마크다운으로 출력

**사용 예시:**
```
/topic-researcher eBPF 네트워크 모니터링

→ 출력:
  - 핵심 개념 정리 (eBPF란, XDP, TC hook 등)
  - 기존 포스트 연관성: kubernetes-networking(네트워크 기초), prometheus-grafana(모니터링)
  - 차별화 포인트: 기존 포스트는 L7 중심, 이번 글은 L3/L4 패킷 레벨
  - 레퍼런스 5-10개
```

**기존 스킬과의 연계:**
- `new-post` 전에 실행 → 리서치 결과를 바탕으로 아웃라인 구성
- `tech-review`에 레퍼런스 목록 전달 → 검증 기준으로 활용

---

### 3.2. `outline-expander` (아웃라인 확장) — 우선순위: 높음

**목적:** `new-post`가 생성한 스켈레톤 아웃라인을 섹션별 상세 구성으로 확장

**동작:**
1. `new-post`로 생성된 `index.ko.md`의 아웃라인을 읽음
2. 각 섹션(H2/H3)에 대해:
   - 다룰 핵심 포인트 3-5개 나열
   - 포함할 코드 예제 유형 제안
   - 필요한 다이어그램/시각자료 명시
   - 예상 분량 (단어 수) 제시
3. `topic-researcher` 결과가 있으면 레퍼런스를 섹션별로 매핑
4. 확장된 아웃라인을 `index.ko.md`에 주석 형태로 삽입

**사용 예시:**
```
/outline-expander content/posts/ebpf-monitoring/index.ko.md

→ 기존:
  ## eBPF란?
  ## 네트워크 모니터링 아키텍처
  ## 실습

→ 확장:
  ## eBPF란?
  <!-- 핵심 포인트: 커널 내 가상머신, 안전성 보장(verifier), JIT 컴파일 -->
  <!-- 코드: 간단한 BPF 프로그램 C 코드 -->
  <!-- 시각자료: eBPF 실행 흐름 다이어그램 -->
  <!-- 예상 분량: 400-500자 -->
```

**기존 스킬과의 연계:**
- `new-post` → `outline-expander` → 직접 집필 → `tech-review` 순서로 사용
- `visual-guide`가 확장된 아웃라인의 시각자료 제안을 구체화

---

### 3.3. `code-example-gen` (코드 예제 생성) — 우선순위: 높음

**목적:** 블로그 포스트에 포함할 실행 가능한 코드 예제를 생성

**동작:**
1. 포스트의 주제와 컨텍스트를 분석
2. 요청된 유형의 코드 예제 생성:
   - Docker/Docker Compose 설정
   - Kubernetes YAML (Deployment, Service, Ingress 등)
   - Kustomize overlay 구조
   - Python/Java/JavaScript 코드 스니펫
   - Shell 스크립트 / CLI 명령어 시퀀스
3. 코드에 한국어 주석 추가
4. 단계별 설명 텍스트 생성 (코드 블록 사이에 넣을 설명)
5. 흔한 실수/주의사항을 "주의" 블록으로 생성

**사용 예시:**
```
/code-example-gen "Kustomize로 dev/staging/prod 환경별 설정 분리" --lang yaml

→ 출력:
  - base/kustomization.yaml + deployment.yaml
  - overlays/dev/kustomization.yaml (replicas: 1, 리소스 최소)
  - overlays/prod/kustomization.yaml (replicas: 3, HPA, PDB)
  - 각 파일별 설명 텍스트
  - 주의: "namespace를 overlay에서 설정하면 모든 리소스에 적용됨"
```

**기존 스킬과의 연계:**
- `outline-expander`가 "코드 예제 필요" 표시한 섹션에 대해 실행
- `tech-review`가 생성된 코드의 정확성 재검증

**적용 가능한 기존 포스트:**
- `kustomize-multi-environment`: Kustomize overlay 예제 보강
- `docker-complete-guide`: 멀티스테이지 빌드 예제 추가
- `jenkins-cicd-pipeline`: Jenkinsfile 예제 보강
- `aws-serverless-chat-server`: Lambda/API Gateway 코드 예제

---

### 3.4. `style-checker` (스타일 일관성 검사) — 우선순위: 중간

**목적:** 블로그 전체의 용어, 톤, 포맷팅 규칙 일관성 검사

**동작:**
1. 대상 포스트를 읽고 기존 포스트 샘플과 비교
2. 검사 항목:
   - **용어 일관성**: 같은 개념에 다른 용어 사용 여부 (예: "컨테이너" vs "Container")
   - **톤**: 경어체/평어체 혼용, 구어체/문어체 일관성
   - **포맷팅**: 코드 블록 언어 태그, 제목 계층(H2→H3→H4), 리스트 스타일
   - **Front matter**: tags 네이밍 규칙 (소문자, 하이픈 구분), summary 길이
3. 불일치 항목과 수정 제안 출력

**사용 예시:**
```
/style-checker content/posts/redis-deep-dive/index.ko.md

→ 출력:
  - [용어] L3: "레디스" → 다른 포스트에서는 "Redis"로 통일
  - [톤] L45: "~했다" (평어체) → 전체 톤은 "~합니다" (경어체)
  - [포맷] L78: ```code``` → 언어 태그 누락, ```bash```로 변경 권장
  - [태그] "Redis" → 기존 태그 규칙에 따라 "redis"로 소문자화
```

**기존 스킬과의 연계:**
- `content-enhance`가 내용 품질을 보면, `style-checker`는 형식 품질을 봄
- `translate-ko-to-en` 전에 실행 → 원문 일관성이 번역 품질에 영향

---

### 3.5. `cross-linker` (내부 링크 전략) — 우선순위: 중간

**목적:** 기존 포스트 간 내부 링크를 추천하고 관련글 네트워크를 구축

**동작:**
1. 대상 포스트의 주제와 키워드를 분석
2. 기존 18개 포스트와의 의미적 연관성 평가
3. 추천 항목:
   - 본문 중 자연스럽게 링크를 삽입할 위치와 대상 포스트
   - 포스트 하단 "관련 글" 섹션 내용
   - 역방향 링크: 기존 포스트에서 새 포스트로 링크할 위치
4. SEO 관점에서의 내부 링크 전략 제안

**사용 예시:**
```
/cross-linker content/posts/kubernetes-networking-ingress-to-istio/index.ko.md

→ 출력:
  삽입 추천:
  - L23 "컨테이너 네트워크" → [Docker 네트워크 완전 정복](/posts/docker-complete-guide/)
  - L67 "서비스 메시" → [Kustomize로 환경별 설정 관리](/posts/kustomize-multi-environment/)

  관련 글 섹션:
  - Docker 완전 가이드 (컨테이너 네트워크 기초)
  - Prometheus & Grafana (Istio 메트릭 수집)
  - 네트워크 기초 (SSH, HTTPS, REST)

  역방향 링크:
  - docker-complete-guide L142 근처에 K8s 네트워킹 링크 추가 권장
```

**기존 스킬과의 연계:**
- `series-manager`가 순차적 시리즈를 관리하면, `cross-linker`는 비순차적 연관 관계를 관리
- `seo-optimize`의 내부 링크 항목을 구체적으로 실행

**적용 가능한 연결 맵:**
```
docker ←→ kubernetes ←→ kustomize ←→ jenkins
  ↕                        ↕
network-fundamentals    prometheus-grafana
  ↕
kubernetes-networking
  ↕
aws-production-deployment ←→ aws-cost-optimization ←→ aws-serverless
```

---

### 3.6. `publish-helper` (발행 도우미) — 우선순위: 낮음

**목적:** 초안을 발행 상태로 전환하고 발행 후 작업을 자동화

**동작:**
1. `draft-manager`의 체크리스트가 모두 통과했는지 확인
2. 발행 전환:
   - `draft: true` → `draft: false` 변경
   - `date` 필드를 현재 날짜로 업데이트
   - 모든 언어 버전(ko, en, ja)에 동일 적용
3. Git 작업:
   - 변경 파일 스테이징
   - 커밋 메시지 생성 (예: `feat: publish post - kubernetes-networking`)
   - 커밋 (push는 사용자 확인 후)
4. 소셜 공유 텍스트 생성:
   - Twitter/X용 (280자 이내, 해시태그 포함)
   - LinkedIn용 (전문적 톤, 3-4문장)
   - 한국어/영어 버전 각각

**사용 예시:**
```
/publish-helper content/posts/ebpf-monitoring/

→ 체크리스트 확인: 모두 통과
→ draft: false 적용 (ko, en)
→ 커밋 메시지: "feat: publish post - eBPF 기반 네트워크 모니터링"
→ 소셜 공유:
  [Twitter/KO] eBPF로 커널 레벨 네트워크 모니터링을 구현하는 방법을 정리했습니다. XDP, TC hook부터 실전 예제까지. #eBPF #Kubernetes #DevOps
  [Twitter/EN] Deep dive into eBPF-based network monitoring — from XDP hooks to production setup. #eBPF #Kubernetes #DevOps
```

**기존 스킬과의 연계:**
- `draft-manager` → `publish-helper` 순서로 사용 (체크 → 실행)
- `translate-ko-to-en/ja` 완료 후 실행

---

## 4. 통합 워크플로우

기존 10개 스킬 + 추천 6개 sub-agent를 포함한 전체 워크플로우:

```
Phase 1: 기획
  topic-researcher ──→ new-post ──→ outline-expander
  (주제 리서치)       (스캐폴딩)     (아웃라인 상세화)

Phase 2: 집필
  (직접 작성) ←── code-example-gen
                  (코드 예제 보조)

Phase 3: 리뷰
  tech-review ──→ content-enhance ──→ style-checker ──→ seo-optimize
  (기술 검증)    (스토리텔링 보강)   (스타일 일관성)   (SEO 최적화)
       └──→ visual-guide
            (시각자료 보강)

Phase 4: 발행 준비
  translate-ko-to-en ──→ translate-ko-to-ja ──→ cross-linker
  (영어 번역)            (일본어 번역)           (내부 링크 연결)
       └──→ draft-manager ──→ publish-helper
            (발행 체크리스트)   (발행 + 공유)

Phase 5: 유지보수
  link-checker ──→ series-manager ──→ cross-linker
  (링크 검증)      (시리즈 관리)      (링크 네트워크 갱신)
```

### 워크플로우 커버리지 (개선 후)

```
리서치 → 아이디어 → 아웃라인 → 집필 → 리뷰 → 발행 → 유지보수
  O         O        O        O      O      O       O
```

| 단계 | 기존 | 추가 | 커버리지 |
|------|------|------|----------|
| 리서치 | - | `topic-researcher` | X → O |
| 아이디어 | - | `topic-researcher` (기존 포스트 기반 파생 주제) | X → O |
| 아웃라인 | `new-post` | `outline-expander` | △ → O |
| 집필 | - | `code-example-gen` | X → O |
| 리뷰 | `tech-review`, `content-enhance`, `seo-optimize`, `visual-guide` | `style-checker` | O → O |
| 발행 | `draft-manager`, `translate-*` | `publish-helper`, `cross-linker` | △ → O |
| 유지보수 | `link-checker`, `series-manager` | `cross-linker` | △ → O |

---

## 5. 구현 우선순위 로드맵

```
1차 (높음): topic-researcher, outline-expander, code-example-gen
  → 집필 전 단계의 생산성을 크게 높임
  → 현재 워크플로우에서 가장 큰 공백

2차 (중간): style-checker, cross-linker
  → 포스트 수가 늘어날수록 가치 증가
  → 20개 이상 포스트 시점에 특히 유용

3차 (낮음): publish-helper
  → 발행 빈도가 높아지면 도입
  → Git 작업과 소셜 공유의 반복 비용 절감
```
