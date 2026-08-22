# Istio 딥다이브 — Envoy, xDS, mTLS, 그리고 커널까지

## 목차

1. [전체 아키텍처: Control Plane + Data Plane](#1-전체-아키텍처-control-plane--data-plane)
2. [사이드카 주입 메커니즘: Mutating Admission Webhook](#2-사이드카-주입-메커니즘-mutating-admission-webhook)
3. [트래픽 인터셉트의 핵심: iptables 리다이렉트](#3-트래픽-인터셉트의-핵심-iptables-리다이렉트)
4. [실제 요청 흐름 (Pod A → Pod B)](#4-실제-요청-흐름-pod-a--pod-b)
5. [Pilot, Citadel, Galley → istiod 통합](#5-pilot-citadel-galley--istiod-통합)
6. [Envoy와 Istio의 관계](#6-envoy와-istio의-관계)
7. [xDS 프로토콜 딥다이브](#7-xds-프로토콜-딥다이브)
8. [istiod의 배치와 동작](#8-istiod의-배치와-동작)
9. [mTLS와 Zero Trust](#9-mtls와-zero-trust)
10. [Istio + CNI 조합 (Cilium, Calico, Flannel)](#10-istio--cni-조합-cilium-calico-flannel)
11. [Istio Ambient Mesh — 사이드카 없는 미래](#11-istio-ambient-mesh--사이드카-없는-미래)
12. [전체 그림 정리](#12-전체-그림-정리)

---

## 1. 전체 아키텍처: Control Plane + Data Plane

Istio는 크게 두 층으로 나뉜다.

### Control Plane (istiod)

중앙 관리자. Pilot, Citadel, Galley가 하나로 합쳐진 컴포넌트로, Envoy 프록시들에게 설정을 내려보내는 역할을 한다. Kubernetes API를 watch하면서 Service, Endpoint, VirtualService 같은 리소스 변화를 감지하고, 이를 Envoy가 이해할 수 있는 xDS(discovery service) 설정으로 변환해서 각 사이드카에 push한다.

### Data Plane (Envoy proxies)

실제 트래픽이 흐르는 곳. 모든 Pod에 사이드카로 주입된 Envoy가 트래픽을 가로채서 처리한다.

---

## 2. 사이드카 주입 메커니즘: Mutating Admission Webhook

### Kubernetes API Server의 요청 처리 파이프라인

`kubectl apply`로 Pod을 생성하면, 그 요청이 API Server에 도달한 뒤 **저장되기 전에** 여러 단계를 거친다:

```
kubectl apply (Pod 생성 요청)
       │
       ▼
  Authentication (인증: 너 누구야?)
       │
       ▼
  Authorization (인가: 너 이 작업 할 권한 있어?)
       │
       ▼
  Mutating Admission Webhooks  ← ★ 여기서 Pod spec을 "변형"할 수 있음
       │
       ▼
  Schema Validation (스키마 검증)
       │
       ▼
  Validating Admission Webhooks  ← 변형은 못 하고, 거부만 가능
       │
       ▼
  etcd에 저장 → Pod 생성 진행
```

**Mutating Admission Webhook**은 이 파이프라인 중간에 끼어들어서 "요청 내용을 수정(mutate)할 수 있는 외부 HTTP 엔드포인트"다. API Server가 Pod 생성 요청을 받으면, 등록된 webhook 서버에 그 요청을 보내고, webhook 서버가 "이 Pod spec에 이런 컨테이너를 추가해"라는 JSON Patch를 응답으로 돌려보내는 구조다.

### Istio의 경우 구체적으로

Istio를 설치하면 istiod가 자기 자신을 Mutating Admission Webhook으로 API Server에 등록한다. 이때 **"네임스페이스에 `istio-injection=enabled` 라벨이 있는 경우에만 호출해줘"**라는 조건을 같이 건다.

```yaml
# MutatingWebhookConfiguration (간략화)
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
webhooks:
- name: sidecar-injector.istio.io
  clientConfig:
    service:
      name: istiod          # webhook 서버 = istiod
      namespace: istio-system
      path: /inject          # 이 경로로 요청이 감
  namespaceSelector:
    matchLabels:
      istio-injection: enabled   # 이 라벨이 있는 네임스페이스만
  rules:
  - operations: ["CREATE"]
    resources: ["pods"]          # Pod 생성 시에만 동작
```

### 주입 흐름

1. `istio-injection=enabled` 라벨이 붙은 네임스페이스에서 Pod 생성 요청이 들어옴
2. API Server가 "이 네임스페이스에 매칭되는 webhook이 있네" 하고 istiod의 `/inject` 엔드포인트에 Pod spec을 보냄
3. istiod가 Pod spec을 받아서 두 개의 컨테이너를 추가하는 JSON Patch를 응답:
   - **istio-init** (init container) — 기동 시 iptables 규칙을 설정하는 역할
   - **istio-proxy** (sidecar container) — Envoy 프록시 자체
4. API Server가 그 patch를 적용해서 수정된 Pod spec을 etcd에 저장
5. kubelet이 수정된 spec대로 Pod을 띄움 → 사이드카가 같이 뜸

**핵심: 앱 개발자가 Deployment yaml에 Envoy 관련 설정을 전혀 안 써도, webhook이 알아서 끼워넣는다.**

### Prometheus 어노테이션과의 비교

- **Prometheus 어노테이션** (`prometheus.io/scrape: "true"`) → "나 여기 있으니 수집하러 와" (발견의 힌트, Pod 자체는 안 변함)
- **Istio Mutating Webhook** → "이 Pod이 생성될 때 내가 컨테이너를 끼워넣을게" (Pod spec 자체가 변형됨)

공통점: 둘 다 **라벨/어노테이션이라는 메타데이터를 기반으로 자동화된 동작이 트리거된다**는 점. 자동화의 메커니즘이 다를 뿐이다.

---

## 3. 트래픽 인터셉트의 핵심: iptables 리다이렉트

istio-init 컨테이너가 설정하는 iptables 규칙의 핵심 로직:

```bash
# 1. ISTIO_REDIRECT 체인 생성 — 모든 트래픽을 Envoy의 15001 포트로 보냄
iptables -t nat -N ISTIO_REDIRECT
iptables -t nat -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001

# 2. ISTIO_IN_REDIRECT — 인바운드 트래픽을 Envoy의 15006 포트로
iptables -t nat -N ISTIO_IN_REDIRECT
iptables -t nat -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006

# 3. OUTPUT 체인 — Pod에서 나가는 트래픽 가로채기
iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT

# 4. PREROUTING 체인 — Pod으로 들어오는 트래픽 가로채기
iptables -t nat -A PREROUTING -p tcp -j ISTIO_INBOUND
```

netfilter의 5개 훅 포인트 중에서 **PREROUTING**과 **OUTPUT**이 사용된다. nat 테이블의 REDIRECT 타겟을 써서, 목적지 포트를 Envoy가 리스닝하는 포트로 바꿔버리는 구조다.

### 무한루프 방지

Envoy 자신이 보내는 트래픽이 다시 인터셉트되면 무한루프에 빠진다. 따라서 **Envoy 프로세스의 UID(보통 1337)**로 나가는 트래픽은 iptables 규칙에서 제외한다. `--uid-owner 1337`로 매칭해서 RETURN시킨다.

### SO_ORIGINAL_DST와 conntrack의 관계

Envoy가 트래픽을 가로챌 때 `SO_ORIGINAL_DST` 소켓옵션으로 원래 목적지를 복원하는데, 이는 netfilter의 **conntrack**(connection tracking)이 REDIRECT 전의 원래 목적지 정보를 기억하고 있기 때문에 가능하다. Istio의 트래픽 인터셉트는 "netfilter + conntrack + iptables nat 테이블"이라는 리눅스 커널 네트워크 스택 위에 완전히 의존하는 구조다.

---

## 4. 실제 요청 흐름 (Pod A → Pod B)

```
[Pod A: App Container]
     │
     │ ① 앱이 Pod B의 Service IP:Port로 TCP connect()
     │
     ▼
[Pod A: netfilter OUTPUT hook]
     │
     │ ② nat 테이블에서 ISTIO_OUTPUT 체인 매칭
     │    → 목적지를 127.0.0.1:15001 (Envoy outbound)로 REDIRECT
     │
     ▼
[Pod A: Envoy Sidecar (15001)]
     │
     │ ③ Envoy가 원래 목적지를 SO_ORIGINAL_DST 소켓옵션으로 복원
     │    → xDS 설정에 따라 라우팅 결정 (load balancing, retry, timeout 등)
     │    → mTLS 핸드셰이크 (istiod가 발급한 인증서 사용)
     │    → Pod B의 실제 IP로 새 연결 생성
     │
     ▼
  ── 네트워크 (CNI: Flannel/Calico/Cilium) ──
     │
     ▼
[Pod B: netfilter PREROUTING hook]
     │
     │ ④ nat 테이블에서 ISTIO_INBOUND 체인 매칭
     │    → 목적지를 127.0.0.1:15006 (Envoy inbound)으로 REDIRECT
     │
     ▼
[Pod B: Envoy Sidecar (15006)]
     │
     │ ⑤ Envoy가 mTLS 검증, 인가 정책 체크, 메트릭 수집
     │    → 원래 목적지 포트(앱 포트)로 localhost 연결
     │
     ▼
[Pod B: App Container]
     │
     │ ⑥ 앱이 요청을 받음 — 앱 입장에서는 직접 받은 것처럼 보임
```

**핵심: 앱은 상대 앱과 직접 통신한다고 생각하지만, 실제로는 양쪽의 Envoy끼리 통신하는 것이다.** 이 구조 덕분에 앱 코드를 전혀 수정하지 않고도 Envoy끼리 mTLS 핸드셰이크를 수행할 수 있다.

### Envoy가 제공하는 기능들

이 사이드카 구조 덕분에 앱 코드 수정 없이:

- **트래픽 관리** — VirtualService, DestinationRule로 가중치 기반 라우팅, 카나리 배포, 서킷 브레이커, 리트라이, 타임아웃을 선언적으로 설정
- **보안** — Pod 간 통신이 자동으로 mTLS로 암호화. PeerAuthentication, AuthorizationPolicy로 L7 레벨(HTTP path, method까지) 접근 제어
- **관측성(Observability)** — 모든 요청의 지연시간, 성공률, 처리량을 자동 수집하여 Prometheus 메트릭으로 노출, 분산 트레이싱 헤더 전파, 액세스 로그

---

## 5. Pilot, Citadel, Galley → istiod 통합

Istio 초기 버전(1.4 이전)에는 Control Plane이 별도의 마이크로서비스 3개로 분리돼 있었다.

### Pilot — 트래픽 관리의 두뇌

**"Envoy에게 트래픽을 어떻게 라우팅할지 알려주는 역할."**

Kubernetes의 Service, Endpoint, 그리고 Istio의 VirtualService, DestinationRule 같은 리소스를 watch하면서, 이 정보를 Envoy가 이해하는 **xDS API**(LDS, RDS, CDS, EDS)로 변환해서 각 사이드카에 gRPC 스트림으로 내려보냄. Envoy가 "이 요청은 어디로 보내야 해?", "리트라이는 몇 번?", "타임아웃은?" 같은 걸 아는 건 전부 Pilot이 내려보낸 설정 덕분이다.

### Citadel — 보안 담당 (인증서 관리)

**"서비스 간 mTLS에 쓰이는 인증서를 발급하고 갱신하는 CA(Certificate Authority)."**

각 Envoy 사이드카는 기동할 때 Citadel에게 자기 서비스 ID에 해당하는 X.509 인증서를 요청하고, Citadel이 서명해서 내려준다. 이 인증서가 있어야 Pod A의 Envoy와 Pod B의 Envoy가 서로 mTLS 핸드셰이크를 할 수 있다. 인증서 만료 전 자동 갱신도 Citadel이 담당했다.

### Galley — 설정 검증 및 배포

**"사용자가 작성한 Istio 설정(VirtualService, DestinationRule 등)을 검증하고, 정규화해서 다른 컴포넌트에 전달하는 중간 계층."**

설정이 올바른지 체크하고, 내부 포맷으로 변환하는 전처리기 역할이었다. 다른 컴포넌트가 Kubernetes API를 직접 찔러야 하는 부분을 Galley이 추상화해주려 했다.

### 왜 합쳤나 → istiod

이 3개가 따로 돌아가면서 생긴 문제들:

- **운영 복잡도** — 3개의 Deployment를 각각 배포, 모니터링, 스케일링해야 했음
- **컴포넌트 간 통신** — 서로 네트워크로 통신하다 보니 장애 포인트가 늘어남
- **리소스 오버헤드** — 각각이 별도 프로세스로 메모리와 CPU를 잡아먹음
- **Galley의 존재 의의** — 실제로는 Pilot이 Kubernetes API를 직접 watch하는 게 더 효율적이라, Galley의 추상화 계층이 오히려 복잡도만 추가

Istio 1.5부터 이 세 기능을 **하나의 바이너리 `istiod`로 통합**:

```
istiod = Pilot(트래픽 설정 배포)
       + Citadel(인증서 발급/갱신)
       + Galley(설정 검증)
```

기능은 동일한데 하나의 프로세스에서 돌아가는 것. istiod의 "d"는 Unix 전통의 daemon 네이밍이다 (`sshd`, `httpd`처럼).

---

## 6. Envoy와 Istio의 관계

### Envoy는 독립 프로젝트다

**Envoy는 Istio의 일부가 아니다.** 원래 Lyft에서 만든 독립적인 오픈소스 프록시로, 현재는 CNCF graduated 프로젝트다. Istio가 나오기 전부터 존재했고, Istio 없이도 단독으로 쓸 수 있다.

Envoy 자체가 할 수 있는 것들:

- L7 프로토콜 인식 (HTTP/1.1, HTTP/2, gRPC, TCP, MongoDB, Redis 등)
- 로드밸런싱 (Round Robin, Least Request, Ring Hash 등)
- 서킷 브레이커, 리트라이, 타임아웃
- TLS termination / origination
- 관측성 (메트릭, 트레이싱, 액세스 로그)
- 동적 설정 변경 (재시작 없이 xDS API를 통해)

마지막 포인트가 핵심이다 — Envoy는 **설정을 외부에서 동적으로 주입받을 수 있도록 xDS라는 API 인터페이스를 제공**한다.

### Istio는 뭘 하나?

Envoy는 혼자서도 강력하지만, **수십~수백 개의 Envoy를 누가 관리하느냐**는 문제가 있다. Pod이 100개면 Envoy도 100개인데, 각각에게 설정 변경을 알려줘야 한다. Istio는 바로 이 **관리 문제를 해결하는 Control Plane**이다.

비유:

- **Envoy** = 각 교차로에 서 있는 교통경찰. 직접 차량(패킷)을 통제하는 능력이 있음.
- **Istio(istiod)** = 중앙 교통관제센터. 모든 교통경찰에게 실시간으로 지시를 내려보냄.

### istio-proxy = Envoy + α

`kubectl describe pod`에서 보이는 `istio-proxy` 컨테이너는 **Envoy 바이너리 + Istio가 추가한 부트스트랩 로직**이다.

```
istio-proxy 컨테이너 안에서 돌아가는 것:

1. pilot-agent (Istio가 만든 래퍼)
   ├── Envoy 프로세스를 시작하고 관리
   ├── istiod에서 초기 부트스트랩 설정을 가져옴
   ├── 인증서 갱신 처리
   ├── Envoy 헬스체크
   └── Envoy가 죽으면 재시작

2. Envoy (실제 프록시 엔진)
   ├── pilot-agent가 넘겨준 부트스트랩 설정으로 기동
   ├── istiod에 xDS gRPC 연결을 맺고 설정 수신
   └── 실제 트래픽 처리 (라우팅, mTLS, 메트릭 등)
```

**istio-proxy 컨테이너 안에서 실제 트래픽을 처리하는 엔진은 Envoy가 맞다. 다만 Istio가 pilot-agent라는 래퍼로 감싸서 라이프사이클 관리와 istiod 연동을 추가한 것이다.**

### xDS 표준 인터페이스의 의미

이 분리 덕분에:

- **Envoy는 Istio 외에도 여러 Control Plane과 붙을 수 있다.** AWS App Mesh, Consul Connect, Gloo Edge 같은 것들도 Envoy를 데이터 플레인으로 쓰면서 자기만의 Control Plane을 제공한다.
- **Istio도 Envoy 외의 프록시를 데이터 플레인으로 쓸 수 있다.** Ambient Mesh의 ztunnel이 이 케이스다 — Envoy가 아니라 Rust로 새로 작성한 L4 전용 프록시인데, istiod의 xDS를 통해 설정을 받는다.

```
istiod (Control Plane)
   │
   │ xDS API (표준 인터페이스)
   │
   ├──▶ Envoy (사이드카 / waypoint)  — L7 전체 기능
   │
   └──▶ ztunnel (Rust, 노드 레벨)    — L4 특화, 경량
```

---

## 7. xDS 프로토콜 딥다이브

### xDS 구성 요소

xDS는 "x Discovery Service"의 약자로, x에 여러 글자가 들어가면서 각각 다른 종류의 설정을 담당한다:

| xDS | 이름 | 역할 |
|-----|------|------|
| **LDS** | Listener Discovery Service | 어떤 포트에서 트래픽을 받을지 |
| **RDS** | Route Discovery Service | 받은 트래픽을 어떤 규칙으로 라우팅할지 |
| **CDS** | Cluster Discovery Service | 라우팅 대상(upstream 그룹)이 뭔지 |
| **EDS** | Endpoint Discovery Service | 각 upstream 그룹의 실제 Pod IP:Port가 뭔지 |
| **SDS** | Secret Discovery Service | mTLS에 쓸 인증서와 키 |

이 5개가 조합되면 Envoy가 트래픽을 처리하는 데 필요한 모든 정보가 완성된다.

### 구체적 예시: 카나리 배포

`reviews` 서비스로 가는 트래픽의 90%를 v1으로, 10%를 v2로 보내는 설정:

```yaml
# 사용자가 작성하는 Istio CRD
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-dest
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

istiod가 이를 xDS로 변환하면:

```
[LDS] Listener 설정
  "0.0.0.0:15001에서 아웃바운드 트래픽을 받아라"
  "Host 헤더가 'reviews'이면 → Route Config 'reviews-route'를 사용해라"
         │
         ▼
[RDS] Route 설정
  "reviews-route:"
  "  90% → Cluster 'reviews.default.svc.cluster.local|v1'"
  "  10% → Cluster 'reviews.default.svc.cluster.local|v2'"
         │
         ▼
[CDS] Cluster 설정
  "reviews|v1: 로드밸런싱=RoundRobin, 서킷브레이커=5xx 3회시 차단"
  "reviews|v2: 로드밸런싱=RoundRobin, 서킷브레이커=5xx 3회시 차단"
         │
         ▼
[EDS] Endpoint 설정
  "reviews|v1 의 실제 엔드포인트: [10.244.1.5:8080, 10.244.2.8:8080]"
  "reviews|v2 의 실제 엔드포인트: [10.244.3.2:8080]"
         │
         ▼
[SDS] Secret 설정
  "이 Envoy의 인증서: <X.509 cert>, 키: <private key>"
  "신뢰할 CA: <root cert>"
```

### gRPC 스트리밍으로 실시간 동기화

xDS는 **양방향 gRPC 스트림**을 사용한다. 핵심은 **Envoy가 istiod에게 연결을 먼저 맺는다**는 점이다:

```
시간 →

Envoy 기동
  │
  ├──[gRPC CONNECT]──────────────────────▶ istiod:15012
  │                                          │
  │◀──[LDS 응답: Listener 설정]──────────────┤
  │◀──[RDS 응답: Route 설정]─────────────────┤
  │◀──[CDS 응답: Cluster 설정]───────────────┤
  │◀──[EDS 응답: Endpoint 목록]──────────────┤
  │◀──[SDS 응답: 인증서]─────────────────────┤
  │                                          │
  │   ... 연결 유지 (long-lived stream) ...   │
  │                                          │
  │        ── Pod 하나가 새로 뜸 ──           │
  │                                          │
  │◀──[EDS 업데이트: Endpoint 추가]──────────┤
  │                                          │
  │        ── VirtualService 변경 ──          │
  │                                          │
  │◀──[RDS 업데이트: Route 변경]─────────────┤
```

모든 xDS를 하나의 gRPC 스트림으로 합쳐서 보내는 방식을 **ADS(Aggregated Discovery Service)**라고 한다. LDS/RDS/CDS/EDS가 따로 오면 순서가 꼬일 수 있기 때문에(예: Route가 참조하는 Cluster가 아직 안 옴), 하나의 스트림에서 순서를 보장하며 보낸다.

### istiod 내부 처리 흐름

```
[Kubernetes API Server]
     │
     │  Watch (Service, Endpoint, Pod, Istio CRDs)
     │
     ▼
[istiod: Config Controller]
     │
     │  "reviews Service의 Endpoint가 변경됐다"
     │  "VirtualService 'reviews-route'가 새로 생겼다"
     │
     ▼
[istiod: xDS Generator]
     │
     │  변경된 리소스 → 영향받는 Envoy들 계산
     │  → 해당 Envoy에 맞는 xDS 설정 생성
     │
     │  ★ 핵심: 모든 Envoy에 같은 설정을 보내는 게 아님
     │    각 Envoy의 위치(어떤 Pod, 어떤 네임스페이스)에 따라
     │    필요한 설정만 골라서 보냄
     │
     ▼
[istiod: xDS Server]
     │
     │  변경된 설정을 해당 Envoy들의 gRPC 스트림으로 Push
     │
     ▼
[각 Envoy] ── 설정 적용 (hot reload, 재시작 없음)
```

### VirtualService 변경 시 전체 흐름

```
① kubectl apply -f virtualservice.yaml
     │
     ▼
② K8s API Server가 VirtualService 리소스를 etcd에 저장
     │
     ▼
③ istiod가 watch하고 있다가 변경 이벤트 수신
     │
     ▼
④ istiod가 해당 VirtualService에 영향받는 Envoy들을 계산
     │
     ▼
⑤ 각 Envoy에 맞는 RDS(Route) 설정을 생성
     │
     ▼
⑥ 이미 열려있는 gRPC 스트림을 통해 해당 Envoy들에 Push
     │
     ▼
⑦ Envoy가 새 Route 설정을 hot reload로 즉시 적용
     │
     ▼
⑧ 다음 요청부터 새 라우팅 규칙이 적용됨
     (기존 연결은 영향 없음, 새 연결부터 적용)
```

이 전체 과정이 **재시작 없이, 수 초 이내에** 일어난다.

### 실제 확인 명령어

```bash
# 특정 Pod의 Envoy가 가진 Listener 설정 (LDS)
istioctl proxy-config listeners <pod-name>

# Route 설정 (RDS)
istioctl proxy-config routes <pod-name>

# Cluster 설정 (CDS)
istioctl proxy-config clusters <pod-name>

# Endpoint 설정 (EDS)
istioctl proxy-config endpoints <pod-name>

# 전체 설정을 JSON으로 덤프
istioctl proxy-config all <pod-name> -o json
```

이 명령들은 해당 Pod의 Envoy admin API(15000 포트)에 접속해서 현재 적용된 xDS 설정을 읽어온다.

---

## 8. istiod의 배치와 동작

### istiod는 워커 노드에 뜬다

istiod는 **일반 워커 노드에 뜨는 평범한 Deployment**다. 마스터 노드(K8s Control Plane 노드)에 뜨는 게 아니다. "Istio Control Plane"이라는 용어 때문에 헷갈릴 수 있지만, 이는 **Istio의 Control Plane**이지 **Kubernetes의 Control Plane(API Server, etcd, scheduler 등)**과는 별개다.

```
Kubernetes Cluster
├── Master Node (K8s Control Plane)
│   ├── kube-apiserver
│   ├── etcd
│   ├── kube-scheduler
│   └── kube-controller-manager
│
├── Worker Node 1
│   ├── kubelet
│   ├── istiod (Istio Control Plane) ← 여기에 스케줄링될 수 있음
│   ├── Pod A + istio-proxy
│   └── Pod B + istio-proxy
│
├── Worker Node 2
│   ├── kubelet
│   ├── Pod C + istio-proxy
│   └── Pod D + istio-proxy
```

```yaml
# istiod Deployment (간략화)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istiod
  namespace: istio-system
spec:
  replicas: 1          # 프로덕션에서는 2-3으로 HA 구성
  template:
    spec:
      containers:
      - name: discovery
        image: istio/pilot:1.24.0
        ports:
        - containerPort: 15010  # xDS gRPC (plaintext)
        - containerPort: 15012  # xDS gRPC (mTLS)
        - containerPort: 443    # Webhook (사이드카 주입)
        - containerPort: 15014  # 모니터링
```

Kubernetes scheduler가 리소스 상황에 따라 아무 워커 노드에나 배치한다. Service로 노출되니까 어떤 노드에 있든 모든 Envoy가 접근할 수 있다.

---

## 9. mTLS와 Zero Trust

### TLS vs mTLS

#### 일반 TLS (HTTPS)

브라우저로 `https://github.com`에 접속할 때 일어나는 일:

```
[브라우저]                              [github.com 서버]
    │                                        │
    ├── ClientHello ────────────────────────▶ │
    │   (지원하는 암호화 알고리즘 목록)         │
    │                                        │
    │ ◀──────────────────── ServerHello ──────┤
    │   (선택된 암호화 알고리즘)               │
    │                                        │
    │ ◀──────────────── Certificate ──────────┤
    │   (서버의 X.509 인증서)                  │
    │                                        │
    │   브라우저가 검증:                       │
    │   "이 인증서가 신뢰할 수 있는 CA가       │
    │    서명한 게 맞나?"                      │
    │   "인증서의 도메인이 github.com이 맞나?" │
    │                                        │
    │── Key Exchange ───────────────────────▶ │
    │   (세션 키 교환)                         │
    │                                        │
    │◀═══════ 암호화된 통신 시작 ═══════════▶ │
```

일반 TLS의 핵심 특징: **서버만 자기가 누구인지 증명한다.** 클라이언트는 인증서를 제출하지 않는다.

#### mTLS (Mutual TLS)

**클라이언트도 자기가 누구인지 인증서로 증명**한다:

```
[클라이언트]                              [서버]
    │                                       │
    ├── ClientHello ──────────────────────▶  │
    │                                       │
    │ ◀──────────────── ServerHello ─────────┤
    │ ◀──────────────── Certificate ─────────┤  ← 서버가 인증서 제출
    │ ◀──────────── CertificateRequest ──────┤  ← ★ "너도 인증서 내놔"
    │                                       │
    │── Certificate ──────────────────────▶  │  ← ★ 클라이언트도 인증서 제출
    │── Key Exchange ─────────────────────▶  │
    │                                       │
    │   서버가 검증:                          │
    │   "이 클라이언트 인증서가 신뢰할 수     │
    │    있는 CA가 서명한 게 맞나?"           │
    │                                       │
    │◀═══════ 암호화된 통신 시작 ═══════════▶│
```

**Mutual = 상호적.** 양쪽이 서로에게 "너 누구야?"를 물어보고, 양쪽 다 인증서로 답한다.

### 왜 Kubernetes에서 mTLS가 필요한가

일반 Kubernetes 클러스터에서 Pod 간 통신은 기본적으로 **평문**이다. 이게 문제가 되는 이유:

- 노드의 네트워크에 접근할 수 있는 누군가가 패킷을 캡처하면 내용이 다 보임
- Pod A가 Pod B에 요청을 보낼 때, Pod B는 "이게 정말 Pod A가 보낸 건지" 구별할 방법이 없음

이것이 **Zero Trust**의 핵심 전제다 — "네트워크 위치(같은 클러스터, 같은 VLAN 등)를 신뢰의 근거로 쓰지 않겠다." 대신 모든 통신에서 상대방의 **신원(identity)**을 암호학적으로 검증한다.

### Istio에서 mTLS가 동작하는 원리

#### Step 1: 워크로드 ID 체계 — SPIFFE

Istio는 각 워크로드에 **SPIFFE ID**(Secure Production Identity Framework for Everyone)라는 고유 신원을 부여한다:

```
spiffe://cluster.local/ns/default/sa/reviews
         ───────────── ────────── ──────────
          trust domain  namespace  service account
```

이 ID는 Kubernetes의 ServiceAccount에 매핑된다. Pod이 어떤 ServiceAccount로 실행되느냐에 따라 SPIFFE ID가 결정된다.

#### Step 2: 인증서 발급 과정

```
Envoy(istio-proxy) 기동
    │
    │ ① CSR(Certificate Signing Request) 생성
    │   "나는 spiffe://cluster.local/ns/default/sa/reviews 이야,
    │    이 공개키에 대한 인증서를 발급해줘"
    │
    │── SDS (Secret Discovery Service) 요청 ──▶ istiod
    │                                              │
    │                                    ② istiod가 검증:
    │                                    "이 Pod이 정말 해당
    │                                     ServiceAccount로
    │                                     실행 중인지 확인"
    │                                    (K8s API로 검증)
    │                                              │
    │                                    ③ 인증서 서명:
    │                                    istiod의 CA가 X.509
    │                                    인증서에 서명
    │                                    (SAN에 SPIFFE ID 포함)
    │                                              │
    │◀──── 서명된 인증서 + 개인키 + 루트 CA ────────┤
    │
    │ ④ Envoy가 인증서를 메모리에 로드
    │   (디스크에 저장하지 않음 — 더 안전)
    │
    │ ... 인증서 만료 전에 자동 갱신 (기본 24시간) ...
```

#### Step 3: 실제 mTLS 핸드셰이크

Pod A(productpage)가 Pod B(reviews)에 요청을 보낼 때:

```
[Pod A: App]
    │  GET http://reviews:8080/api  (평문 HTTP)
    │
    ▼
[Pod A: netfilter OUTPUT]
    │  iptables REDIRECT → Envoy 15001
    │
    ▼
[Pod A: Envoy]
    │
    │  ① Envoy가 목적지 확인: "reviews 서비스구나"
    │  ② xDS 설정 확인: "reviews는 mTLS STRICT 모드"
    │  ③ TLS 핸드셰이크 시작
    │
    ├──── ClientHello ──────────────────────────────▶ [Pod B: Envoy]
    │                                                    │
    │ ◀──── ServerHello + 서버 인증서 ───────────────────┤
    │       (SAN: spiffe://cluster.local/ns/default/sa/reviews)
    │                                                    │
    │ ◀──── CertificateRequest ──────────────────────────┤
    │       "너도 인증서 보여줘"                            │
    │                                                    │
    │  ④ Pod A Envoy가 Pod B 인증서 검증:                 │
    │     "istiod CA가 서명한 게 맞나?" ✓                  │
    │     "SPIFFE ID가 reviews ServiceAccount 맞나?" ✓    │
    │                                                    │
    ├──── 클라이언트 인증서 ──────────────────────────────▶│
    │     (SAN: spiffe://cluster.local/ns/default/sa/productpage)
    │                                                    │
    │                              ⑤ Pod B Envoy가 검증:  │
    │                              "CA 서명 맞나?" ✓       │
    │                              "SPIFFE ID 확인" ✓      │
    │                              "AuthorizationPolicy에  │
    │                               따라 이 ID가 접근      │
    │                               허용된 건가?" ✓         │
    │                                                    │
    │◀═══════════ 암호화된 채널 수립 ══════════════════▶  │
    │                                                    │
    │──── 암호화된 HTTP 요청 전송 ───────────────────────▶│
    │                                                    │
    │                              ⑥ Envoy가 복호화 후     │
    │                                 localhost → App:8080 │
    │                                 (평문 HTTP로 전달)    │
    │                                                    │
    │◀──── 암호화된 HTTP 응답 ───────────────────────────┤
    │                                                    │
    ▼
[Pod A: Envoy]
    │  복호화
    ▼
[Pod A: App]
    │  평문 HTTP 응답 수신
```

#### 앱이 전혀 모른다는 게 핵심

이 전체 과정에서 앱은:

- 평문 HTTP를 보냄 → Envoy가 암호화
- 평문 HTTP를 받음 → Envoy가 복호화
- 인증서를 관리할 필요 없음 → Envoy + istiod가 자동 처리
- 코드에 TLS 관련 로직이 전혀 없어도 됨

### mTLS + AuthorizationPolicy = Zero Trust

mTLS만으로는 "암호화 + 상호 인증"이다. 여기에 Istio의 AuthorizationPolicy를 더하면 **"누가 누구에게 접근할 수 있는지"**까지 제어할 수 있다:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/default/sa/productpage"  # 이 SPIFFE ID만
    to:
    - operation:
        methods: ["GET"]           # GET만 허용
        paths: ["/api/reviews/*"]  # 이 경로만 허용
```

네트워크 IP가 아니라 **암호학적으로 검증된 워크로드 신원**을 기준으로 판단하는 것이 Zero Trust의 핵심이다.

```
전통적 네트워크 보안:
  "10.244.1.0/24 대역에서 온 트래픽은 허용" ← IP 기반, 스푸핑 가능

Zero Trust (Istio mTLS):
  "spiffe://cluster.local/ns/default/sa/productpage 가
   서명된 인증서로 자신을 증명한 경우에만 허용" ← 암호학적 검증
```

### mTLS 모드

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system   # 메시 전체 적용
spec:
  mtls:
    mode: STRICT    # or PERMISSIVE
```

- **PERMISSIVE** — mTLS와 평문 둘 다 받아들임. 메시에 아직 포함 안 된 서비스가 있을 때 사용. Istio를 점진적으로 도입할 때 유용.
- **STRICT** — mTLS만 허용. 인증서 없이 오는 평문 트래픽은 거부. 완전한 Zero Trust.

---

## 10. Istio + CNI 조합 (Cilium, Calico, Flannel)

### CNI vs Service Mesh: 레이어가 다르다

```
┌─────────────────────────────────────────────────┐
│            Service Mesh (Istio)                  │
│    L7: HTTP 라우팅, mTLS, 서킷브레이커,         │
│        리트라이, 분산 트레이싱, L7 인가정책       │
├─────────────────────────────────────────────────┤
│              CNI (Cilium / Calico / Flannel)     │
│    L3/L4: Pod IP 할당, 노드 간 라우팅,          │
│           NetworkPolicy, 패킷 포워딩             │
└─────────────────────────────────────────────────┘
```

**CNI**는 "Pod이 IP를 받고 다른 Pod과 통신할 수 있게 해주는 기본 네트워크 인프라"이고, **Service Mesh**는 "그 위에서 애플리케이션 레벨의 트래픽을 정교하게 제어하는 계층"이다.

### Flannel + Istio

가장 단순한 조합. Flannel은 L3 오버레이(VXLAN)만 하고, NetworkPolicy도 지원 안 하니까 Istio와 충돌할 일이 거의 없다. 하지만 Flannel이 해주는 게 적어서, 네트워크 정책은 전부 Istio의 AuthorizationPolicy에 의존하게 된다. k3s의 기본 CNI가 Flannel이라 가장 쉽게 시작할 수 있는 조합.

### Calico + Istio

Calico가 L3/L4 NetworkPolicy를 담당하고, Istio가 L7 정책을 담당하는 **계층적 보안** 구조를 만들 수 있다. Calico와 Istio Ambient Mesh를 결합하면 ztunnel이 모든 트래픽을 암호화하고 ID를 검증하며, Calico가 CNI 레벨에서 어떤 연결이 허용되는지를 제어하는 심층 방어 전략을 구현할 수 있다.

### Cilium + Istio

Cilium이 커널 레벨 네트워킹(IP 관리, 라우팅, L3/L4 정책)을 처리하고, Istio가 애플리케이션 계층(HTTP 라우팅, mTLS ID, 세밀한 인가 정책, 트래픽 쉐이핑)을 처리하는 것이 권장 접근법이다.

#### 핵심 설정: 소켓 레벨 로드밸런싱 충돌 방지

Cilium의 `kubeProxyReplacement` 기능을 켜면, Cilium이 eBPF로 소켓 레벨 로드밸런싱을 수행한다. `connect()` 시스템콜 시점에 eBPF가 커널 레벨에서 바로 목적지 IP를 바꿔버리면, netfilter를 거치기도 전에 목적지가 결정되어 **Istio가 iptables로 설정해둔 REDIRECT 규칙을 우회**한다.

해결책:

```yaml
# Cilium Helm values
socketLB:
  hostNamespaceOnly: true   # Pod 내부가 아닌, 호스트 네임스페이스에서만 소켓 LB 적용
```

추가 설정 사항:

- Istio CNI 플러그인과 같이 쓰려면 CNI chaining 설정 필요
- Istio가 이미 mTLS를 하고 있으면 Cilium의 암호화는 꺼서 이중 암호화 방지
- Cilium L7 정책과 Istio L7 정책을 동시에 쓰면 split-brain 문제 발생 가능 → 하나만 선택 권장

```yaml
# CNI chaining 설정
# Cilium ConfigMap
cni-chaining-mode: "generic-veth"
custom-cni-conf: "false"
enable-endpoint-routes: "true"

# 이중 암호화 방지
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --set encryption.enabled=false
```

### 2026년의 세 가지 접근법

#### 접근법 1: 전통적 사이드카 (Istio + Envoy per Pod)

```
Pod = [App Container] + [Envoy Sidecar]
      iptables로 트래픽 인터셉트
```

성숙하고 검증됨. 리소스 오버헤드와 운영 복잡도가 단점.

#### 접근법 2: Istio Ambient Mesh (사이드카 제거)

```
노드 레벨:  ztunnel (DaemonSet) — L4: mTLS, 기본 라우팅
네임스페이스 레벨:  waypoint proxy (선택적) — L7: HTTP 라우팅, 인가정책
```

#### 접근법 3: Cilium만으로 서비스 메시 대체

Cilium이 자체적으로 mTLS(SPIFFE 기반), L7 정책, 로드밸런싱, Hubble 관측성을 제공한다.

> Istio가 사이드카를 제거하는 방향이고, Cilium은 서비스 메시 자체를 제거하는 방향. 두 프로젝트는 다른 베팅을 하고 있다.

### 현실적인 조합 선택 기준

```
                       L7 기능 필요도
                    낮음 ◄─────────► 높음
                    │                  │
  Cilium만으로 충분  │    Cilium CNI    │  Cilium CNI
  (서비스 메시 불필요)│  + Istio Ambient │  + Istio Sidecar
                    │    (ztunnel만)   │  (전체 Envoy)
                    │                  │
```

참고: CNI를 운영 중인 클러스터에서 교체하려면 모든 노드를 drain해야 하고, Pod 하나하나, 노드 하나하나 롤링 재시작이 필요하다.

---

## 11. Istio Ambient Mesh — 사이드카 없는 미래

Istio Ambient Mode는 사이드카 프록시 없이 서비스 메시를 관리하는 새로운 방식이다. 네트워크 트래픽 처리를 L4와 L7 두 개의 레이어로 분리한다.

### 핵심 컴포넌트

#### ztunnel (Zero-Trust Tunnel)

- Rust로 작성된 경량 L4 노드 레벨 프록시
- 노드당 하나만 DaemonSet으로 배포
- mTLS, 워크로드 ID, 기본 트래픽 포워딩, 제로 트러스트 분할을 처리
- 공유 노드 레벨 컴포넌트임에도 단일 "노드 ID"를 쓰지 않음 — 각 워크로드에 대해 고유한 X.509 인증서를 발급받음

#### waypoint proxy (선택적)

- Envoy 기반 L7 프록시
- 네임스페이스 또는 서비스 단위로 배포
- HTTP 라우팅, L7 인가정책 등 고급 기능이 필요할 때만 배포

### 활성화 방법

```bash
# 네임스페이스에 라벨만 붙이면 됨
kubectl label namespace <namespace> istio.io/dataplane-mode=ambient
```

### 사이드카 모드와의 공존

사이드카 모드를 사용하는 Pod과 ambient 모드를 사용하는 Pod이 같은 메시 안에서 공존할 수 있다. 점진적 전환이 가능하다.

### Ambient Mode의 장점

- **리소스 절감** — 노드당 하나의 ztunnel이 많은 사이드카를 대체. CPU/메모리 소비 대폭 감소 (최대 70%)
- **운영 간소화** — 업그레이드 시 앱 Pod 재시작 불필요
- **점진적 도입** — L4 메시 기능은 전체 적용, L7 기능은 waypoint로 선택적 추가
- **보안 향상** — L7 프록시가 없는 순수 L4 모드에서는 CVE 노출 면이 줄어듦

### Cilium과 함께 쓸 때 주의사항

- Cilium이 kube-proxy 대체 기능을 켜면 소켓 레벨 로드밸런싱이 ztunnel보다 먼저 트래픽을 가로챌 수 있음
- Cilium NetworkPolicy는 L3/L4 규칙을 적용하고, Istio의 AuthorizationPolicy는 L7 규칙을 적용 → 정책이 정렬되지 않으면 한 레이어에서 허용되고 다른 레이어에서 거부될 수 있음
- Ambient Mode가 L4 흐름을 암호화하므로, Cilium은 mTLS 터널에 들어간 트래픽의 L7 내용을 검사할 수 없음

---

## 12. 전체 그림 정리

### 계층 구조

```
사용자가 작성하는 것:
  VirtualService, DestinationRule, AuthorizationPolicy (Istio CRDs)
          │
          ▼
istiod가 하는 것:
  K8s 리소스 + Istio CRD → xDS 설정으로 변환 → 각 Envoy에 gRPC로 푸시
  + 인증서 발급/갱신 (CA)
  + Mutating Webhook으로 사이드카 자동 주입
          │
          ▼
Envoy(istio-proxy)가 하는 것:
  xDS로 받은 설정에 따라 실제 트래픽 처리
  - 라우팅, 로드밸런싱, 리트라이
  - mTLS 핸드셰이크
  - 메트릭 수집, 트레이싱 헤더 전파
  - 인가 정책 적용
          │
          ▼
iptables(또는 eBPF)가 하는 것:
  앱 ↔ Envoy 사이의 투명한 트래픽 리다이렉트
  (앱은 자기가 직접 통신한다고 생각함)
```

### 의존 관계 요약

```
netfilter (커널)
  └── iptables 규칙으로 트래픽을 Envoy로 REDIRECT
        └── Envoy (istio-proxy)
              ├── xDS로 istiod에서 받은 라우팅 규칙 적용
              ├── SDS로 istiod에서 받은 인증서로 mTLS 수행
              ├── SPIFFE ID 기반 상호 인증
              └── AuthorizationPolicy로 L7 인가 결정
                    └── istiod (Control Plane)
                          ├── K8s API watch → xDS 설정 생성/배포
                          ├── CA로서 인증서 발급/갱신
                          └── Webhook으로 사이드카 자동 주입
```

**결론: "커널의 netfilter가 트래픽을 가로채고 → Envoy가 처리하고 → istiod가 관리한다"는 3계층 구조에서, mTLS는 Envoy 계층에서 수행되는 암호화/인증 메커니즘이고, 그 인증서 라이프사이클을 istiod가 자동으로 관리해주는 것이다.**