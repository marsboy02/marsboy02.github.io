# 쿠버네티스 네트워크의 본질: CNI, VXLAN, 그리고 Calico vs Cilium

## 1. 쿠버네티스의 네트워크 모델

쿠버네티스는 네트워크 구현을 **전혀 제공하지 않는다.** 대신 세 가지 근본적인 요구사항만 선언한다.

1. **모든 Pod는 NAT 없이 다른 모든 Pod와 통신할 수 있어야 한다**
2. **모든 노드는 NAT 없이 모든 Pod와 통신할 수 있어야 한다**
3. **Pod가 자기 자신이라고 인식하는 IP가 다른 Pod가 보는 IP와 같아야 한다**

한 마디로 **"클러스터 전체가 하나의 플랫한 L3 네트워크처럼 보여야 한다"**는 것이다.

Docker 기본 네트워킹에서는 컨테이너가 외부와 통신할 때 호스트 IP로 SNAT되어, 수신 측에서 보는 소스 IP가 실제 컨테이너 IP가 아닌 호스트 IP가 된다. 이러면 로깅, 보안 정책, 서비스 디스커버리가 전부 꼬인다. 쿠버네티스는 이 문제를 원천적으로 없애기 위해 "NAT 없는 플랫 네트워크"를 요구했다.

하지만 현실 세계의 물리 네트워크는 플랫하지 않다. 노드들은 다른 서브넷에 있을 수 있고, 중간에 라우터가 있고, 클라우드 VPC가 Pod IP를 알 리 없다. **이 이상과 현실의 간극을 메우는 것이 CNI 플러그인의 역할**이다.

---

## 2. CNI (Container Network Interface) — 구현이 아닌 계약

CNI는 CNCF 프로젝트로, 컨테이너의 네트워크 연결을 설정/해제하는 **인터페이스 스펙**이다. "CNI는 네트워킹 솔루션이 아니라 계약(contract)"이라는 점이 핵심이다.

### 2.1 스펙이 정의하는 것

**바이너리 인터페이스:** CNI 플러그인은 `/opt/cni/bin/`에 위치한 실행 파일이고, 컨테이너 런타임(containerd, CRI-O)이 이 바이너리를 직접 exec한다. stdin으로 JSON 설정을 받고, stdout으로 결과를 반환하는 단순한 구조다.

**오퍼레이션:** `ADD`(컨테이너를 네트워크에 연결), `DEL`(연결 해제), `CHECK`(상태 확인), `VERSION` — 이 네 가지뿐이다.

CNI 스펙 자체는 "veth를 쓸지, VXLAN을 쓸지, BGP를 쓸지"에 대해 아무것도 규정하지 않는다. 그건 전부 플러그인 구현체의 영역이다.

### 2.2 Pod 생성 시 CNI 호출 흐름

```
1. kubelet → CRI를 통해 containerd에 Pod 생성 요청
2. containerd → pause 컨테이너를 만들어 network namespace 확보
3. containerd → /etc/cni/net.d/ 에서 CNI 설정 파일 읽음
4. CNI 바이너리 exec → ADD 호출
5. CNI 플러그인 → veth pair 생성, IP 할당, 라우팅 설정
6. 결과(할당된 IP, 인터페이스 정보)를 JSON으로 반환
7. 실제 애플리케이션 컨테이너들이 이 namespace에 합류
```

### 2.3 CNI Chaining

하나의 Pod에 여러 CNI 플러그인을 체인으로 연결할 수 있다. 예를 들어 `calico → bandwidth → portmap`처럼 메인 플러그인이 네트워크를 구성하고, 이후 플러그인들이 QoS나 포트 매핑을 추가하는 방식이다.

---

## 3. 같은 노드 내 Pod 통신 — CNI 종류와 무관한 공통 구조

Pod가 생성되면 커널은 별도의 **network namespace**를 만든다. 이 namespace는 독립적인 네트워크 스택(자기만의 인터페이스, 라우팅 테이블, iptables 규칙)을 갖는다.

격리된 namespace를 호스트와 연결하기 위해 **veth pair**를 사용한다.

```
[Pod A namespace]          [호스트 namespace]          [Pod B namespace]

  eth0 ←──────────→ vethA                    vethB ←──────────→ eth0
  10.42.0.11                cni0 (브릿지)                      10.42.0.12
                         10.42.0.1
```

veth pair는 가상 이더넷 케이블이다. 한쪽 끝(eth0)은 Pod namespace 안에, 다른 쪽 끝(vethXXXX)은 호스트 namespace에 존재한다. 한쪽에 패킷을 넣으면 다른 쪽에서 나오는 커널 내부의 파이프다.

Flannel의 경우, 호스트 namespace 쪽 veth들은 **cni0**이라는 Linux 브릿지에 연결된다.

**같은 노드 내 Pod A → Pod B 통신 경로:**

1. Pod A에서 패킷 생성 (src: 10.42.0.11, dst: 10.42.0.12)
2. Pod A의 eth0 → vethA를 통해 호스트 namespace로
3. cni0 브릿지가 MAC 주소 테이블을 보고 vethB로 포워딩
4. vethB → Pod B의 eth0

순수한 L2 브릿지 동작이라 캡슐화 오버헤드가 전혀 없다.

---

## 4. 다른 노드 간 Pod 통신 — 핵심 문제

```
[Node 1: 192.168.1.10]              [Node 2: 192.168.1.20]
  Pod A: 10.42.0.11                   Pod C: 10.42.1.15
  Pod B: 10.42.0.12                   Pod D: 10.42.1.16
```

Pod A(10.42.0.11)가 Pod C(10.42.1.15)로 패킷을 보내려 할 때:

- 10.42.1.15라는 IP는 Node 2 안에서만 의미 있는 주소
- 물리 네트워크의 라우터는 Pod CIDR을 전혀 모름
- 패킷이 Node 1을 벗어나는 순간, 물리 네트워크는 이 패킷을 어디로 보내야 할지 알 수 없음

해결 방법은 크게 두 가지:

- **언더레이 방식 (Calico BGP):** 물리 네트워크에 Pod 대역 라우팅을 가르친다
- **오버레이 방식 (Flannel VXLAN 등):** 원본 패킷을 감싸서 물리 네트워크가 이해하는 주소로 전달한다

---

## 5. VXLAN (Virtual eXtensible LAN)

### 5.1 핵심 아이디어

**L2 프레임을 UDP 패킷 안에 넣어서 L3 네트워크를 통해 전달한다.** WireGuard가 "IP 패킷을 UDP로 감싸서 보내는" 캡슐화와 같은 패턴이지만, 감싸는 대상이 IP 패킷이 아니라 **이더넷 프레임 전체**라는 점이 다르다.

VXLAN의 원래 목적은 "물리적으로 떨어진 네트워크를 하나의 L2 세그먼트처럼 보이게 만드는 것"이다. 데이터센터에서 VLAN의 4,096개 ID 제한을 넘기 위해 만들어진 기술인데, 쿠버네티스 오버레이 네트워크에 재활용되었다.

### 5.2 캡슐화 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 외부 Ethernet │ 외부 IP  │ 외부 UDP │ VXLAN  │ 내부 Ethernet │ 내부 IP │ Payload │
│   14 bytes    │ 20 bytes │ 8 bytes  │ 8 bytes│   14 bytes    │ 20 bytes│         │
└─────────────────────────────────────────────────────────────────────────┘

외부 Ethernet: src=Node1 MAC, dst=Node2 MAC (또는 게이트웨이 MAC)
외부 IP:       src=192.168.1.10 (Node 1), dst=192.168.1.20 (Node 2)
외부 UDP:      src=<hash>, dst=8472 (Linux VXLAN 기본 포트, 표준은 4789)
VXLAN Header:  VNI (VXLAN Network Identifier) = 1
내부 Ethernet: src=Pod A MAC, dst=Pod C MAC
내부 IP:       src=10.42.0.11 (Pod A), dst=10.42.1.15 (Pod C)
Payload:       실제 데이터
```

물리 네트워크 입장에서 이 패킷은 "Node 1이 Node 2에게 보내는 평범한 UDP 패킷"이다.

### 5.3 오버헤드 계산

| 구성 요소 | 크기 |
|---|---|
| 외부 IP 헤더 | 20 bytes |
| 외부 UDP 헤더 | 8 bytes |
| VXLAN 헤더 | 8 bytes |
| 내부 Ethernet 헤더 | 14 bytes |
| **합계** | **50 bytes** |

MTU 1500 환경에서 VXLAN을 쓰면, 내부 패킷은 **1450바이트**까지만 사용할 수 있다. Flannel이 Pod 인터페이스의 MTU를 1450으로 설정하는 이유가 바로 이 50바이트 오버헤드 때문이다.

**비교:**
- VXLAN: 50 bytes (L2 오버레이, 암호화 없음)
- WireGuard: 60 bytes (IPv4 20 + UDP 8 + WireGuard 32, 암호화 포함)
- 이중 캡슐화 (VXLAN + WireGuard): 110 bytes → 유효 MTU = 1390

### 5.4 VXLAN 헤더 상세

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│ Flags (8)       │          Reserved (24)                         │
├─────────────────┼───────────────────────────────────────────────┤
│          VNI (24)                                 │ Reserved (8)│
└───────────────────────────────────────────────────┴─────────────┘
```

**VNI (VXLAN Network Identifier)** 24비트 → 약 1,677만 개의 논리적 네트워크 생성 가능. VLAN의 12비트(4,096개) 대비 압도적이다. Flannel에서는 보통 VNI=1을 사용한다.

---

## 6. VTEP과 FDB — VXLAN의 주소 학습 메커니즘

### 6.1 VTEP (VXLAN Tunnel End Point)

Flannel 환경에서 `flannel.1` 디바이스가 VTEP이다. 캡슐화/디캡슐화를 수행하며, "내부 MAC 주소를 어떤 외부 IP로 매핑할 것인가"를 **FDB(Forwarding Database)**로 관리한다.

```bash
# FDB 확인
bridge fdb show dev flannel.1
# aa:bb:cc:dd:ee:ff dst 192.168.1.20 self permanent
# → "이 MAC 주소를 가진 VTEP은 192.168.1.20에 있다"
```

WireGuard의 cryptokey routing과 개념적으로 대응된다:

| | WireGuard | VXLAN |
|---|---|---|
| 매핑 | IP 대역 → public key(피어) | MAC → VTEP IP |
| 관리 주체 | Tailscale coordination 서버 | Flannel flanneld |

### 6.2 BUM 트래픽 문제

**BUM (Broadcast, Unknown unicast, Multicast)** — 일반 L2 네트워크에서 스위치가 목적지 MAC을 모르면 모든 포트로 플러딩한다. VXLAN에서는 "모든 포트"가 "모든 원격 VTEP"을 의미하게 되어 심각한 확장성 문제가 발생한다.

**순수 VXLAN 스펙의 방법:** 모든 VTEP이 멀티캐스트 그룹에 가입하고 BUM 트래픽을 멀티캐스트로 전파. 하지만 대부분의 클라우드 환경은 멀티캐스트를 지원하지 않는다.

**Flannel의 해결책:** flanneld가 **컨트롤 플레인에서 FDB와 ARP 엔트리를 미리 채워넣는다(prepopulate)**. 노드가 클러스터에 조인하면 flanneld가 모든 노드의 FDB와 ARP 테이블에 정보를 직접 주입한다.

```bash
# Flannel이 자동 관리하는 ARP 엔트리
ip neigh show dev flannel.1
# 10.42.1.0 lladdr aa:bb:cc:dd:ee:ff PERMANENT
# → flanneld가 미리 넣어준 것, 실제 ARP 브로드캐스트 불필요
```

이는 **"데이터플레인의 문제를 컨트롤 플레인으로 끌어올려서 해결"**하는 패턴이다. Tailscale의 coordination 서버가 피어 정보를 미리 배포하는 것과 정확히 같다.

---

## 7. Flannel + VXLAN 노드 간 통신 전체 흐름

Pod A (Node 1, 10.42.0.11) → Pod C (Node 2, 10.42.1.15)

### Node 1 (송신)

**1단계 — Pod 내부 라우팅 결정:**
Pod A namespace의 라우팅 테이블은 단순하다.

```
default via 10.42.0.1 dev eth0
```

10.42.1.15는 로컬 서브넷이 아니므로 default route를 타고 eth0(veth의 Pod 쪽 끝)으로 나간다.

**2단계 — 호스트 namespace 도착, 라우팅 결정:**
호스트의 라우팅 테이블에서 핵심 엔트리:

```
10.42.0.0/24 dev cni0                      # 로컬 Pod 대역 → 브릿지
10.42.1.0/24 via 10.42.1.0 dev flannel.1   # Node 2의 Pod 대역 → VXLAN 디바이스
```

목적지 10.42.1.15는 10.42.1.0/24에 매칭 → `flannel.1` 디바이스로 전달.

**3단계 — VXLAN 캡슐화:**
`flannel.1`(VTEP)에 패킷이 들어오면 커널 VXLAN 모듈이:

1. FDB 조회 → "10.42.1.0/24 대역은 Node 2(192.168.1.20)에 있다"
2. 원본 패킷을 내부 Ethernet 프레임으로 감쌈
3. VXLAN 헤더(VNI=1) 추가
4. 외부 UDP 헤더(dst port=8472) 추가
5. 외부 IP 헤더(src=192.168.1.10, dst=192.168.1.20) 추가

**4단계 — 물리 네트워크 전송:**
호스트의 실제 NIC(eth0)를 통해 전송. 물리 네트워크는 평범한 UDP 패킷으로 처리.

### Node 2 (수신)

**5단계 — VXLAN 디캡슐화:**
커널이 UDP 포트 8472를 보고 VXLAN 모듈로 전달 → 외부 헤더를 벗기고 내부 이더넷 프레임 추출.

**6단계 — 호스트 라우팅 → Pod 전달:**
디캡슐화된 패킷(dst=10.42.1.15)은 `10.42.1.0/24 dev cni0` 라우트를 타고 cni0 브릿지 → Pod C의 veth로 전달.

**7단계 — Pod C 수신:**
Pod C가 보는 소스 IP는 10.42.0.11 (Pod A의 실제 IP). NAT이 없으므로 쿠버네티스 네트워크 모델 충족.

---

## 8. Flannel의 역할과 한계

각 노드에서 실행되는 `flanneld` 데몬이 관리하는 것:

- **서브넷 할당:** 클러스터 조인 시 자기 노드의 Pod CIDR 할당 (Node 1 = 10.42.0.0/24, Node 2 = 10.42.1.0/24)
- **flannel.1 디바이스 생성:** VXLAN 타입 네트워크 디바이스
- **cni0 브릿지 생성:** 로컬 Pod들이 연결될 브릿지
- **라우팅 엔트리 관리:** 다른 노드의 Pod CIDR → flannel.1 경로 추가/삭제
- **FDB/ARP 엔트리 관리:** VTEP 간 매핑 정보 사전 주입

**Flannel의 한계:** 오직 오버레이 네트워크 구성만 담당한다. NetworkPolicy 없음, BGP 없음, L7 처리 없음. k3s의 기본 CNI로 별도 옵션 없이 설치하면 Flannel이 자동 배포되며, 기본 백엔드는 VXLAN이다.

---

## 9. NIC 하드웨어 오프로딩

VXLAN은 오래된 표준이라 대부분의 서버급 NIC가 하드웨어 오프로딩을 지원한다:

- **TX offload:** 캡슐화를 CPU가 아니라 NIC가 수행
- **RX offload:** 디캡슐화를 NIC가 수행
- **TSO/GRO:** 캡슐화된 패킷에 대해서도 TCP Segmentation Offload와 Generic Receive Offload 작동

```bash
ethtool -k eth0 | grep vxlan
# tx-udp_tnl-segmentation: on
# tx-udp_tnl-csum-segmentation: on
```

WireGuard는 비교적 새롭고 암호화가 포함되어 NIC 하드웨어 오프로딩 지원이 제한적이다.

---

## 10. Calico — netfilter 기반의 성숙한 아키텍처

### 10.1 데이터플레인

Calico의 전통적 데이터플레인은 Linux 커널의 라우팅 스택과 iptables를 그대로 활용한다. netfilter의 다섯 가지 훅(PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING) 중 주로 `FORWARD` 체인과 `filter` 테이블에 규칙을 삽입하여 NetworkPolicy를 구현한다.

```
Pod A (eth0) → veth → 호스트 라우팅 테이블 → iptables FORWARD chain
→ Calico policy rules → 호스트 라우팅 테이블 → veth → Pod B (eth0)
```

각 Pod의 veth 인터페이스에 `/32` 호스트 라우트를 설정하고, Pod 간 트래픽은 호스트의 라우팅 테이블을 통해 전달된다.

### 10.2 노드 간 통신: 세 가지 모드

**BGP 모드 (기본):** 각 노드에서 BIRD 데몬이 Pod CIDR을 BGP로 광고. 물리 네트워크가 BGP를 지원하면 오버레이 없이 순수 L3 라우팅으로 통신 → **MTU 오버헤드 제로.**

**VXLAN 모드:** BGP를 쓸 수 없는 환경(대부분의 클라우드)에서 VXLAN 오버레이 사용. 50바이트 캡슐화 오버헤드.

**IPIP 모드:** IP-in-IP 캡슐화로 20바이트만 추가. VXLAN보다 오버헤드가 작지만 일부 네트워크 장비에서 호환성 문제 가능.

### 10.3 아키텍처 컴포넌트

- **Felix (DaemonSet):** 각 노드에서 실행. etcd/Kubernetes API에서 리소스 변경을 watch하고 iptables 규칙과 라우트 엔트리로 변환하여 커널에 적용.
- **BIRD:** BGP 데몬. 노드 간 Pod CIDR 라우트 교환.
- **eBPF 모드 (최근 추가):** iptables 대신 eBPF 프로그램으로 정책 적용. 기존 아키텍처에 eBPF를 얹은 형태.

### 10.4 NetworkPolicy

Kubernetes 표준 NetworkPolicy를 완전 지원하면서, 자체 CRD(`GlobalNetworkPolicy`, `NetworkSet` 등)로 확장. 클러스터 전체에 걸친 기본 deny 정책 등을 쉽게 설정할 수 있다.

---

## 11. Cilium — eBPF-native 아키텍처

### 11.1 핵심 철학: netfilter 우회

Cilium의 핵심 아이디어는 **"netfilter를 우회하자"**이다. eBPF 프로그램을 **TC (Traffic Control) 훅**과 **XDP (eXpress Data Path) 훅**에 직접 부착한다.

```
전통적 경로 (Calico/iptables):
  NIC → [netfilter PREROUTING] → routing → [netfilter FORWARD]
  → [netfilter POSTROUTING] → NIC

Cilium eBPF 경로:
  NIC → [XDP/TC eBPF 프로그램] → 직접 redirect → 대상 Pod veth
```

iptables는 수백~수천 개의 규칙을 선형 탐색하지만, Cilium은 **eBPF 맵(해시 테이블)**을 사용하여 O(1) 룩업으로 정책을 평가한다.

### 11.2 규모에서의 차이

1,000개 Pod + 100개 NetworkPolicy 시:

- **iptables:** 관련 규칙 수천 줄로 폭발, 업데이트 시 전체 체인 재작성
- **Cilium:** eBPF 맵 엔트리만 원자적으로 업데이트

### 11.3 kube-proxy 대체

Cilium은 kube-proxy를 완전히 대체한다:

- Service VIP → backend Pod 매핑을 eBPF 맵에 저장
- TC 훅에서 DNAT 수행
- conntrack도 eBPF 맵으로 자체 구현
- `iptables -t nat -L` 해도 Service 관련 규칙이 없음

### 11.4 Identity 기반 보안 모델

Calico가 IP 기반으로 정책을 적용하는 것과 달리, Cilium은 **Identity** 개념을 도입한다:

- 각 Pod에 label 조합 기반의 numeric identity 부여
- 같은 노드 내: skb의 mark에 인코딩
- 노드 간: VXLAN/Geneve 헤더의 옵션 필드에 인코딩
- Pod IP 변경에 무관하게 label이 같으면 동일 identity → 정책 유지

### 11.5 L7 가시성 (Hubble)

Cilium 내장 관측 도구. eBPF로 수집한 네트워크 플로우를 L7까지 관측한다. HTTP 메서드/경로/상태 코드, gRPC, Kafka, DNS 프로토콜 파싱 지원. Istio 사이드카의 L7 가시성 기능 일부를 CNI 레벨에서 제공하는 셈이다.

---

## 12. 구조적 비교

| 관점 | Calico (iptables 모드) | Cilium |
|---|---|---|
| **패킷 처리 위치** | netfilter 훅 (PREROUTING → FORWARD → POSTROUTING) | TC/XDP eBPF 훅 (netfilter 우회) |
| **정책 룩업** | iptables 규칙 선형 탐색 | eBPF 맵 해시 룩업 O(1) |
| **정책 업데이트** | 전체 iptables 체인 재작성 | eBPF 맵 엔트리 원자적 업데이트 |
| **kube-proxy** | 별도 운용 (또는 IPVS 모드) | 완전 대체 |
| **보안 모델** | IP 기반 | Identity (label) 기반 |
| **노드 간 통신** | BGP / VXLAN / IPIP | VXLAN / Geneve / WireGuard 내장 |
| **L7 처리** | 없음 (Istio 등 별도 필요) | Envoy 프록시 내장 + Hubble |
| **커널 요구사항** | 특별 요구 없음 | 커널 4.19+ (권장 5.10+) |
| **디버깅 도구** | iptables -L, route, tcpdump | cilium monitor, hubble, bpftool |

---

## 13. Overlay vs Underlay 트레이드오프

**오버레이 (Flannel VXLAN, Cilium VXLAN/Geneve):**
물리 네트워크를 건드리지 않아도 된다. 어떤 인프라에서든 동작 — 클라우드, 온프레미스, 서로 다른 서브넷 환경. 대신 캡슐화 오버헤드와 CPU 비용 발생.

**언더레이 (Calico BGP):**
캡슐화 없으니 오버헤드 제로, 최대 성능. 대신 물리 네트워크가 BGP를 지원해야 하고 Pod CIDR 라우트를 수용해야 한다. 클라우드 환경에서는 대부분 불가능.

---

## 14. WireGuard vs VXLAN 비교 정리

둘 다 "원본 패킷을 UDP로 감싸서 물리 네트워크를 통과시키는" 터널링이지만 목적이 다르다.

| | WireGuard | VXLAN |
|---|---|---|
| **목적** | 암호화 터널 | L2 오버레이 |
| **감싸는 대상** | L3 (IP 패킷) | L2 (이더넷 프레임 전체) |
| **암호화** | ChaCha20-Poly1305 | 없음 |
| **오버헤드** | 60 bytes | 50 bytes |
| **피어 식별** | public key (cryptokey routing) | VTEP IP (FDB) |
| **컨트롤 플레인** | Tailscale coordination 서버 | Flannel flanneld |
| **NIC 오프로딩** | 제한적 | 광범위한 하드웨어 지원 |

공통 패턴: **"데이터플레인의 문제를 컨트롤 플레인으로 끌어올려 해결."** WireGuard는 Tailscale coordination 서버가 피어 정보를 배포하고, VXLAN은 flanneld가 FDB/ARP 엔트리를 사전 주입한다.

---

## 부록: 환경 확인 명령어

```bash
# CNI 바이너리 확인
ls /opt/cni/bin/

# CNI 설정 확인
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist

# 어떤 CNI가 돌고 있는지
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium"

# k3s 시작 옵션 확인
cat /etc/systemd/system/k3s.service
ps aux | grep k3s

# VXLAN 디바이스 확인
ip -d link show flannel.1

# FDB 확인
bridge fdb show dev flannel.1

# ARP 엔트리 확인
ip neigh show dev flannel.1

# VXLAN 오프로드 확인
ethtool -k eth0 | grep vxlan

# Pod 인터페이스 MTU 확인 (1450이면 VXLAN 50바이트 오버헤드 반영)
kubectl exec <pod> -- ip link show eth0
```