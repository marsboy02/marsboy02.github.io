# 네트워크 심화 — 라우팅 테이블과 Netfilter(iptables)

## 1. 라우팅 테이블과 iptables의 관계

커널의 L3(IP) 계층에는 두 가지 핵심 시스템이 존재한다. 둘 다 패킷 처리에 관여하지만 역할이 완전히 다르다.

- **라우팅 테이블**: "이 패킷을 **어디로** 보낼까?" — 경로를 결정
- **iptables / Netfilter**: "이 패킷을 **어떻게 처리**할까?" — 허용, 차단, 변조를 결정

| 구분 | 라우팅 테이블 | iptables / Netfilter |
|------|-------------|---------------------|
| 핵심 질문 | "어디로 보낼까?" | "보내도 될까? 수정할까?" |
| 판단 기준 | 목적지 IP | 출발지/목적지 IP, 포트, 프로토콜, 상태 등 |
| 동작 | 경로 선택 (인터페이스, 게이트웨이) | 허용/차단/주소변환/패킷수정 |
| 실행 횟수 | 패킷당 1~2회 (수신 시, 송신 시) | 패킷당 여러 훅을 거치며 다수의 규칙 체크 |

둘은 독립적인 시스템이지만 밀접하게 연동된다. Netfilter가 패킷을 변조하면 라우팅 판단이 달라지고, 라우팅 판단 결과에 따라 거치는 Netfilter 훅이 달라진다.

---

## 2. 라우팅 테이블 (Routing Table)

### 2.1 개념

라우팅 테이블은 커널이 관리하는 **경로 정보 데이터베이스**다. 패킷의 목적지 IP 주소를 보고, "이 패킷은 어떤 네트워크 인터페이스로, 어떤 다음 홉(next hop)으로 보내야 하는가?"를 결정한다.

커널의 핵심 알고리즘은 **Longest Prefix Match(LPM)** — 목적지 IP에 가장 구체적으로 일치하는 경로를 선택한다.

### 2.2 라우팅 테이블의 핵심 요소

- **Destination**: 목적지 네트워크 (CIDR 표기)
- **Gateway (via)**: 다음 홉 주소 (직접 연결이면 없음)
- **Device (dev / Netif)**: 나갈 네트워크 인터페이스
- **Metric**: 같은 목적지에 대해 여러 경로가 있을 때 우선순위
- **Scope**: link(로컬 서브넷), global(원격) 등

### 2.3 라우팅 테이블 확인 명령어

**Linux:**

```bash
ip route show
```

```
default via 192.168.1.1 dev eth0          # 기본 게이트웨이
192.168.1.0/24 dev eth0 scope link        # 로컬 서브넷은 eth0으로 직접 전달
10.0.0.0/8 via 10.1.1.1 dev wg0          # 10.x 대역은 WireGuard 터널로
```

**macOS (BSD 계열):**

macOS는 `iproute2` 패키지가 없으므로 BSD 계열 명령어를 사용한다.

```bash
netstat -rn                    # 라우팅 테이블 전체 조회
route -n get default           # 기본 게이트웨이 상세 정보
route -n get 10.0.5.3          # 특정 IP로의 경로 조회
```

### 2.4 macOS 라우팅 테이블 출력 해석

```
Destination        Gateway            Flags               Netif Expire
default            192.168.20.1       UGScg                 en0
127                127.0.0.1          UCS                   lo0
127.0.0.1          127.0.0.1          UH                    lo0
169.254            link#14            UCS                   en0      !
192.168.20/23      link#14            UCS                   en0      !
192.168.20.25/32   link#14            UCS                   en0      !
192.168.20.1       ac:71:2e:f:ad:88   UHLWIir               en0   1198
192.168.20.7       56:ad:d5:2f:95:28  UHLWI                 en0   1170
192.168.21.255     ff:ff:ff:ff:ff:ff  UHLWbI                en0      !
224.0.0/4          link#14            UmCS                  en0      !
255.255.255.255/32 link#14            UCS                   en0      !
```

#### 구역별 의미

**① 기본 경로 (Default Route)**

`default → 192.168.20.1` — 라우팅 테이블에서 매칭되는 경로가 없으면 무조건 게이트웨이(공유기)로 보낸다. 인터넷으로 나가는 모든 트래픽이 여기로 빠진다.

**② 루프백 (Loopback)**

`127.0.0.0/8` 대역 — localhost로 가는 트래픽은 네트워크 밖으로 나가지 않고 `lo0`에서 자기 자신에게 돌아온다. `localhost:3000` 같은 개발 서버가 동작하는 이유.

**③ Link-Local (169.254)**

DHCP에서 IP를 못 받았을 때 자동 할당되는 대역(APIPA). `!` 표시는 현재 비활성(inactive) 상태를 의미한다. `169.254.169.254`는 클라우드 환경(AWS 등)에서 메타데이터 서버 주소로 사용된다.

**④ 로컬 서브넷과 자기 자신**

`192.168.20/23` — 이 서브넷(192.168.20.0~192.168.21.255)은 게이트웨이 없이 `en0`에서 직접 통신 가능하다는 의미. `link#14`는 en0의 링크 인덱스. `192.168.20.25/32`는 이 Mac 자신의 IP.

**⑤ ARP 캐시 연동 호스트 경로**

`192.168.20.7 → MAC주소` — 같은 서브넷의 실제 기기들. macOS는 ARP로 알게 된 호스트를 라우팅 테이블에 호스트 경로(/32)로 추가한다. Expire 숫자는 ARP 캐시 만료까지 남은 초(약 1200초 기준 카운트다운).

**⑥ 브로드캐스트**

`192.168.21.255 → ff:ff:ff:ff:ff:ff` — /23 서브넷의 브로드캐스트 주소. `255.255.255.255`는 제한된 브로드캐스트(limited broadcast).

**⑦ 멀티캐스트**

`224.0.0.0/4` — 멀티캐스트 대역 전체(224~239). `224.0.0.251`은 mDNS(Bonjour — AirDrop, AirPlay, 프린터 자동 검색). `239.255.255.250`은 SSDP(UPnP 기기 검색).

#### Flags 의미

| Flag | 의미 |
|------|------|
| U | Up (경로 활성) |
| G | Gateway 경유 (직접 연결이 아님) |
| H | Host 경로 (/32, 특정 호스트 하나) |
| S | Static (수동 또는 시스템 설정) |
| C | Clone (새 호스트 경로 생성 가능) |
| L | Link-layer 주소 있음 (MAC 확인됨) |
| W | Was cloned (C 경로에서 복제됨) |
| I | ifref (인터페이스 참조) |
| m | Multicast |
| b | Broadcast |
| ! (Expire 열) | 비활성 또는 만료된 경로 |

### 2.5 Policy Routing

Linux에는 여러 라우팅 테이블을 가질 수 있다. `ip rule` 명령으로 "출발지 IP가 X이면 테이블 100번을 참조해라" 같은 규칙을 만들 수 있다.

### 2.6 패킷 흐름 예시

라우팅 테이블 기준으로 실제 패킷 처리 흐름:

- **로컬 서브넷 통신** (`192.168.20.7`로 ping): 서브넷 매칭 → 게이트웨이 없이 en0에서 직접 전달 → ARP 캐시에서 MAC 확인 → 이더넷 프레임 전송
- **인터넷 통신** (`8.8.8.8`로 ping): 서브넷 매칭 없음 → default 경로 매칭 → 게이트웨이 `192.168.20.1`로 전달 → 공유기가 인터넷으로 라우팅

---

## 3. Netfilter와 iptables

### 3.1 iptables의 역할

iptables는 커널의 Netfilter에게 "이런 패킷이 오면 이렇게 처리해"라고 **규칙을 등록해주는 사용자 도구(userspace tool)**. 실제 패킷 처리는 커널 안의 Netfilter가 수행한다.

---

## 4. 훅 (Hook) — Netfilter의 5개 체크포인트

### 4.1 훅의 개념

훅(Hook)이란 "어떤 처리 흐름의 특정 지점에 끼어들 수 있는 진입점"이다. Netfilter는 커널 네트워크 스택의 패킷 처리 흐름 중간중간에 **"여기서 잠깐, 등록된 규칙이 있으면 실행해"** 하는 체크포인트를 5개 만들어 두었다.

### 4.2 전체 패킷 흐름과 훅의 위치

커널 입장에서 패킷이 도착하면 반드시 하나의 질문을 한다: **"이 패킷의 목적지가 나인가, 아닌가?"** 이 질문의 답에 따라 경로가 갈리고, 5개의 훅은 이 경로 위의 서로 다른 지점에 배치되어 있다.

```
                        ┌─────────────────────────────┐
                        │        이 호스트 (커널)        │
                        │                             │
                        │    ┌───────────────────┐    │
                        │    │   로컬 프로세스      │    │
                        │    │ (nginx, ssh, app…) │    │
                        │    └──────▲──────┬──────┘    │
                        │           │      │           │
                        │       ③ INPUT  ⑤ OUTPUT     │
                        │           │      │           │
  ──── 패킷 도착 ──▶  ① PREROUTING  │      │           │
                        │           │      ▼           │
                        │       ② 라우팅   라우팅        │
                        │        판단      판단         │
                        │           │      │           │
                        │       ④ FORWARD  │           │
                        │           │      │           │
                        │           ▼      ▼           │
                        │       ⑥ POSTROUTING          │
                        │           │                  │
                        └───────────┼──────────────────┘
                                    │
                                    ▼
                              패킷 송신 ──▶
```

### 4.3 각 훅의 상세 설명

#### ① PREROUTING — "패킷이 막 도착했을 때, 라우팅 판단 전"

패킷이 NIC를 통해 커널에 들어오자마자 **가장 먼저** 거치는 훅. 아직 이 패킷이 나한테 온 건지, 다른 곳으로 가는 건지 판단하기 **전**이다.

핵심 이유: **여기서 목적지 주소를 바꾸면(DNAT) 뒤의 라우팅 판단 결과가 달라진다.**

```bash
# 외부 8080 → 내부 192.168.1.10:80으로 DNAT
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 192.168.1.10:80
```

라우팅 판단 전에 목적지를 바꿔야 커널이 변경된 주소 기준으로 올바르게 경로를 결정한다.

#### ② 라우팅 판단 (Routing Decision) — 핵심 분기점

Netfilter 훅이 아니라 커널 IP 스택의 동작이지만 흐름 이해에 필수적이다. 커널이 목적지 IP를 보고 라우팅 테이블을 조회하여 두 갈래로 나눈다.

- **목적지가 이 호스트의 IP** → INPUT 경로
- **목적지가 다른 IP** → FORWARD 경로

#### ③ INPUT — "이 호스트로 향하는 패킷이 로컬 프로세스에 전달되기 직전"

라우팅 판단 결과 "이 패킷은 나한테 온 것"이라고 결정된 후, 로컬 프로세스에 넘겨주기 직전에 거치는 훅. **서버 방화벽의 핵심**.

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT     # SSH 허용
iptables -A INPUT -p tcp --dport 80 -j ACCEPT     # HTTP 허용
iptables -A INPUT -j DROP                          # 나머지 차단
```

INPUT은 **이 호스트가 최종 목적지인 패킷**에만 적용된다. 경유(포워딩)하는 패킷은 INPUT을 거치지 않는다.

**INPUT 통과 후 프로세스 전달까지의 두 단계:**

1. **Netfilter(INPUT)**: "이 포트로 오는 패킷을 통과시킬 것인가?" → iptables 규칙으로 결정
2. **커널 소켓 계층**: "이 포트를 listen하고 있는 프로세스가 있는가?" → 있으면 전달, 없으면 RST 응답

이 두 단계는 독립적이다. iptables에서 80번을 ACCEPT했더라도 nginx가 안 떠 있으면 RST가 발생하고, nginx가 떠 있어도 iptables에서 DROP하면 패킷이 도달하지 못한다.

#### ④ FORWARD — "이 호스트를 경유하는 패킷"

라우팅 판단 결과 "이 패킷은 나한테 온 게 아니라 다른 곳으로 가는 것"인 경우에 거치는 훅. 이 호스트가 **라우터 역할**을 할 때만 의미가 있다.

중요한 경우: Linux 라우터/게이트웨이, Docker 컨테이너 네트워킹, VPN 서버

```bash
iptables -A FORWARD -s 172.17.0.0/16 -j ACCEPT
```

Linux에서는 기본적으로 IP 포워딩이 꺼져 있다. `sysctl net.ipv4.ip_forward=1`로 활성화 필요.

#### ⑤ OUTPUT — "이 호스트에서 생성된 패킷이 나가기 직전"

로컬 프로세스가 만든 패킷이 커널 네트워크 스택으로 내려온 직후에 거치는 훅. PREROUTING의 반대편.

```bash
# 외부 SMTP(25번 포트) 발신 차단
iptables -A OUTPUT -p tcp --dport 25 -j DROP
```

OUTPUT 훅 뒤에도 라우팅 판단이 한 번 더 있다 — "내가 만든 이 패킷을 어떤 인터페이스로 보낼까" 결정.

#### ⑥ POSTROUTING — "패킷이 최종적으로 NIC를 통해 나가기 직전"

밖으로 나가는 **모든 패킷**(INPUT/OUTPUT 경로든 FORWARD 경로든)이 마지막으로 거치는 훅. 대표적 동작은 SNAT/MASQUERADE.

```bash
# 내부 네트워크 출발지를 공인 IP로 변환
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

모든 판단(라우팅, 필터링)이 끝난 뒤에 출발지를 바꿔야 하기 때문에 이 위치에 있다.

### 4.4 각 훅이 해당 위치에 있는 이유

| 훅 | 위치 | 핵심 질문 | 이 위치인 이유 |
|---|---|---|---|
| PREROUTING | 수신 직후, 라우팅 전 | 목적지를 바꿀까? | 라우팅 판단에 영향을 줘야 하니까 |
| INPUT | 라우팅 후, 프로세스 전달 전 | 이 접속을 허용할까? | 프로세스에 도달하기 전에 차단해야 하니까 |
| FORWARD | 라우팅 후, 송신 전 | 이 경유를 허용할까? | 남의 패킷을 무조건 통과시키면 안 되니까 |
| OUTPUT | 프로세스 생성 후, 송신 전 | 이 발신을 허용할까? | 나가면 안 되는 트래픽을 잡아야 하니까 |
| POSTROUTING | 모든 처리 후, NIC 직전 | 출발지를 바꿀까? | 모든 판단이 끝난 뒤에 주소를 변환해야 하니까 |

### 4.5 Netfilter 훅 사이에서 라우팅 테이블과의 연동

외부에서 들어오는 패킷 기준 전체 순서:

```
패킷 수신 (NIC → 드라이버 → 커널)
    │
    ▼
① Netfilter: raw → conntrack → mangle → nat (PREROUTING)
    │         ← 여기서 DNAT 하면 목적지 IP가 바뀜
    ▼
② 라우팅 판단 (Routing Decision)
    │         ← 바뀐 목적지 IP 기준으로 판단
    │
    ├── 나한테 온 패킷 → ③ Netfilter: mangle → filter (INPUT) → 로컬 프로세스
    │
    └── 남한테 갈 패킷 → ③ Netfilter: mangle → filter (FORWARD)
                              │
                              ▼
                         ④ Netfilter: mangle → nat (POSTROUTING)
                              │         ← 여기서 SNAT/MASQUERADE
                              ▼
                         패킷 송신
```

**PREROUTING에서 DNAT를 하면 라우팅 판단에 영향을 준다.** 예를 들어 외부에서 `공인IP:8080`으로 온 패킷을 DNAT로 `192.168.1.10:80`으로 바꾸면, 라우팅 테이블은 변경된 `192.168.1.10`을 기준으로 "이건 내부 네트워크니까 FORWARD하자"고 판단한다.

---

## 5. iptables의 구조: 테이블, 체인, 규칙

### 5.1 계층 구조 개요

iptables는 **테이블(Table) > 체인(Chain) > 규칙(Rule)** 의 3단 계층 구조를 가진다.

- **테이블**: "어떤 종류의 작업인가" — 필터링, NAT, 패킷 수정 등
- **체인**: "특정 훅에서, 특정 테이블의 규칙들을 순서대로 담아둔 목록"
- **규칙**: "이 조건에 맞으면 이 동작을 해라" — 하나의 판단 단위

### 5.2 테이블 (Table)

하나의 훅(체크포인트)에서 성격이 다른 여러 작업을 분리하기 위해 테이블이 존재한다.

#### filter — "통과시킬까, 차단할까"

가장 기본이자 가장 많이 쓰는 테이블. `-t` 옵션 생략 시 기본 선택. 패킷을 **허용하거나 차단하는 것**만 담당한다.

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # SSH 허용
iptables -A INPUT -j DROP                         # 나머지 차단
```

타겟: ACCEPT, DROP, REJECT

체인: INPUT, FORWARD, OUTPUT

#### nat — "주소를 바꿀까"

패킷의 **출발지 또는 목적지 IP/포트를 변환**하는 테이블. 공유기 NAT, 포트 포워딩, Docker 컨테이너 네트워킹 등이 여기서 이루어진다.

```bash
# DNAT (목적지 변환) — PREROUTING
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 192.168.1.10:80

# MASQUERADE (출발지 변환) — POSTROUTING
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

중요 특성: nat 테이블은 **연결의 첫 번째 패킷에만 적용**된다. 이후 같은 연결의 패킷은 conntrack이 자동으로 같은 변환을 적용한다.

체인: PREROUTING, OUTPUT, POSTROUTING

#### mangle — "패킷 헤더를 수정할까"

패킷의 **IP 헤더 필드를 직접 수정**하는 테이블.

```bash
# TTL 변경
iptables -t mangle -A POSTROUTING -j TTL --ttl-set 65

# TOS(Type of Service) 변경 — QoS
iptables -t mangle -A PREROUTING -p tcp --dport 22 -j TOS --set-tos Minimize-Delay

# MARK — 커널 내부 태그 설정 (Policy Routing 연동)
iptables -t mangle -A PREROUTING -s 192.168.1.100 -j MARK --set-mark 10
```

MARK는 실제 패킷 헤더를 바꾸는 게 아니라 **커널 내부에서만 유효한 태그**를 붙이는 것. 이 태그로 Policy Routing 등을 할 수 있다.

체인: 5개 훅 전부 (PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING)

#### raw — "연결 추적 자체를 건너뛸까"

Netfilter는 기본적으로 모든 패킷에 **conntrack(연결 추적)** 을 수행한다. raw 테이블은 특정 패킷을 conntrack 추적 대상에서 제외한다. 대규모 트래픽 서버에서 conntrack 테이블 오버플로우 방지용.

```bash
# DNS 트래픽은 conntrack 추적하지 않음
iptables -t raw -A PREROUTING -p udp --dport 53 -j NOTRACK
```

다른 모든 테이블보다 **가장 먼저** 실행된다 — conntrack 추적 시작 전에 결정해야 하기 때문.

체인: PREROUTING, OUTPUT

### 5.3 테이블별 체인 매핑

```
              PREROUTING    INPUT    FORWARD    OUTPUT    POSTROUTING
 raw             ✅                               ✅
 mangle          ✅           ✅        ✅         ✅          ✅
 nat             ✅           ✅                   ✅          ✅
 filter                      ✅        ✅         ✅
```

하나의 훅에서 여러 테이블의 체인이 **고정된 순서(raw → mangle → nat → filter)** 로 실행된다.

### 5.4 체인 (Chain)

체인은 **"특정 훅에서, 특정 테이블의 규칙들을 순서대로 담아둔 목록"** 이다.

예: "PREROUTING 훅에서 nat 테이블이 실행하는 규칙 목록" = **nat 테이블의 PREROUTING 체인**

체인 이름이 훅 이름과 동일하지만, 체인은 항상 특정 테이블에 소속되어 있다.

#### 내장 체인 vs 사용자 정의 체인

내장(built-in) 체인: INPUT, FORWARD, OUTPUT, PREROUTING, POSTROUTING

사용자 정의 체인: 규칙이 많아질 때 정리용. 프로그래밍의 함수 호출과 유사.

```bash
# 사용자 정의 체인 생성 및 사용
iptables -N WEB_TRAFFIC
iptables -A WEB_TRAFFIC -p tcp --dport 80 -j ACCEPT
iptables -A WEB_TRAFFIC -p tcp --dport 443 -j ACCEPT
iptables -A WEB_TRAFFIC -j DROP

# INPUT에서 이 체인으로 분기
iptables -A INPUT -p tcp -j WEB_TRAFFIC
```

### 5.5 규칙 (Rule)

규칙은 **"이 조건에 맞으면 이 동작을 해라"** 라는 하나의 명령문. **매칭 조건(Match)** 과 **타겟(Target)** 으로 구성된다.

```bash
iptables -A INPUT -p tcp -s 10.0.0.0/8 --dport 22 -j ACCEPT
#        ─────── ────── ───────────── ─────────── ──────────
#        체인지정  매칭조건들                        타겟
```

#### 주요 매칭 조건

```bash
-p tcp/udp/icmp               # 프로토콜
-s 192.168.1.0/24             # 출발지 IP/네트워크
-d 10.0.0.5                   # 목적지 IP
--sport 1024                   # 출발지 포트
--dport 80                     # 목적지 포트
-i eth0                        # 들어오는 인터페이스
-o eth1                        # 나가는 인터페이스
-m conntrack --ctstate ESTABLISHED   # 연결 상태 (확장 매칭)
-m multiport --dports 80,443,8080   # 여러 포트 (확장 매칭)
```

#### 타겟 종류

**종료 타겟 (Terminating)** — 판단이 끝나고 다음 규칙으로 넘어가지 않음:

| 타겟 | 동작 |
|---|---|
| ACCEPT | 패킷 통과 |
| DROP | 패킷 무응답 폐기 (상대방은 타임아웃까지 기다림) |
| REJECT | 패킷 거부 + ICMP 에러 응답 전송 |
| DNAT | 목적지 주소 변환 |
| SNAT | 출발지 주소 변환 |
| MASQUERADE | 출발지를 나가는 인터페이스 IP로 변환 |

**비종료 타겟 (Non-terminating)** — 처리 후 다음 규칙도 계속 평가:

| 타겟 | 동작 |
|---|---|
| LOG | 커널 로그에 기록하고 다음 규칙으로 계속 |
| MARK | 내부 마크 설정하고 다음 규칙으로 계속 |

```bash
# LOG(비종료) 후 DROP(종료) — 로그 기록 뒤 차단
iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "SSH attempt: "
iptables -A INPUT -p tcp --dport 22 -s !10.0.0.0/8 -j DROP
```

#### 규칙 평가 순서

체인 안의 규칙은 **위에서 아래로 순서대로** 평가. 종료 타겟에 매칭되면 즉시 결정, 나머지 규칙은 보지 않는다.

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # 규칙 1: 매칭 → 통과
iptables -A INPUT -p tcp --dport 22 -j DROP       # 규칙 2: 절대 도달 불가!
```

어떤 규칙에도 매칭되지 않으면 체인의 **기본 정책(Policy)** 적용:

```bash
iptables -P INPUT DROP    # INPUT 체인 기본 정책을 DROP으로
```

### 5.6 전체 구조를 하나로

```
패킷 도착 → PREROUTING 훅
              │
              ├─ raw 테이블의 PREROUTING 체인    [규칙1] → [규칙2] → ...
              ├─ mangle 테이블의 PREROUTING 체인  [규칙1] → [규칙2] → ...
              └─ nat 테이블의 PREROUTING 체인     [규칙1] → [규칙2] → ...
              │
              ▼
           라우팅 판단
              │
              ▼ (나한테 온 패킷)
           INPUT 훅
              │
              ├─ mangle 테이블의 INPUT 체인       [규칙1] → [규칙2] → ...
              ├─ nat 테이블의 INPUT 체인          [규칙1] → [규칙2] → ...
              └─ filter 테이블의 INPUT 체인       [규칙1] → [규칙2] → ...
              │
              ▼
           로컬 프로세스
```

하나의 훅 안에서 테이블 순서(raw → mangle → nat → filter)는 커널이 정한 고정 순서이고, 하나의 체인 안에서 규칙 순서는 사용자가 `-A`(append), `-I`(insert)로 직접 관리한다.

---

## 6. AWS 보안 그룹과 Netfilter 비교

AWS 보안 그룹은 Netfilter의 일부 기능을 단순화한 것이다. Netfilter의 기능을 AWS가 여러 서비스로 분리하여 제공하고 있다.

| Netfilter | AWS 대응 |
|---|---|
| INPUT (filter) | 보안 그룹 인바운드 |
| OUTPUT (filter) | 보안 그룹 아웃바운드 |
| FORWARD (filter) | Network ACL (서브넷 레벨) |
| PREROUTING (DNAT) | Load Balancer, NAT Gateway |
| POSTROUTING (SNAT) | NAT Gateway, Internet Gateway |

보안 그룹은 **Stateful** — 인바운드에서 허용한 요청의 응답은 아웃바운드 규칙과 상관없이 자동 허용된다. 이는 Netfilter의 conntrack(연결 추적) 모듈과 같은 개념이다.

---

## 7. 방화벽 관점에서의 각 훅의 역할

대부분의 서버는 라우터가 아니라 최종 목적지이므로, INPUT 체인이 서버 방화벽의 핵심이다.

| 훅 | 방화벽 역할 | 중요한 경우 |
|---|---|---|
| INPUT | 이 호스트를 보호 | 일반 서버 (가장 흔한 케이스) |
| FORWARD | 이 호스트 뒤의 네트워크를 보호 | 라우터, Docker, VPN 서버 |
| OUTPUT | 이 호스트의 발신 트래픽 제한 | 보안이 엄격한 환경 (금융, 군사 등) |
| PREROUTING / POSTROUTING | 주소 변환 (NAT) 담당 | 방화벽이라기보다 NAT 역할 |

### 실전 서버 방화벽 설정 패턴 (INPUT 중심)

```bash
# 기본 정책: 모든 인바운드 차단
iptables -P INPUT DROP

# 이미 연결된 세션의 패킷은 허용 (stateful)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 루프백은 허용
iptables -A INPUT -i lo -j ACCEPT

# SSH, HTTP, HTTPS만 허용
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 나머지는 기본 정책(DROP)에 의해 차단됨
```

이는 AWS 보안 그룹에서 인바운드 규칙으로 "SSH 22번 허용, HTTP 80번 허용" 하는 것과 본질적으로 동일하다.

---

## 8. macOS 추가 네트워크 명령어

macOS는 Linux의 iproute2가 없으므로 BSD 계열 명령어를 사용한다.

```bash
# 라우팅
netstat -rn                              # 라우팅 테이블 전체
route -n get default                     # 기본 게이트웨이 상세
route -n get <IP>                        # 특정 IP 경로 조회

# 인터페이스
ifconfig                                 # 인터페이스 목록 및 IP (Linux의 ip addr)

# ARP / NDP
arp -a                                   # ARP 테이블 (IP ↔ MAC 매핑)
ndp -a                                   # IPv6 Neighbor Discovery 테이블

# DNS / 네트워크 설정
scutil --dns                             # DNS 설정 확인
networksetup -listallhardwareports       # 물리 인터페이스 목록

# 방화벽 (PF — macOS는 Netfilter 대신 PF 사용)
sudo pfctl -sr                           # 현재 활성 방화벽 규칙
sudo pfctl -sa                           # 전체 상태 (규칙 + 통계 + 테이블)
sudo pfctl -si                           # 인터페이스별 통계
# PF 설정 파일: /etc/pf.conf
```

macOS PF와 Linux iptables 문법 비교:

```bash
# Linux iptables: 80번 포트 차단
iptables -A INPUT -p tcp --dport 80 -j DROP

# macOS PF: 동일 동작 (/etc/pf.conf)
block in proto tcp from any to any port 80
```