# eBPF와 Cilium: 커널 네트워킹의 패러다임 전환

## 1. eBPF란 무엇인가

eBPF(extended Berkeley Packet Filter)는 **커널 소스를 수정하거나 커널 모듈을 로드하지 않고도, 커널 내부에서 샌드박스된 프로그램을 실행할 수 있게 해주는 기술**이다.

원래 BPF(Berkeley Packet Filter)는 1992년에 패킷 필터링 용도로 만들어졌다 — `tcpdump`가 대표적인 사용자다. 이것이 2014년쯤부터 리눅스 커널(3.15+)에서 대폭 확장되면서 "extended" BPF, 즉 eBPF가 되었다. 현재는 네트워킹뿐 아니라 보안, 관측성(observability), 트레이싱까지 커버하는 범용 커널 내 프로그래밍 프레임워크로 진화했다.

> 핵심 아이디어: **커널이 제공하는 특정 훅 포인트에 사용자가 작성한 프로그램을 안전하게 꽂아 넣는 것.**

---

## 2. 아키텍처: eBPF 프로그램이 실행되기까지

### 2.1 작성 (Write)

C 코드(또는 Rust)로 eBPF 프로그램을 작성한다. 단, 아무 C 코드나 되는 것이 아니라 루프 제한, 메모리 접근 제한 등 엄격한 제약이 있다.

### 2.2 컴파일 (Compile)

Clang/LLVM이 이 코드를 eBPF 바이트코드로 컴파일한다. 이것은 커널이 이해하는 전용 명령어 셋(ISA)이다 — x86나 ARM이 아니라 eBPF만의 가상 ISA.

### 2.3 검증 (Verify)

커널 내부의 **Verifier**가 프로그램을 정적 분석한다. 이것이 eBPF의 안전성 핵심이다:

- 무한 루프 불가 (프로그램 종료가 보장되어야 함)
- 범위 밖 메모리 접근 불가
- 초기화되지 않은 변수 사용 불가
- 허용된 helper function만 호출 가능
- 최대 명령어 수 제한 (커널 버전에 따라 다르지만 100만 개 수준)

Verifier를 통과하지 못하면 프로그램은 커널에 로드 자체가 되지 않는다.

### 2.4 JIT 컴파일 (Just-In-Time)

검증을 통과하면 eBPF 바이트코드가 **네이티브 머신 코드**(x86_64, ARM64 등)로 JIT 컴파일된다. 따라서 실행 성능이 커널 모듈과 거의 동등하다.

### 2.5 훅 포인트에 부착 (Attach)

컴파일된 프로그램이 커널의 특정 훅 포인트에 부착되어, 해당 이벤트가 발생할 때마다 실행된다.

---

## 3. eBPF의 핵심 구성 요소

### 3.1 Hook Points (프로그램 타입)

netfilter에는 5개의 훅(PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING)이 있다. eBPF는 netfilter보다 **훨씬 다양하고 더 이른 지점**에 훅을 걸 수 있다:

- **XDP (eXpress Data Path)**: NIC 드라이버 레벨, 패킷이 `sk_buff`로 변환되기도 **전에** 실행. 가장 빠른 지점이다.
- **TC (Traffic Control)**: ingress/egress의 qdisc 레이어. `sk_buff`에 접근 가능하며, netfilter보다 앞서거나 뒤에 위치할 수 있다.
- **Socket operations**: 소켓 레벨에서 연결 수립, 데이터 전송 등에 개입.
- **kprobes/uprobes**: 커널/유저스페이스 함수 진입/탈출 시 트레이싱.
- **tracepoints**: 커널에 미리 정의된 관측 지점.
- **cgroup**: cgroup 단위로 네트워크 정책 적용.

커널 네트워크 스택 흐름에 대입하면:

```
패킷 도착
  → NIC 드라이버 (여기서 XDP 실행!)
  → sk_buff 생성
  → TC ingress (여기서 TC eBPF 실행!)
  → netfilter PREROUTING
  → 라우팅 결정
  → netfilter FORWARD / INPUT
  → ...
```

XDP가 netfilter보다 **훨씬 앞단**에서 동작한다는 것이 핵심이다.

### 3.2 eBPF Maps

eBPF 프로그램은 기본적으로 상태를 가질 수 없다 (함수 실행처럼 들어갔다 나오는 구조이므로). 따라서 **Maps**라는 커널 내 자료구조를 통해 상태를 유지하고 데이터를 공유한다:

- Hash map, Array, LRU hash, Ring buffer, Per-CPU array 등 다양한 타입
- eBPF 프로그램 ↔ eBPF 프로그램 간 데이터 공유
- eBPF 프로그램 ↔ 유저스페이스 간 데이터 공유

예를 들어 Cilium은 "이 Pod의 IP가 어떤 Identity에 매핑되는가"를 Map에 저장한다.

### 3.3 Helper Functions

eBPF 프로그램이 커널 기능을 안전하게 사용할 수 있도록 커널이 제공하는 API:

- 패킷 데이터 읽기/쓰기
- Map 조회/업데이트
- 패킷 리다이렉트, 드롭
- 현재 시간 조회, 난수 생성
- 다른 eBPF 프로그램 tail-call 등

---

## 4. iptables의 구조적 한계

### 4.1 Service 생성 시 일어나는 일

쿠버네티스에서 Service를 하나 생성하면:

1. API Server가 Service 객체와 Endpoints(또는 EndpointSlice) 객체를 etcd에 저장
2. **모든 노드**의 kube-proxy가 이 변경을 watch로 감지
3. 각 노드의 kube-proxy가 자기 노드의 iptables 규칙을 업데이트

Service는 클러스터 전역 리소스이므로, **모든 노드의 kube-proxy가 각각 자기 노드의 iptables를 업데이트**해야 한다.

### 4.2 iptables 규칙의 실제 구조

Service 하나(backend Pod 3개)가 만드는 iptables 규칙의 예시:

```bash
# KUBE-SERVICES 체인 (모든 Service의 진입점)
-A KUBE-SERVICES -d 10.96.3.42/32 -p tcp --dport 80 -j KUBE-SVC-XXXX

# KUBE-SVC-XXXX 체인 (이 Service의 로드밸런싱)
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.333 -j KUBE-SEP-AAAA
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.500 -j KUBE-SEP-BBBB
-A KUBE-SVC-XXXX -j KUBE-SEP-CCCC

# KUBE-SEP-AAAA, BBBB, CCCC (개별 endpoint로의 DNAT)
-A KUBE-SEP-AAAA -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-BBBB -p tcp -j DNAT --to-destination 10.244.2.8:8080
-A KUBE-SEP-CCCC -p tcp -j DNAT --to-destination 10.244.3.2:8080
```

Service 1,000개 × backend 10개로 확장하면 **수만 개의 규칙**이 된다.

### 4.3 `iptables-restore`의 동작 방식

kube-proxy는 개별 규칙을 하나씩 추가/삭제하지 않는다. `iptables-restore --noflush` 명령을 사용한다:

1. kube-proxy가 메모리에서 **변경이 필요한 체인 전체의 규칙 세트**를 텍스트로 생성
2. 이를 `iptables-restore`에 stdin으로 전달
3. `iptables-restore`는 해당 체인들을 **통째로 교체**

`--noflush` 옵션으로 kube-proxy가 관리하지 않는 체인(예: Calico가 만든 체인)은 건드리지 않지만, **kube-proxy가 관리하는 체인들은 전부 다시 써야 한다.**

> Service 하나가 추가돼도, `KUBE-SERVICES` 체인 전체(모든 Service의 진입 규칙이 들어있는)와 관련 `KUBE-SVC-*`, `KUBE-SEP-*` 체인들이 재생성된다.

### 4.4 개별 규칙만 수정할 수 없는 이유

**확률 재계산 문제:** 로드밸런싱을 `statistic --mode random --probability`로 수행하기 때문에, Pod가 하나 추가되면 기존 Pod들의 확률값이 전부 바뀌어야 한다. 3개일 때 `0.333 / 0.500 / 1.0`이었던 것이 4개가 되면 `0.250 / 0.333 / 0.500 / 1.0`으로 변경 — 해당 `KUBE-SVC-*` 체인의 **모든 규칙**이 수정 대상이다.

**iptables의 커널 내부 구조 문제:** iptables 규칙은 커널의 `xt_table` 구조체 안에 연속된 메모리 블록(blob)으로 저장된다. 개별 규칙을 원자적으로 수정하는 것이 아니라, 체인 단위로 규칙 목록 전체를 교체하는 방식이다.

### 4.5 구체적인 성능 영향

**규칙 생성 (유저스페이스 - kube-proxy):**
- kube-proxy가 모든 Service/Endpoint 정보를 순회하며 규칙 텍스트를 생성
- Service 수에 비례하는 CPU/메모리 사용

**규칙 적용 (커널):**
- `iptables-restore`가 커널에 규칙을 로드할 때, 커널은 잠시 **테이블 락**을 잡음
- 이 동안 해당 테이블을 사용하는 패킷 처리가 일시적으로 멈출 수 있음
- 규칙이 수만 개면 이 락 시간이 눈에 띄게 길어짐

**패킷 매칭 (런타임):**
- `KUBE-SERVICES` 체인에 패킷이 도착하면, 매칭되는 규칙을 찾을 때까지 **위에서 아래로 순서대로 비교 (O(n) 선형 탐색)**
- Service가 1,000개면 최악의 경우 1,000번 비교가 필요

**실제 수치:**
- Service 5,000개 수준 → iptables 규칙 업데이트에 수 초
- 규칙 수 만 단위 초과 → 업데이트 latency가 수십 초까지 올라가는 사례 보고

---

## 5. Cilium이 eBPF를 사용하는 방식

### 5.1 kube-proxy 대체 (Service 로드밸런싱)

기존 kube-proxy(iptables 또는 IPVS 모드)를 완전히 대체한다. ClusterIP, NodePort, LoadBalancer 타입 Service의 DNAT/SNAT을 eBPF로 처리:

- TC ingress/egress 훅에 eBPF 프로그램을 부착
- eBPF Map에 Service VIP → Backend Pod IP 매핑 저장
- 패킷이 들어오면 Map 조회 후 즉시 DNAT — netfilter의 PREROUTING 체인을 거칠 필요 없음
- conntrack(연결 추적)도 eBPF Map 기반으로 자체 구현

### 5.2 네트워크 정책 (NetworkPolicy)

Kubernetes NetworkPolicy와 Cilium 자체 CiliumNetworkPolicy를 eBPF로 구현:

- 각 Pod(정확히는 Pod의 veth 인터페이스)에 eBPF 프로그램을 부착
- 패킷의 source/destination을 **Identity 기반**으로 판단 (IP가 아니라 label 기반)
- Identity → Policy 매핑이 eBPF Map에 저장
- L3/L4뿐 아니라 **L7(HTTP, gRPC, Kafka 등)** 정책도 가능

> Identity 기반의 의미: Pod IP가 바뀌어도 label이 같으면 Identity가 같으므로 정책 업데이트가 필요 없다. iptables에서 IP 기반으로 규칙을 관리하던 것과는 근본적으로 다른 접근.

### 5.3 관측성 (Hubble)

Cilium의 관측성 컴포넌트인 Hubble도 eBPF 기반:

- 패킷이 eBPF 프로그램을 지나갈 때 메타데이터(source/dest identity, verdict, L7 info 등)를 eBPF Ring Buffer Map에 기록
- 유저스페이스의 Hubble agent가 이 Map을 읽어서 flow log 생성
- 별도의 패킷 캡처나 사이드카 프록시 없이 **커널 레벨에서 관측**

---

## 6. iptables vs eBPF(Cilium) 비교

| 항목 | iptables (kube-proxy) | eBPF (Cilium) |
|------|----------------------|---------------|
| **Service 조회** | O(n) 선형 탐색 | O(1) 해시 조회 |
| **규칙 업데이트** | 체인 전체 교체 (iptables-restore) | Map entry 개별 추가/수정 (incremental) |
| **업데이트 시 락** | 테이블 전체 락 | per-CPU 최적화, 전체 테이블 락 불필요 |
| **훅 위치** | netfilter (IP 레이어 이후) | XDP/TC (NIC 드라이버 ~ qdisc 레이어) |
| **정책 기반** | IP 주소 기반 | Identity(label) 기반 |
| **L7 지원** | 불가 | 가능 (Envoy 프록시 연동) |

---

## 7. 기존 스택과의 대응 관계

전통적인 쿠버네티스 네트워크 스택과 Cilium이 대체하는 범위:

| 기능 | 전통적 스택 | Cilium |
|------|-----------|--------|
| Pod 네트워킹 (CNI) | Flannel (VXLAN/host-gw) | Cilium (VXLAN, Geneve, native routing) |
| NetworkPolicy | Calico (iptables 기반) | Cilium (eBPF, Identity 기반) |
| Service mesh / L7 | Istio (Envoy sidecar) | Cilium (eBPF + Envoy, sidecar-less 옵션) |
| Service 로드밸런싱 | kube-proxy (iptables) | Cilium (eBPF, kube-proxy 대체) |
| 관측성 | Istio telemetry | Hubble (eBPF 기반) |

> Cilium 하나가 Flannel + Calico + kube-proxy를 대체하고, Istio의 일부 기능까지 커버할 수 있다. Istio의 풀 서비스 메시 기능(mTLS, 트래픽 관리 등)을 완전히 대체하려면 Cilium Service Mesh를 추가로 설정해야 한다.

---

## 8. eBPF의 제약과 트레이드오프

- **디버깅 난이도**: 커널 내부에서 실행되므로 전통적인 디버깅 도구가 적용되지 않는다. `bpftool`, `cilium monitor` 등 전용 도구가 필요하다.
- **커널 버전 의존성**: eBPF 기능이 커널 버전마다 다르게 지원된다. Cilium이 권장하는 최소 커널 버전이 있고(보통 5.x 이상), 최신 기능에는 더 높은 버전이 필요하다.
- **Verifier 제약**: 복잡한 로직을 작성하다 보면 Verifier가 거부하는 경우가 있다. 프로그램 복잡도에 실질적 한계가 존재한다.
- **L7 처리 한계**: 순수 eBPF만으로 복잡한 L7 프로토콜 파싱은 어렵다. Cilium도 L7 정책에는 유저스페이스 Envoy 프록시를 함께 사용한다. 다만 이를 **per-node** 프록시로 운영하여 Istio의 per-pod 사이드카보다 오버헤드가 적다.