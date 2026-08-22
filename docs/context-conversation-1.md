# Context: MTU와 WireGuard 캡슐화 문제

> 이 문서는 블로그 초안 작성을 위한 원본 자료입니다.
> 코딩 에이전트(Claude Code)가 블로그 글을 작성할 때 참조할 context로 사용됩니다.
> 블로그 톤은 https://www.marsboy.online/ko/posts/ 를 참조합니다.

---

## 1. MTU 기본 개념

### 정의

- MTU(Maximum Transmission Unit): 네트워크 인터페이스가 한 번에 전송할 수 있는 최대 패킷 크기
- 정확히는 L2 프레임의 페이로드, 즉 **L3(IP) 패킷의 최대 크기**를 의미
- 이더넷 표준 기본 MTU: **1500바이트** (1980년대 이더넷 규격에서 정해진 값)

### 패킷 구조

```
[Ethernet Frame Header 14B] [IP Header 20B] [TCP Header 20B] [Payload ≤1460B] [FCS 4B]
                             |<------------- MTU 1500B ------------------>|
```

- Ethernet Frame Header(14B)와 FCS(4B)는 MTU 계산에 포함되지 않음
- MTU는 L3 이상의 크기만 카운트

### OSI 레이어와 데이터 단위

- L4(Transport): **Segment**
- L3(Network): **Packet**
- L2(Data Link): **Frame**

### 캡슐화 방향

```
L2 Frame이 가장 바깥 (택배 박스)
  └─ L3 Packet이 그 안 (포장된 상품)
       └─ L4 Segment가 그 안 (상품 본체)
            └─ L7 Data가 가장 안쪽 (내용물)
```

- **아래 레이어가 위 레이어를 감쌈** (L2 → L3 → L4 → L7)
- MTU는 L2 프레임이라는 택배 박스의 최대 적재량

### 비유

- 택배 박스(L2 Frame) 크기가 정해져 있으니, 내용물(L3 이상)이 그 안에 들어맞아야 한다

---

## 2. MSS (Maximum Segment Size)

### 정의

- MSS: 한 TCP 세그먼트에 담을 수 있는 최대 애플리케이션 데이터 크기

### 계산

```
MSS = MTU - IP Header - TCP Header
MSS = 1500 - 20 - 20 = 1460 바이트
```

### 동작 방식

- TCP는 연결 수립(3-way handshake) 시 **인터페이스의 MTU를 보고 MSS를 협상**
- L7에서 10KB 데이터를 보내달라고 하면, L4(TCP)에서 MSS 1460B 단위로 쪼개서 전송

### 데이터 흐름 예시

```
L7 Application:  "여기 데이터 10KB 보내줘"
        ↓
L4 TCP:          "인터페이스 MTU가 1500이네. MSS 1460으로 쪼개서 보내야지"
                  → 10KB를 1460B짜리 세그먼트 7개로 분할
        ↓
L3 IP:           각 세그먼트에 IP 헤더(20B) 붙임 → 1480B 패킷
        ↓
L2 Ethernet:     각 패킷에 프레임 헤더(14B) + FCS(4B) 붙여서 전송
                  "내 MTU(1500) 이하니까 OK"
```

---

## 3. Fragmentation과 DF 비트

### MTU 초과 시 두 가지 시나리오

**시나리오 1: Fragmentation (단편화)**
- IP 헤더의 DF(Don't Fragment) 비트가 **꺼져 있으면**, 라우터가 패킷을 MTU에 맞게 쪼개서 전송
- 수신 측에서 재조립
- 비용: 성능 저하, 패킷 유실 시 전체 재전송 필요

**시나리오 2: 패킷 드롭 + ICMP 에러**
- DF 비트가 **켜져 있으면** (요즘 대부분의 TCP 패킷), 라우터가 "쪼갤 수 없음" 판단
- 패킷 드롭 + ICMP "Fragmentation Needed" 메시지 반환
- 송신자가 패킷 크기를 줄여서 재전송 → **Path MTU Discovery (PMTUD)** 메커니즘

### TCP에서 DF가 기본 설정인 이유

- PMTUD 메커니즘이 DF 비트를 전제로 동작
- Fragmentation의 성능 비용(재조립 오버헤드, 유실 시 전체 재전송)을 피하기 위함

---

## 4. Path MTU Discovery (PMTUD)

### 동작 원리

1. 송신자가 DF 비트를 설정한 패킷을 전송
2. 경로상의 라우터가 MTU 초과를 감지하면 패킷 드롭
3. 라우터가 ICMP "Fragmentation Needed" 메시지를 송신자에게 반환 (허용 가능한 MTU 크기 포함)
4. 송신자가 패킷 크기를 줄여서 재전송
5. 경로의 최소 MTU를 찾을 때까지 반복

### ICMP "Fragmentation Needed" 메시지의 역할

- 송신자에게 "이 경로에서 허용되는 최대 패킷 크기"를 알려주는 피드백 메커니즘
- 이 메시지가 차단되면 PMTUD가 실패함

---

## 5. PMTUD Black Hole

### 정의

- ICMP "Fragmentation Needed" 패킷이 방화벽에서 차단되어 PMTUD가 동작하지 않는 상황

### 증상

- "연결은 되는데 데이터가 안 온다"
- 송신자가 패킷 드롭 이유를 모르고 같은 크기로 계속 재시도 → timeout

### AWS 환경에서 흔한 이유

- AWS Security Group이나 네트워크 방화벽에서 ICMP를 차단하는 경우가 많음
- VPN/터널링 환경에서 ICMP가 경로상 어딘가에서 드롭되기 쉬움
- 클라우드 환경의 다층 네트워크 구조(VPC, 서브넷, 보안 그룹)가 ICMP 전달을 복잡하게 만듦

### 디버깅 명령어

```bash
# DF 비트를 켜고(-M do) 페이로드 크기를 지정(-s)해서 어디서 끊기는지 탐색
ping -M do -s 1400 <target>
```

```bash
# MSS clamping으로 TCP 세그먼트 크기를 강제로 줄이는 우회 방법
iptables -j TCPMSS --clamp-mss-to-pmtu
```

---

## 6. WireGuard/Tailscale의 이중 캡슐화 문제

### 배경

- Tailscale은 WireGuard 기반 VPN
- WireGuard는 **L3 VPN**: IP 패킷을 통째로 캡슐화
- WireGuard 오버헤드: 약 **60바이트** (Outer IP 20B + UDP 8B + WG Header 32B)

### 이중 캡슐화 단계별 과정

**1단계: 앱이 패킷을 만든다**
- Spring 앱(또는 Redis 클라이언트)은 터널의 존재를 모름
- 평소처럼 데이터 전송

```
[Inner IP 20B] [Inner TCP 20B] [Payload 1460B] = 1500B
```

**2단계: WireGuard(Tailscale)가 캡슐화**
- 원본 1500B 패킷을 "데이터"로 취급
- 바깥에 새로운 헤더를 씌움

```
[Outer IP 20B] [UDP 8B] [WG Header 32B] [원래 패킷 1500B] = 1560B
```

**3단계: 물리 NIC 통과 시도**
- 물리 NIC MTU: 1500B
- 캡슐화된 패킷: 1560B → **MTU 초과 → 드롭**

### 캡슐화 전후 비교 (ASCII Diagram)

```
일반 패킷:
[IP 20B] [TCP 20B] [Payload 1460B] = 1500B   ← 물리 NIC MTU 이하 ✅

WireGuard 캡슐화 후:
[Outer IP 20B] [UDP 8B] [WG 32B] [Inner IP 20B] [TCP 20B] [Payload 1460B] = 1560B
                                                                             ← MTU 초과 ❌
```

### TCP가 알아서 조절 못 하는 이유

- TCP는 MSS 협상 시 **자기가 나가는 인터페이스(tailscale0)의 MTU**를 참조
- 터널 인터페이스 MTU가 1500으로 설정되어 있으면, TCP는 MSS 1460으로 보냄
- WireGuard가 60B를 추가하는 것은 TCP가 알 수 없는 영역
- **IP-in-UDP 캡슐화**: L3 패킷 안에 또 다른 L3 패킷이 들어있는 구조

### 실제 트러블슈팅: Redis 헬스체크 실패

```
Redis PING (작은 패킷):
  Inner: [IP+TCP+PING ≈ 50B] = 50B
  캡슐화 후: 50 + 60 = 110B
  물리 NIC MTU 1500 > 110B → ✅ 통과

Redis INFO 응답 (큰 패킷):
  Inner: [IP+TCP+INFO ≈ 1500B] = 1500B
  캡슐화 후: 1500 + 60 = 1560B
  물리 NIC MTU 1500 < 1560B → ❌ 드롭 → healthcheck timeout
```

- **"연결은 되는데 헬스체크만 실패한다"**: 작은 패킷(PING/PONG)은 캡슐화해도 MTU 이내, 큰 패킷(INFO 응답)만 초과
- 이 패턴이 **MTU 문제의 전형적 증상**

### 비유

- 이미 꽉 차게 포장된 택배를 국제배송용 박스에 다시 넣어야 하는데, 바깥 박스 크기 제한도 같아서 안 맞는 상황

---

## 7. 해결 원리

### 핵심: 터널 인터페이스 MTU를 미리 낮추기

- `tailscale0` 인터페이스의 MTU를 **1440** (또는 Tailscale 기본값 **1280**)으로 설정

### 동작 원리

```
터널 인터페이스 MTU를 1440으로 설정
  → TCP가 MSS를 1400으로 협상 (1440 - 20 - 20)
  → 앱이 애초에 1400B 이하의 페이로드로 세그먼트 생성
  → 캡슐화 후: 1440 + 60 = 1500B
  → 물리 NIC MTU 1500B → ✅ 딱 맞게 통과
```

### 적정 값 계산

```
터널 MTU = 물리 NIC MTU - WireGuard 오버헤드
터널 MTU = 1500 - 60 = 1440
```

- Tailscale은 보수적으로 **1280**을 기본값으로 설정하기도 함 (IPv6 최소 MTU)
- 1440이 이론적 최적값, 1280이 안전한 값

### OS에서 MTU 확인

```bash
ip link show
```

- 커널 TCP 스택이 인터페이스 MTU를 읽고 MSS를 협상하는 흐름

---

## 8. WireGuard 프로토콜 상세 (부록)

### WireGuard는 L3 VPN

- **L2 VPN** (OpenVPN TAP, VXLAN): 이더넷 프레임 자체를 터널링, 브릿지처럼 동작
- **L3 VPN** (WireGuard, OpenVPN TUN, IPsec 터널 모드): IP 패킷을 터널링

### 캡슐화 과정 (OSI 관점)

```
1) App(L7)이 데이터를 보냄
2) TCP(L4)가 세그먼트로 쪼갬
3) IP(L3)가 헤더 붙여서 패킷 완성
   → [Inner IP | TCP | Payload] = "원본 패킷"

--- 여기서 WireGuard가 개입 ---

4) WireGuard가 이 원본 패킷을 암호화
5) WireGuard 헤더(32B)를 앞에 붙임
6) 이걸 새로운 UDP(L4) 페이로드로 만듦
7) 새로운 IP(L3) 헤더를 붙임
8) 물리 NIC의 L2 프레임으로 나감
```

### WireGuard 특징

| 항목 | WireGuard | OpenVPN |
|------|-----------|---------|
| 전송 계층 | UDP (항상) | TCP 또는 UDP |
| 동작 레벨 | 커널 모듈 | 유저스페이스 |
| 코드 크기 | ~4,000줄 | ~수십만 줄 |
| 암호화 협상 | 없음 (고정) | TLS 기반 협상 |
| 핸드셰이크 | 1-RTT | 복잡한 TLS 핸드셰이크 |

### 암호화 스위트 (고정, 협상 없음)

- **ChaCha20**: 대칭 암호화
- **Poly1305**: 인증
- **Curve25519**: 키 교환
- **BLAKE2s**: 해시

### Cryptokey Routing

```
Peer A:
  PublicKey = abc123...
  AllowedIPs = 10.0.0.1/32

Peer B:
  PublicKey = def456...
  AllowedIPs = 10.0.0.2/32
```

- 목적지 IP → 매칭되는 피어의 공개키로 암호화 → 해당 피어의 endpoint로 전송
- 라우팅 테이블과 암호화가 하나로 묶인 구조

### Stealth 특성

- idle 상태에서 포트가 열려 있어도 아무 응답을 하지 않음
- 포트 스캔에 잡히지 않음

### TCP over TCP 회피

- WireGuard가 UDP를 사용하는 이유
- 터널 안팎 모두 TCP면 재전송이 중첩(TCP meltdown)되면서 성능이 망가짐
- 바깥쪽 UDP는 전달만, 안쪽 TCP가 재전송 담당

### Tailscale의 역할

- WireGuard 위에 **제어 플레인**을 얹은 것
- 키 배포, NAT traversal, DERP 릴레이 등 관리
- 터널링 자체의 동작은 순수 WireGuard와 동일

---

## 9. 블로그 작성 가이드

### 글의 흐름 제안

1. **도입**: K8s + Tailscale + ElastiCache 연결에서 "연결은 되는데 헬스체크만 실패"하는 트러블슈팅 경험으로 시작
2. **MTU 기본 개념**: MTU 정의, 패킷 구조, MSS 계산
3. **Fragmentation과 PMTUD**: DF 비트, ICMP, PMTUD Black Hole
4. **WireGuard/Tailscale**: L3 VPN의 이중 캡슐화 문제, 왜 드롭이 발생하는지
5. **해결**: 터널 MTU 조정, 적정 값 계산
6. **(부록)** WireGuard 프로토콜 상세

### 톤 & 스타일 (marsboy 블로그 참고)

- **문체**: 합니다/습니다체 (예: "~를 다룹니다", "~할 수 있다")
- **구조**: 번호가 매겨진 대제목 (## 1. ~~, ## 2. ~~), 하위에 ### 소제목
- **코드블록**: 명령어, 설정 파일, 출력 결과를 적극 활용
- **다이어그램**: ASCII 다이어그램 또는 이미지 다이어그램 삽입 (코딩 에이전트가 추후 보강)
- **표**: 비교 항목은 표로 정리
- **인용/참고 박스**: `>` 블록 활용
- **개인 경험 → 개념 → 해결 흐름**: 실전 트러블슈팅으로 시작해서 원리를 설명하는 구조

### 다이어그램 보강 포인트 (코딩 에이전트용)

- [ ] OSI 7 Layer에서 각 레이어별 데이터 단위 + 캡슐화 방향 다이어그램
- [ ] 일반 패킷 vs WireGuard 캡슐화 패킷 비교 다이어그램
- [ ] PMTUD 동작 시퀀스 다이어그램
- [ ] Redis PING(소) vs INFO(대) 패킷이 터널을 통과하는 과정 비교 다이어그램
- [ ] 해결 전/후 패킷 크기 비교 다이어그램
