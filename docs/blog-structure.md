# 블로그 글 구조 가이드

## 메타 정보

- **제목(안)**: `Tailscale 터널에서 Redis 헬스체크가 실패하는 이유: MTU 문제 해결기`
- **카테고리**: Kubernetes / Networking / Troubleshooting
- **대상 독자**: K8s를 운영하고 있고, VPN 터널(Tailscale/WireGuard 등)을 통해 외부 서비스에 연결하는 엔지니어
- **톤**: `~다` 체 기술 문서. 실전 경험 기반 서사 + 개념 설명 + 코드/다이어그램이 어우러지는 스타일
- **참고 블로그**: https://www.marsboy.online/ko/posts/

---

## 글의 전체 흐름

> 장애 현상 목격 → 원인 추적 → 개념 이해 → 해결 → 영구 적용 → 회고

이 글은 **"왜 작은 패킷은 되는데 큰 패킷은 안 되는가?"** 라는 하나의 질문을 축으로 전개한다.

---

## 섹션 구성

### 1. 도입: 인프라 구성과 맥락
- 현재 인프라 소개: 온프레미스 k3s 클러스터 + Tailscale VPN + AWS ElastiCache (Private Subnet)
- 왜 이런 구성이 필요했는가 (온프레미스 → AWS managed service 연결 필요성)
- Tailscale을 선택한 이유 (간편한 WireGuard 기반 메시 VPN)
- **[다이어그램 필요]**: 전체 인프라 토폴로지 (k3s nodes → Tailscale tunnel → AWS VPC → ElastiCache)

### 2. 문제 발생: Redis 헬스체크 실패
- Spring Boot NotificationApplication 배포 후 발생한 현상
- Redis 연결 자체는 성공 (HELLO, CLIENT 명령은 통과)
- 그러나 헬스체크(INFO 명령)에서 타임아웃 반복
- Pod가 Ready 상태가 되지 않는 증상
- **[코드 필요]**: 실제 에러 로그 발췌 (Redis handshake 성공 → INFO timeout 패턴)

### 3. 원인 추적: 작은 패킷은 되고, 큰 패킷은 안 된다
- 로그에서 발견한 패턴: 150 bytes, 10 bytes → 통과 / 수 KB의 INFO 응답 → 드롭
- "패킷 크기에 따라 성공/실패가 갈린다"는 결정적 단서
- MTU 관련 문제라는 가설 수립
- **[코드 필요]**: ping 테스트로 MTU 확인
  ```bash
  ping -M do -s 1400 10.128.168.231
  # 결과: local error: message too long, mtu=1280
  ```

### 4. MTU 개념 정리: 패킷은 어떻게 전달되는가
> 이 섹션은 트러블슈팅 서사를 잠시 멈추고, 독자에게 MTU를 이해시키는 "개념 파트"이다.

#### 4-1. MTU와 MSS
- MTU(Maximum Transmission Unit): 네트워크 인터페이스가 한 번에 전송할 수 있는 최대 프레임 크기
- 일반적인 이더넷 MTU: 1500 bytes
- 패킷 구조: IP Header(20B) + TCP Header(20B) + Payload
- MSS(Maximum Segment Size) = MTU - IP Header - TCP Header = 1460 bytes
- **[다이어그램 필요]**: 패킷 구조도 (Ethernet Frame → IP Header → TCP Header → Payload)

#### 4-2. Fragmentation과 DF 비트
- MTU를 초과하는 패킷 → 분할(fragmentation) 또는 폐기
- DF(Don't Fragment) 비트가 설정된 경우: 분할하지 않고 ICMP "Fragmentation Needed" 메시지를 보냄
- TCP는 기본적으로 DF 비트를 설정함

#### 4-3. Path MTU Discovery (PMTUD)
- 경로상 가장 작은 MTU에 맞추어 패킷 크기를 조정하는 메커니즘
- ICMP "Fragmentation Needed" 메시지에 의존
- **PMTUD Black Hole**: 중간 장비(방화벽, Security Group 등)가 ICMP를 차단하면 발신자가 MTU 초과를 인지하지 못해 패킷이 무한히 드롭되는 현상
- AWS 환경에서 Security Group이 ICMP를 기본 차단하므로 특히 빈번
- **[다이어그램 필요]**: PMTUD 정상 동작 vs Black Hole 비교 흐름도

#### 4-4. WireGuard(Tailscale)가 MTU에 미치는 영향
- VPN 터널의 이중 캡슐화 문제의 핵심
- Inner packet (1500B) + WireGuard Header + UDP Header + Outer IP Header ≈ 1560B
- 물리 NIC MTU(1500B) 초과 → 패킷 드롭
- 해결 원리: Tailscale 터널 인터페이스의 MTU를 ~1420으로 낮추어 캡슐화 후에도 1500B 이내로 유지
- **[다이어그램 필요]**: 이중 캡슐화 과정 시각화 (Inner Packet → WireGuard Encapsulation → Outer Packet > 1500)

### 5. 해결: MSS Clamping 적용
- 개념 섹션에서 다시 트러블슈팅 서사로 복귀
- 왜 MSS Clamping인가: 터널 MTU에 맞추어 TCP MSS를 강제로 낮추는 방법
- 적용한 iptables 룰:
  ```bash
  iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
  ```
- 적용 직후 결과: Redis INFO 응답이 여러 TCP 세그먼트(1024B, 4175B 등)로 분할되어 정상 수신
- **[코드 필요]**: 적용 전/후 로그 비교

### 6. 두 번째 문제: 다른 워커 노드
- 첫 번째 Pod는 해결되었으나, 두 번째 replica가 여전히 Ready 상태가 되지 않음
- 원인: MSS clamping은 k3s-worker2에만 적용했고, 두 번째 Pod는 다른 노드에 스케줄링됨
- K8s에서 노드별 네트워크 설정이 독립적이라는 점을 놓친 실수
- 모든 워커 노드에 동일한 iptables 룰 적용하여 해결

### 7. 영구 적용: 재부팅 후에도 유지되도록
- iptables 룰은 재부팅하면 사라짐 → `iptables-persistent`로 영속화
  ```bash
  apt install iptables-persistent
  netfilter-persistent save
  ```
- flannel MTU 영구 설정:
  - `/etc/rancher/k3s/config.yaml`에 flannel backend 설정
  - flannel MTU = Tailscale MTU(1280) - VXLAN overhead(50) - buffer(10) = **1220**
  - 설정 변경 후 기존 Pod는 rollout restart 필요
- **[코드 필요]**: k3s config.yaml 및 flannel 설정 예시

### 8. 부록/회고
- 이번 트러블슈팅에서 배운 것들 정리
- Spring Security 자동 생성 패스워드가 prod 프로파일에서 사용되고 있던 설정 갭 언급
- 향후 모니터링 방안 (ArgoCD Notifications, Datadog Monitor + Webhook, Botkube 등)
- "작은 패킷은 되는데 큰 패킷이 안 된다"는 증상이 보이면 MTU를 의심하라는 교훈

---

## 코딩 에이전트를 위한 지시사항

### Hugo 관련
- Hugo 블로그 포맷에 맞는 front matter 포함 (title, date, summary, tags 등)
- 이미지/다이어그램 경로는 `images/` 하위 디렉토리 기준으로 작성
- 다이어그램은 placeholder로 `![설명](images/파일명.png)` 형태로 남겨두되, 주석으로 다이어그램에 포함되어야 할 요소를 기술

### 톤 & 스타일
- 반드시 `~다` 체 사용 (예: "~이다", "~한다", "~된다")
- 도입부에서 독자의 흥미를 끄는 문제 상황 제시로 시작
- 개념 설명 시 비유나 단계적 설명 활용
- 코드 블록에는 주석과 실행 결과를 함께 포함
- 기존 블로그 글(https://www.marsboy.online/ko/posts/)의 포맷과 톤을 최대한 따를 것

### 콘텐츠 소스
- `context-conversation-1.md`: MTU 개념 설명 관련 대화 내용 (섹션 4에 주로 활용)
- `context-conversation-2.md`: 실전 트러블슈팅 대화 내용 (섹션 2, 3, 5, 6, 7에 주로 활용)
- 두 파일의 내용을 이 구조 가이드에 맞추어 재구성할 것

### 다이어그램 (사용자가 직접 작성 예정)
- 본문에 `[다이어그램 필요]` 표시가 있는 곳에 placeholder 이미지 태그를 남기고, 주석으로 다이어그램에 들어갈 핵심 요소를 설명
- 사용자가 후속 작업으로 다이어그램을 추가할 예정
