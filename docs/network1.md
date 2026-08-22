# 네트워크 기초 이론 정리

> 이 문서는 OS 레벨에서 물리 장비까지, 패킷이 실제로 어떻게 움직이는지를 다룬다.
> ifconfig 출력을 읽는 법부터 시작해서 커널 네트워크 스택, netfilter, 터널링까지 이어진다.

---

## 1. 네트워크 인터페이스란 무엇인가

네트워크 인터페이스는 **운영체제 커널이 네트워크와 대화하기 위해 만든 추상화된 출입구(door)**다.

프로그램은 하드웨어를 직접 제어할 수 없다. Wi-Fi 칩에 직접 명령을 내리거나 NIC 메모리에 접근하는 것은 불가능하며, 반드시 커널을 통해야 한다. 인터페이스는 커널이 제공하는 이 "문"이고, 프로그램은 소켓을 통해 이 문에 데이터를 넘기면 된다.

### 핵심 원칙: 인터페이스 뒤에 반드시 물리 하드웨어가 있을 필요는 없다

| 인터페이스 | 문 뒤에 있는 것 | 설명 |
|---|---|---|
| en0 | Wi-Fi 칩 (하드웨어) | 패킷이 전파로 변환되어 물리적으로 전송됨 |
| lo0 | 커널 메모리 (아무것도 없음) | 패킷이 밖으로 나가지 않고 커널 안에서 즉시 되돌아옴 |
| utun / wg0 | 유저스페이스 프로세스 | VPN 앱(Tailscale 등)이 패킷을 받아서 암호화 후 다시 물리 인터페이스로 전송 |
| bridge0 | 여러 인터페이스를 묶는 가상 스위치 | L2 레벨에서 프레임을 포워딩 |
| veth | 컨테이너/Pod의 네트워크 네임스페이스 | 한쪽에서 넣은 패킷이 반대쪽에서 나오는 파이프 |

커널 입장에서는 이 모든 인터페이스가 동일한 추상화로 취급된다. "패킷을 넣으면 어딘가로 간다"는 인터페이스의 계약(contract)은 물리든 가상이든 동일하다.

---

## 2. 이더넷(Ethernet)

### 정의

이더넷은 **같은 네트워크 안에서 장비들이 데이터를 주고받는 규칙**이다. OSI 모델에서 L1(물리)과 L2(데이터 링크)를 정의하는 IEEE 802.3 표준이다.

이름의 유래는 19세기 물리학에서 빛이 전달되는 매질이라 믿었던 "에테르(Ether)"로, 네트워크라는 보이지 않는 매질을 통해 데이터가 전달된다는 비유다.

### 이더넷 프레임 구조

이더넷의 핵심 전송 단위는 **프레임(Frame)**이다.

```
┌──────────┬──────────┬──────────┬────────┬─────────────────┬──────┐
│ Preamble │ Dst MAC  │ Src MAC  │  Type  │    Payload      │ FCS  │
│ 8 bytes  │ 6 bytes  │ 6 bytes  │2 bytes │ 46-1500 bytes   │4 bytes│
└──────────┴──────────┴──────────┴────────┴─────────────────┴──────┘
```

각 필드의 역할:

- **Preamble (8 bytes)**: 수신 측 NIC의 클럭 동기화를 위한 패턴 (`10101010...`). 마지막 1바이트(SFD)는 `10101011`로 "여기서 프레임이 시작된다"는 신호다.
- **Dst MAC (6 bytes)**: 목적지 MAC 주소. 48비트 하드웨어 주소. `ff:ff:ff:ff:ff:ff`면 브로드캐스트.
- **Src MAC (6 bytes)**: 출발지 MAC 주소. ifconfig의 `ether` 필드에 표시되는 값.
- **Type (2 bytes)**: 페이로드에 담긴 상위 프로토콜 식별자. L2와 L3를 연결하는 다리 역할.
  - `0x0800` = IPv4
  - `0x86DD` = IPv6
  - `0x0806` = ARP
- **Payload (46–1500 bytes)**: 상위 계층의 데이터 (IP 패킷 등). 최대 1500바이트가 표준 MTU.
- **FCS (4 bytes)**: Frame Check Sequence. CRC-32 알고리즘으로 프레임 무결성 검증.

### MAC 주소

MAC(Media Access Control) 주소는 48비트(6바이트)의 하드웨어 식별자다.

```
96:60:2d:e2:2f:03
│        │
├────────┤
│ OUI    │ NIC 고유번호
│(제조사)│
```

- 앞 3바이트 (OUI, Organizationally Unique Identifier): 제조사 식별
- 뒤 3바이트: 제조사가 부여한 고유 번호
- MAC 주소는 동일 네트워크 세그먼트(L2) 안에서만 의미가 있다. 라우터를 넘어가면 MAC은 다음 홉의 것으로 교체된다.

### Wi-Fi와 이더넷의 관계

Wi-Fi(802.11)는 무선 이더넷이다. 동일한 프레임 구조를 공유하며, MAC 주소 기반으로 통신한다. macOS에서 Wi-Fi 인터페이스가 `en0`(Ethernet의 약자)인 이유가 이것이다.

---

## 3. macOS 인터페이스 이름 규칙

macOS의 인터페이스 이름은 BSD 계열에서 물려받은 **드라이버명 + 인스턴스 번호** 패턴이다.

### 접두사별 의미

| 접두사 | 의미 | 예시 |
|---|---|---|
| `en` | Ethernet (유선 + Wi-Fi 포함) | en0 = Wi-Fi, en1-3 = Thunderbolt |
| `lo` | Loopback | lo0 = 127.0.0.1 |
| `utun` | User-space Tunnel | utun1 = Tailscale (WireGuard) |
| `bridge` | L2 Bridge | bridge0 = Thunderbolt Bridge |
| `awdl` | Apple Wireless Direct Link | awdl0 = AirDrop |
| `llw` | Low Latency WLAN | llw0 = Apple Watch 연결 |
| `ap` | Access Point | ap1 = Personal Hotspot |
| `anpi` | Apple Network Port Interface | anpi0-2 = Apple Silicon 예비 포트 |
| `gif` | Generic IP-in-IP tunnel | gif0 = 레거시 터널 |
| `stf` | Six To Four | stf0 = IPv6 전환 터널 |

### Apple Silicon Mac의 en 번호 할당

- `en0`: 항상 Wi-Fi (Intel Mac에서는 유선이 en0, Wi-Fi가 en1이었지만 Apple Silicon에서 반전됨)
- `en1, en2, en3`: Thunderbolt 포트의 네트워크 기능 (bridge0의 멤버)
- `en4, en5, en7 등`: USB 이더넷 어댑터 연결 이력. macOS가 어댑터의 MAC 주소에 번호를 영구 매핑

번호가 연속이 아닌 이유는 부팅 시 하드웨어 enumerate 순서에 따라 할당되며, 내부적으로 다른 가상 인터페이스에 번호가 이미 사용된 경우가 있기 때문이다.

### Linux와의 차이

| 항목 | macOS | Linux (systemd) |
|---|---|---|
| 명명 규칙 | 발견 순서 기반 (en0, en1...) | 물리 위치 기반 (enp3s0, ens1) |
| 장점 | 단순 | 하드웨어 변경 시 이름 불변 |
| 네트워크 스택 | XNU (BSD 계열 ifnet) | Linux 고유 (net_device) |
| 방화벽 | pf (Packet Filter) | netfilter (iptables/nftables) |

---

## 4. ifconfig 출력 읽는 법

### 기본 구조

```
인터페이스명: flags=값<플래그들> mtu 값
    options=값<옵션들>
    ether MAC주소          ← L2 (Data Link)
    inet IPv4주소          ← L3 (Network)
    inet6 IPv6주소         ← L3 (Network)
    media: ...             ← L1 (Physical)
    status: active/inactive
```

이 구조 자체가 OSI 레이어 순서 — Physical → Data Link → Network 순으로 정보가 나열된다.

### flags 해석

| 플래그 | 의미 |
|---|---|
| UP | 커널에 의해 인터페이스 활성화됨 |
| RUNNING | L1 레벨에서 링크 살아있음. UP인데 RUNNING 없으면 "드라이버는 올라왔지만 케이블 빠진" 상태 |
| BROADCAST | 브로드캐스트 가능 (이더넷 계열) |
| SIMPLEX | 자신이 보낸 패킷을 자신이 다시 수신하지 않음 (BSD 전통) |
| MULTICAST | 멀티캐스트 그룹 참여 가능 |
| SMART | macOS 전용. 트래픽 기반 전력 관리 |
| LOOPBACK | lo0 전용. 자기 자신에게 보내는 트래픽 전용 |
| POINTOPOINT | 터널 인터페이스 (utun). 양 끝점만 연결하는 1:1 링크 |
| PROMISC | Promiscuous mode. 자기 MAC이 아닌 패킷도 전부 수신. bridge 멤버에서 활성화 |

### options (NIC 하드웨어 오프로드 기능) 해석

| 옵션 | 의미 |
|---|---|
| RXCSUM / TXCSUM | 수신/송신 체크섬을 NIC가 계산 (CPU 부하 절감) |
| TSO4 / TSO6 | TCP Segmentation Offload. 커널이 큰 세그먼트를 통째로 NIC에 넘기면 NIC가 MTU 크기로 분할. tcpdump에서 MTU보다 큰 패킷이 보이는 이유 |
| CHANNEL_IO | Apple 전용 고성능 I/O 채널 기반 패킷 처리 경로 |

### 서브넷 마스크 읽기

```
inet 192.168.20.25 netmask 0xfffffe00 broadcast 192.168.21.255
```

`0xfffffe00`을 이진수로 풀면:

```
0xfffffe00 = 11111111.11111111.11111110.00000000
           = 255.255.254.0
           = /23 (CIDR)
```

/23은 2개의 C클래스를 합친 512개 IP 범위 (192.168.20.0 ~ 192.168.21.255).

---

## 5. TUN 인터페이스와 VPN 터널링

### TUN이란

TUN(Tunnel) 인터페이스는 커널이 만드는 **가상 네트워크 카드**다. 물리 NIC가 하드웨어를 제어하는 것처럼, TUN은 유저스페이스 프로세스에 패킷을 전달한다.

### OS별 TUN 인터페이스 이름

| OS | 인터페이스 | 이유 |
|---|---|---|
| Linux | `wg0` | WireGuard가 커널 모듈이라 자기 전용 타입 생성 가능 |
| macOS | `utun1` | Apple이 커널 확장(kext) 차단, 범용 utun을 빌려 씀 |
| Linux (범용) | `tun0` | OpenVPN 등이 사용하는 범용 터널 |

### TUN vs TAP

| 구분 | TUN | TAP |
|---|---|---|
| 동작 계층 | L3 (IP 패킷) | L2 (이더넷 프레임 전체) |
| 주고받는 데이터 | IP 헤더부터 시작 | MAC 주소, 이더넷 헤더 포함 |
| 용도 | 대부분의 VPN (WireGuard, Tailscale, OpenVPN) | L2 overlay 필요 시 (DHCP 브로드캐스트 전달, ARP 직접 처리) |
| k8s 연관 | - | Flannel VXLAN 모드가 TAP과 유사한 L2 overlay |

### WireGuard 패킷 여정 (Linux 기준)

```
1. 앱이 100.64.0.5:443으로 TCP 연결 요청
2. 커널 라우팅 테이블: "100.64.0.0/10 → wg0"
3. 패킷이 wg0(TUN) 인터페이스로 진입
4. WireGuard가 패킷 전체를 ChaCha20-Poly1305로 암호화
5. 암호화된 데이터를 새 UDP 패킷(port 51820)으로 포장
6. 새 패킷에 새 IP 헤더(실제 peer의 공인 IP) 부착
7. 이 UDP 패킷이 eth0/en0을 통해 인터넷으로 전송
```

### Linux vs macOS WireGuard 구현 차이

```
Linux:   앱 → 커널 → wg0 (커널 내 암호화) → eth0 → 와이어
macOS:   앱 → 커널 → utun (유저스페이스로 올림) → Tailscale (암호화) → 커널 → en0 → 와이어
```

Linux에서는 WireGuard가 커널 모듈(Linux 5.6+)이므로 암호화가 커널 안에서 바로 일어난다. macOS에서는 Apple의 커널 확장 차단 정책으로 인해 유저스페이스에서 암호화가 일어나며, 커널 ↔ 유저스페이스 간 한 번의 추가 복사(U턴)가 발생한다.

### MTU와 캡슐화 오버헤드

WireGuard 캡슐화 시 추가되는 바이트:

```
20 bytes (outer IPv4 header)
 8 bytes (UDP header)
32 bytes (WireGuard header)
16 bytes (Poly1305 auth tag)
 4 bytes (misc)
─────────
~80 bytes total overhead
```

Tailscale은 보수적으로 utun MTU를 1380으로 설정한다 (1500 - 120).

ifconfig에서 utun의 MTU가 1380인 것은 Tailscale/WireGuard 터널이라는 강력한 지표다.

---

## 6. Bridge 인터페이스

### 정의

Bridge는 여러 물리(또는 가상) 인터페이스를 **하나의 L2 브로드캐스트 도메인으로 묶는 가상 스위치**다.

### macOS bridge0의 역할

macOS의 bridge0은 Thunderbolt 포트의 네트워크 인터페이스(en1, en2, en3)를 묶는다. 이를 통해 Thunderbolt 케이블로 연결된 여러 Mac이 마치 같은 이더넷 스위치에 꽂혀 있는 것처럼 L2 레벨에서 직접 통신할 수 있다.

```
bridge0 (가상 L2 스위치)
├── en1 (Thunderbolt port 1) ── Mac B
├── en2 (Thunderbolt port 2) ── Mac C
└── en3 (Thunderbolt port 3) ── Mac D
```

bridge0이 없다면 en1과 en2는 별개의 네트워크 세그먼트이므로, Mac B와 Mac C가 통신하려면 L3 라우팅이 필요하다.

### 멤버 인터페이스의 PROMISC 플래그

bridge0의 멤버인 en1/en2/en3에 PROMISC(Promiscuous mode) 플래그가 켜지는 이유:

- 브리지 포트는 자기 MAC이 아닌 프레임도 전부 수신해야 함
- 수신한 프레임의 Src MAC을 보고 MAC 학습 테이블을 유지 (`LEARNING` 플래그)
- 목적지 MAC이 어느 포트 뒤에 있는지 알면 해당 포트로만 전달 (스위칭)

### k8s와의 연관

k8s에서 Flannel의 `cni0` 브리지가 동일한 역할을 한다. 같은 노드의 Pod들의 veth 인터페이스를 cni0 브리지로 묶어서 L2 레벨에서 직접 통신하게 한다. macOS의 bridge0은 "Thunderbolt로 연결된 Mac들을 위한 cni0"과 같은 개념이다.

---

## 7. 커널과 네트워크의 관계

### 왜 네트워크가 커널에 있는가

커널은 하드웨어와 프로그램 사이의 유일한 중재자다. 이것은 모니터, 키보드, 마우스와 동일한 원칙이다.

1. **NIC는 하드웨어** → 커널만 제어 가능
2. **여러 앱이 동시에 네트워크 사용** → 누군가 중재해야 함 → 커널의 역할
3. **라우팅, 방화벽 등 공통 정책** → 모든 앱에 일관되게 적용 → 커널이 한 곳에서 처리

### Linux 커널 네트워크 스택의 구조

패킷이 커널을 통과하는 계층 (위에서 아래로):

```
┌─────────────────────────────────────────┐
│           Userspace (apps)              │ ← curl, nginx, Tailscale
│  socket(), send(), recv(), ioctl()      │
├─────────────────────────────────────────┤
│  ┌───────────────────────────────────┐  │
│  │         Socket layer              │  │ ← per-process fd, buffers
│  ├───────────────────────────────────┤  │
│  │       TCP / UDP / ICMP            │  │ ← L4 transport processing
│  ├───────────────────────────────────┤  │
│  │    IP layer ←→ Netfilter hooks    │  │ ← L3 routing + 검문소
│  ├───────────────────────────────────┤  │
│  │    Network device subsystem       │  │ ← 인터페이스 관리, tc, bridging
│  ├───────────────────────────────────┤  │
│  │  NIC driver │ TUN driver │ veth   │  │ ← 하드웨어/가상 디바이스 드라이버
│  └───────────────────────────────────┘  │
│              Kernel space               │
└─────────────────────────────────────────┘
```

각 층의 역할:

- **Socket layer**: 앱이 커널과 대화하는 유일한 창구. socket() 시스템 콜로 소켓 fd를 생성하고, send/recv로 데이터 교환.
- **Transport layer**: TCP면 시퀀스 번호, 흐름 제어, 재전송, 혼잡 제어. UDP면 거의 passthrough.
- **IP layer + Routing**: 라우팅 테이블을 조회해서 "이 패킷은 어떤 인터페이스로 나가야 하는가" 결정. `ip route show`의 테이블이 여기서 참조됨.
- **Network device subsystem**: 인터페이스 목록, MTU, 상태(UP/DOWN), 큐잉 규칙(tc/qdisc) 등을 관리.
- **Device drivers**: 하드웨어 제어(NIC driver), 유저스페이스 연결(TUN driver), 컨테이너 연결(veth driver).

### macOS vs Linux 커널 네트워크 비교

```
macOS XNU 커널                    Linux 커널
├─ ifnet (BSD 네트워크 스택)       ├─ net_device (리눅스 네트워크 스택)
├─ pf (Packet Filter)             ├─ netfilter (+ iptables/nftables)
├─ utun (NetworkExtension)        ├─ tun/tap, wireguard 모듈
└─ bridge (ifnet bridging)        └─ bridge, veth, vxlan 모듈
```

---

## 8. Netfilter — 커널의 패킷 검문소

### 핵심 개념

Netfilter는 독립적인 모듈이 아니라, **IP layer에 박혀있는 hook point들**이다. 패킷이 IP 처리를 거치는 과정에서 특정 지점마다 등록된 규칙을 실행한다.

**비유**: 패킷이 고속도로를 달리다가 톨게이트(netfilter hook)를 만나는 것. 톨게이트가 목적지를 정해주는 게 아니라, 지나가는 차를 검사하는 것이다.

### Netfilter의 5개 hook point

```
                        ┌──────────────┐
                        │  NIC 수신    │
                        └──────┬───────┘
                               ▼
                    ┌─────────────────────┐
                    │  1. PREROUTING      │ ← raw, conntrack, mangle, nat(DNAT)
                    └──────────┬──────────┘
                               ▼
                    ┌─────────────────────┐
                    │   Routing Decision  │
                    └───┬─────────────┬───┘
                        │             │
                   For me?        Not for me?
                        ▼             ▼
              ┌──────────────┐ ┌──────────────┐
              │  2. INPUT    │ │  3. FORWARD  │
              │ mangle,filter│ │ mangle,filter│
              └──────┬───────┘ └──────┬───────┘
                     ▼                │
              ┌──────────────┐        │
              │  Local App   │        │
              └──────┬───────┘        │
                     ▼                │
              ┌──────────────┐        │
              │  4. OUTPUT   │        │
              │raw,mangle,nat│        │
              │   filter     │        │
              └──────┬───────┘        │
                     ▼                ▼
                    ┌─────────────────────┐
                    │  5. POSTROUTING     │ ← mangle, nat(SNAT/MASQUERADE)
                    └──────────┬──────────┘
                               ▼
                        ┌──────────────┐
                        │  NIC 송신    │
                        └──────────────┘
```

각 hook의 역할:

1. **PREROUTING**: NIC에서 패킷을 받자마자, 라우팅 전에 실행. DNAT(목적지 NAT)이 여기서 일어남. k8s에서 Service ClusterIP를 Pod IP로 변환하는 규칙이 여기에 등록됨.
2. **INPUT**: 이 호스트가 최종 목적지인 패킷에 적용. 방화벽(DROP/ACCEPT) 판단.
3. **FORWARD**: 이 호스트를 경유하는 패킷에 적용. 라우터/게이트웨이 역할 시 사용.
4. **OUTPUT**: 로컬 앱이 보내는 패킷에 적용. Istio sidecar의 iptables REDIRECT가 여기서 동작.
5. **POSTROUTING**: 패킷이 인터페이스로 나가기 직전. SNAT/MASQUERADE가 여기서 일어남. 공유기의 사설→공인 IP 변환이 이 지점.

### iptables와 netfilter의 관계

```
iptables (유저스페이스 CLI)
    │
    │ netlink 소켓으로 커널에 규칙 전달
    ▼
netfilter (커널 프레임워크)
    ├─ PREROUTING hook: 등록된 규칙 체크
    ├─ INPUT hook: 등록된 규칙 체크
    ├─ FORWARD hook: 등록된 규칙 체크
    ├─ OUTPUT hook: 등록된 규칙 체크
    └─ POSTROUTING hook: 등록된 규칙 체크
```

- **netfilter** = 커널 안의 프레임워크. 유저가 직접 건드릴 수 없음.
- **iptables** = netfilter hook에 규칙을 등록/조회하는 유저스페이스 CLI 도구.
- **nftables** = iptables의 후계자. 같은 netfilter hook을 사용하지만 문법이 개선됨.
- **Cilium eBPF** = netfilter를 우회하고, 커널의 더 낮은 레벨(XDP, tc)에 직접 프로그램을 부착. iptables 규칙이 수천 개일 때의 선형 검색 성능 문제를 피함.

규칙이 한번 등록되면 iptables 프로세스가 종료돼도 규칙은 커널에 남아있다. 커널이 직접 실행하기 때문이다.

### k8s/인프라 도구와 netfilter hook의 매핑

| 도구 | 사용하는 hook | 하는 일 |
|---|---|---|
| kube-proxy (iptables 모드) | PREROUTING (DNAT) | Service ClusterIP → Pod IP 변환 |
| kube-proxy (iptables 모드) | FORWARD | Pod 간 패킷 포워딩 허용 |
| Istio sidecar | OUTPUT (REDIRECT) | 앱의 모든 outbound 트래픽을 Envoy로 리다이렉트 |
| Tailscale MSS Clamping | FORWARD (mangle) | PMTUD 실패 시 TCP MSS 강제 축소 |
| NAT Gateway / 공유기 | POSTROUTING (MASQUERADE) | 사설 IP를 공인 IP로 변환 |
| Cilium | XDP / tc (netfilter 우회) | eBPF 프로그램으로 직접 패킷 처리 |

---

## 9. OSI 7계층과 패킷의 실제 여정

### `curl https://api.example.com` 실행 시 일어나는 모든 일

#### Step 1: L7 Application (Userspace)

curl 프로세스가 HTTP 요청 문자열을 생성한다. 아직 네트워크와는 무관한 순수한 바이트 데이터다. `socket()` 시스템 콜로 커널에 소켓 fd를 요청한다.

```
GET / HTTP/1.1
Host: api.example.com
Accept: */*
```

#### Step 2: L6-5 Session/Presentation (Kernel — socket layer)

TLS 핸드셰이크가 진행된다 (ClientHello → ServerHello → 인증서 교환 → 키 합의). HTTP 데이터가 AES-GCM으로 암호화되어 외부에서는 내용을 읽을 수 없다.

```
[TLS Record] 17 03 03 00 1A [encrypted payload]
```

#### Step 3: L4 Transport (Kernel — TCP stack)

커널의 TCP 스택이 암호화된 데이터에 TCP 헤더를 부착한다. 출발지 포트(임시), 목적지 포트(443), 시퀀스 번호, 체크섬이 포함된다.

```
[TCP] Src:52341 → Dst:443 Seq:1001 Ack:1 Flags:PSH,ACK
[TLS encrypted payload]
```

#### Step 4: L3 Network (Kernel — IP layer)

커널이 IP 헤더를 부착하고 라우팅 테이블을 조회한다. 목적지가 로컬 서브넷이 아니면 default gateway로, 출구 인터페이스가 en0으로 결정된다. Tailscale 대역(100.x.x.x)이면 utun이 선택된다.

```
[IP] Src:192.168.20.25 → Dst:93.184.216.34 TTL:64 Proto:TCP
[TCP header + TLS payload]
```

#### Step 5: L3 (Kernel — netfilter OUTPUT hook)

패킷이 IP layer를 통과하는 도중에 netfilter OUTPUT hook을 만난다. 등록된 iptables/nftables 규칙을 순서대로 체크한다. DROP이면 폐기, ACCEPT이면 통과.

#### Step 6: L3 (Kernel — netfilter POSTROUTING hook)

인터페이스로 나가기 직전 마지막 검문소. SNAT/MASQUERADE 규칙이 있으면 적용. 커널 레벨 NAT이 없으면 공유기가 이 역할을 대신한다.

#### Step 7: L2 Data Link (Kernel — device subsystem)

IP 패킷을 이더넷 프레임으로 감싼다. ARP 캐시에서 게이트웨이의 MAC 주소를 조회하고 (없으면 ARP 요청 브로드캐스트), 이더넷 헤더 + FCS를 부착한다.

```
[Ethernet] Dst:aa:bb:cc:dd:ee:ff(GW) Src:96:60:2d:e2:2f:03(en0) Type:0x0800
[IP header][TCP header][TLS payload]
[FCS: 4 bytes CRC-32]
```

#### Step 8: L2 (Kernel — NIC driver + offload)

en0의 NIC 드라이버가 프레임을 NIC 하드웨어로 전달한다. TSO가 동작하면 NIC가 큰 세그먼트를 MTU(1500) 크기로 분할해서 전송. TXCSUM이 켜져있으면 체크섬도 NIC가 계산한다.

#### Step 9: L1 Physical

NIC가 이더넷 프레임의 비트들을 Wi-Fi 신호(2.4GHz/5GHz 전파)로 변환한다. 802.11 프로토콜에 따라 PLCP preamble 부착, OFDM 변조를 거쳐 안테나에서 전파를 방사한다.

#### Step 10: 물리 네트워크 경로

```
Mac (192.168.20.25)
  → 공유기 (NAT: 사설IP → 공인IP)
    → ISP 라우터 (BGP 라우팅)
      → IX (Internet Exchange)
        → 목적지 ISP
          → 서버 NIC
            → 서버 커널 (L1 → L7 역순 처리)
              → nginx / 서버 앱
```

응답은 이 전체 경로를 역순으로 되돌아온다.

---

## 10. 라우팅과 인터페이스 확인 명령어

### macOS

```bash
# 라우팅 테이블
netstat -rn

# 특정 목적지 경로
route get 10.0.0.5

# 인터페이스 목록
ifconfig

# pf 방화벽 규칙
sudo pfctl -s rules
sudo pfctl -s nat
sudo pfctl -s state
```

### Linux

```bash
# 라우팅 테이블
ip route show
ip route get 10.0.0.5

# 모든 라우팅 테이블 (policy routing 포함)
ip rule show
ip route show table all

# iptables 규칙 (테이블별)
sudo iptables -L -v -n --line-numbers
sudo iptables -t nat -L -v -n
sudo iptables -t mangle -L -v -n
sudo iptables-save

# nftables
sudo nft list ruleset

# k8s 관련 규칙만 필터링
sudo iptables-save | grep -i "KUBE\|FLANNEL\|CALICO"

# conntrack (NAT 매핑 상태)
sudo conntrack -L -d 10.96.0.1

# 실시간 패킷 추적
sudo iptables -t raw -A PREROUTING -s 192.168.1.100 -j TRACE
dmesg -w | grep TRACE
```

---

## 부록: 핵심 개념 요약

| 개념 | 한 줄 정의 |
|---|---|
| 인터페이스 | 커널이 관리하는 네트워크 출입구. 물리/가상 무관, 동일한 추상화 |
| 이더넷 | L1+L2 표준. MAC 주소 기반 프레임으로 데이터 전송 |
| TUN | 커널이 만드는 가상 NIC. 유저스페이스 앱에 패킷 전달 |
| TAP | TUN의 L2 버전. 이더넷 프레임 전체를 전달 |
| Bridge | 여러 인터페이스를 하나의 L2 브로드캐스트 도메인으로 묶는 가상 스위치 |
| Netfilter | 리눅스 커널 IP layer의 hook 프레임워크. 패킷이 지나갈 때 규칙 실행 |
| iptables | netfilter hook에 규칙을 등록하는 유저스페이스 CLI |
| MTU | Maximum Transmission Unit. 한 프레임에 담을 수 있는 최대 페이로드. 이더넷 표준 1500 bytes |
| PMTUD | Path MTU Discovery. 경로상의 최소 MTU를 찾는 메커니즘. ICMP "Fragmentation Needed"가 차단되면 Black Hole 발생 |
| MSS | Maximum Segment Size. TCP 세그먼트의 최대 데이터 크기. MTU - IP header(20) - TCP header(20) = 1460 |