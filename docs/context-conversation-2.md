# context-conversation-2.md
# 트러블슈팅 컨텍스트: 온프레미스 K8s → AWS ElastiCache MTU 문제

> 이 문서는 Claude Code가 블로그 초안을 작성할 때 참조할 원본 자료입니다.
> 2026-03-22에 발생한 실제 장애 대응 과정을 기록합니다.

---

## 1. 인프라 구성

### K3s 클러스터

- **배포판**: K3s (경량 Kubernetes)
- **CNI**: Flannel (VXLAN 백엔드)
- **Pod CIDR**: `10.42.0.0/16`
- **확인된 워커 노드**: `k3s-worker2` (Ubuntu, SSH 접속 시 `*** System restart required ***` 표시)
- **노드 OS 유저**: `uoslife`
- **GitOps**: ArgoCD

### 네트워크 연결 (온프레미스 ↔ AWS)

- **VPN**: Tailscale (WireGuard 기반)
- **터널 인터페이스**: `tailscale0`
- **터널 MTU**: **1280** (Tailscale 기본값, IPv6 최소 MTU 호환을 위해 보수적 설정)
- **Subnet Router**: Tailscale subnet router로 AWS VPC CIDR을 advertise하는 구조

### AWS 리소스

- **ElastiCache (Redis)**:
  - IP: `10.128.168.231`
  - Port: `6379`
  - Private Subnet에 위치 (온프레미스에서 직접 접근 불가, Tailscale 터널 경유 필수)
  - Redis 버전: `7.1.0`
  - OS: `Amazon ElastiCache` (ARM 기반, `monotonic_clock:ARM CNTVCT`)
  - Mode: `standalone`, Role: `master`
  - Max Memory: `384.00MB`, Used: `46.95MB`
  - Keys: `29,589` (expires: `28,874`)
  - Uptime: `425일`
  - SSL: `no`
  - Max Clients: `20,000`, Connected: `18`

### 애플리케이션 스택

- **서비스**: `NotificationApplicationKt` (Spring Boot `v3.5.4`)
- **언어/런타임**: Kotlin, Java 21.0.10
- **Redis 클라이언트**: Lettuce (Spring Data Redis 기본)
- **서버 포트**: `8081`
- **프로파일**: `prod`
- **모니터링**: Datadog
- **배포**: ArgoCD

---

## 2. 장애 현상

### 증상 요약

- ArgoCD에서 Notification 서비스 Pod가 **Ready 상태로 전환되지 않음**
- Spring Boot 앱 자체는 정상 기동됨 (13.787초)
- Redis 연결(TCP handshake)은 성공
- **Redis HELLO/CLIENT 커맨드는 정상 응답, INFO 커맨드에 대한 응답이 없음**
- Spring Boot Actuator의 Redis health check가 INFO 커맨드를 사용 → health check 실패 → Pod Not Ready

### 장애 로그 (1차 — MSS Clamping 적용 전)

**정상 부분: Redis 연결 및 HELLO/CLIENT 커맨드**

```log
2026-03-22 01:22:20.871 [http-nio-8081-exec-3] DEBUG io.lettuce.core.RedisClient - Resolved SocketAddress 10.128.168.231/<unresolved>:6379 using redis://10.128.168.231
2026-03-22 01:22:20.871 [http-nio-8081-exec-3] DEBUG io.lettuce.core.AbstractRedisClient - Connecting to Redis at 10.128.168.231/<unresolved>:6379
```

```log
2026-03-22 01:22:20.983 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0e890d24, /10.42.1.48:49078 -> /10.128.168.231:6379, epid=0x1, chid=0x1] write(ctx, AsyncCommand [type=HELLO, output=GenericMapOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command], promise)
2026-03-22 01:22:21.039 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0e890d24, /10.42.1.48:49078 -> /10.128.168.231:6379, epid=0x1, chid=0x1] Received: 150 bytes, 1 commands in the stack
2026-03-22 01:22:21.051 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0e890d24, /10.42.1.48:49078 -> /10.128.168.231:6379, epid=0x1, chid=0x1] Completing command LatencyMeteredCommand [type=HELLO, output=GenericMapOutput [output={server=redis, version=7.1.0, proto=3, id=2334161, mode=standalone, role=master}, error='null'], commandType=io.lettuce.core.protocol.AsyncCommand]
```

```log
2026-03-22 01:22:21.053 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0e890d24, /10.42.1.48:49078 -> /10.128.168.231:6379, epid=0x1, chid=0x1] write(ctx, [AsyncCommand [type=CLIENT, ...], AsyncCommand [type=CLIENT, ...]], promise)
2026-03-22 01:22:21.062 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0e890d24, /10.42.1.48:49078 -> /10.128.168.231:6379, epid=0x1, chid=0x1] Received: 10 bytes, 2 commands in the stack
2026-03-22 01:22:21.063 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - Completing command LatencyMeteredCommand [type=CLIENT, output=StatusOutput [output=OK, error='null'], ...]
```

```log
2026-03-22 01:22:21.065 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.AbstractRedisClient - Connecting to Redis at 10.128.168.231/<unresolved>:6379: Success
```

**비정상 부분: INFO 커맨드 — 응답 없음 (10초 간격 반복)**

```log
2026-03-22 01:22:21.085 [http-nio-8081-exec-3] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, output=StatusOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2026-03-22 01:22:21.089 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandEncoder - [channel=0x0e890d24, /10.42.1.48:49078 -> /10.128.168.231:6379] writing command AsyncCommand [type=INFO, ...]
# ⚠️ 이후 Received 로그 없음 — 응답이 돌아오지 않음

2026-03-22 01:22:30.582 [http-nio-8081-exec-4] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
2026-03-22 01:22:30.582 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandEncoder - writing command AsyncCommand [type=INFO, ...]
# ⚠️ Received 로그 없음

2026-03-22 01:22:40.623 [http-nio-8081-exec-6] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
2026-03-22 01:22:40.624 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandEncoder - writing command AsyncCommand [type=INFO, ...]
# ⚠️ Received 로그 없음

2026-03-22 01:22:50.664 [http-nio-8081-exec-8] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
# ⚠️ Received 로그 없음

2026-03-22 01:22:55.989 [http-nio-8081-exec-9] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
# ⚠️ Received 로그 없음

2026-03-22 01:23:00.705 [http-nio-8081-exec-10] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
# ⚠️ Received 로그 없음

2026-03-22 01:23:06.030 [http-nio-8081-exec-2] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
# ⚠️ Received 로그 없음
```

**핵심 관찰 포인트: 응답 크기별 통과 여부**

| 커맨드 | 응답 크기 | 결과 |
|--------|----------|------|
| HELLO | 150 bytes | ✅ 정상 수신, `Completing command` 로그 있음 |
| CLIENT (×2) | 10 bytes (OK, OK) | ✅ 정상 수신, `Completing command` 로그 있음 |
| INFO | ~5,000 bytes (Redis 전체 정보 덤프) | ❌ `Received` 로그 자체가 없음, 응답 드롭 |

---

## 3. 원인 분석 과정

### 가설 수립

- 작은 패킷(150B, 10B)은 통과, 큰 패킷(INFO 응답 ~5KB)은 사라지는 패턴
- 온프레미스 K8s Pod (10.42.1.48) → AWS ElastiCache (10.128.168.231, Private Subnet) 경로
- Tailscale VPN 터널 경유 → **MTU 문제 가설**

### 검토한 원인 후보들

1. **Redis ACL/권한 문제**: HELLO/CLIENT는 되는데 INFO 권한이 없을 수 있음 → 가능성 낮음 (ACL 문제면 에러 응답이 와야 함)
2. **네트워크 MTU 문제** (최유력): 작은 패킷만 통과하고 큰 패킷이 드롭되는 전형적 증상
3. **Redis 과부하/slow query**: INFO가 블로킹될 수 있음 → 가능성 낮음 (INFO는 가벼운 커맨드)
4. **비대칭 라우팅**: ElastiCache 응답 목적지 10.42.1.48 (Pod IP)에 대한 VPC 라우팅 미설정 → MTU와 복합 가능성

---

## 4. MTU 확인

### 워커 노드에서 ping DF 테스트

```bash
uoslife@k3s-worker2:~$ ping -M do -s 1400 10.128.168.231
PING 10.128.168.231 (10.128.168.231) 1400(1428) bytes of data.
ping: local error: message too long, mtu=1280
ping: local error: message too long, mtu=1280
ping: local error: message too long, mtu=1280
ping: local error: message too long, mtu=1280
^C
--- 10.128.168.231 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3055ms
```

```bash
uoslife@k3s-worker2:~$ ping -M do -s 1200 10.128.168.231
PING 10.128.168.231 (10.128.168.231) 1200(1228) bytes of data.
# (응답 대기 — 1228 < 1280이므로 패킷은 나가지만 return path 이슈 가능)
```

### 확인된 사실

- Tailscale 인터페이스(`tailscale0`) MTU: **1280**
- `ping -M do -s 1400` → 즉시 `message too long, mtu=1280` (로컬에서 바로 거부)
- 1400 + 28 (IP+ICMP 헤더) = 1428 > 1280 → 드롭
- 1200 + 28 = 1228 < 1280 → 패킷은 나감

### 왜 MTU가 1280인가

- Tailscale은 다양한 NAT/네트워크 환경 호환성을 위해 **보수적으로 1280 설정**
- IPv6 최소 MTU가 1280이라 거의 모든 경로에서 보장됨
- WireGuard 프로토콜 오버헤드: 외부 IP(20B) + UDP(8B) + WireGuard 헤더(32B) ≈ 60B

---

## 5. 1차 해결 — MSS Clamping

### 적용한 iptables 룰

```bash
sudo iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

**룰 해석:**
- `-t mangle`: 패킷 변조(mangle) 테이블
- `-A FORWARD`: 노드를 경유(forward)하는 패킷에 적용 (Pod → 외부 트래픽)
- `-o tailscale0`: tailscale0 인터페이스로 나가는 패킷 대상
- `-p tcp --tcp-flags SYN,RST SYN`: TCP SYN 패킷만 대상 (핸드셰이크 시점)
- `-j TCPMSS --clamp-mss-to-pmtu`: MSS 값을 Path MTU에 맞게 자동 조정

### 적용 후 로그 (정상 — INFO 응답 수신 확인)

**첫 번째 INFO health check:**

```log
2026-03-22 01:38:20.012 [http-nio-8081-exec-2] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
2026-03-22 01:38:20.014 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandEncoder - [channel=0x0b4947d8, /10.42.1.48:55962 -> /10.128.168.231:6379] writing command AsyncCommand [type=INFO, ...]
2026-03-22 01:38:20.022 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0b4947d8, /10.42.1.48:55962 -> /10.128.168.231:6379, epid=0x1, chid=0x1] Received: 1024 bytes, 1 commands in the stack
2026-03-22 01:38:20.022 [lettuce-nioEventLoop-6-1] DEBUG i.l.core.protocol.RedisStateMachine - Decode done, empty stack: false
2026-03-22 01:38:20.022 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0b4947d8, /10.42.1.48:55962 -> /10.128.168.231:6379, epid=0x1, chid=0x1] Received: 4175 bytes, 1 commands in the stack
2026-03-22 01:38:20.023 [lettuce-nioEventLoop-6-1] DEBUG i.l.core.protocol.RedisStateMachine - Decode done, empty stack: true
2026-03-22 01:38:20.023 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - [channel=0x0b4947d8, /10.42.1.48:55962 -> /10.128.168.231:6379, epid=0x1, chid=0x1] Completing command LatencyMeteredCommand [type=INFO, output=StatusOutput [output=# Server
redis_version:7.1.0
...
# Keyspace
db0:keys=29589,expires=28874,avg_ttl=754179338
, error='null'], commandType=io.lettuce.core.protocol.AsyncCommand]
2026-03-22 01:38:20.024 [http-nio-8081-exec-2] DEBUG o.s.data.redis.core.RedisConnectionUtils - Closing Redis Connection
```

**두 번째 INFO health check (10초 후):**

```log
2026-03-22 01:38:29.423 [http-nio-8081-exec-4] DEBUG io.lettuce.core.RedisChannelHandler - dispatching command AsyncCommand [type=INFO, ...]
2026-03-22 01:38:29.431 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - Received: 2252 bytes, 1 commands in the stack
2026-03-22 01:38:29.432 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - Received: 2947 bytes, 1 commands in the stack
2026-03-22 01:38:29.432 [lettuce-nioEventLoop-6-1] DEBUG io.lettuce.core.protocol.CommandHandler - Completing command LatencyMeteredCommand [type=INFO, ...]
2026-03-22 01:38:29.433 [http-nio-8081-exec-4] DEBUG o.s.data.redis.core.RedisConnectionUtils - Closing Redis Connection
```

### 전후 비교 요약

| 항목 | Before (MSS Clamping 없음) | After (MSS Clamping 적용) |
|------|---------------------------|--------------------------|
| HELLO (150B) | ✅ Received: 150 bytes | ✅ Received: 150 bytes |
| CLIENT (10B) | ✅ Received: 10 bytes | ✅ Received: 10 bytes |
| INFO (~5KB) | ❌ Received 로그 없음, 응답 드롭 | ✅ 1024B + 4175B 두 청크로 분할 수신 |
| Health Check | ❌ 실패 (Pod Not Ready) | ✅ 성공 (Pod Ready) |
| `Closing Redis Connection` | 로그 없음 (타임아웃 대기) | ✅ 정상 종료 |

### 핵심 관찰: TCP 세그먼트 분할

MSS Clamping 적용 후 INFO 응답이 **여러 TCP 세그먼트로 나뉘어** 도착:
- 1차 수신: `Received: 1024 bytes` → `Decode done, empty stack: false` (아직 더 받아야 함)
- 2차 수신: `Received: 4175 bytes` → `Decode done, empty stack: true` (디코딩 완료)
- 총 ~5,199 bytes 수신

이는 MSS가 MTU 1280에 맞게 줄어들면서, ElastiCache가 응답을 더 작은 세그먼트로 나눠 보내게 된 것.

---

## 6. 2차 문제 — 다른 워커 노드

### 증상

- MSS Clamping 적용 후 첫 번째 Pod는 정상화
- **두 번째 Pod replica가 여전히 Ready 안 됨**
- ArgoCD에서 Pod가 최대 1개까지만 뜨고 2개가 안 뜨는 상태

### 두 번째 Pod 로그 (다른 워커 노드)

```log
2026-03-22 01:49:50.695 [main] INFO  c.u.n.NotificationApplicationKt - Starting NotificationApplicationKt v0.0.1-SNAPSHOT using Java 21.0.10 with PID 1 (/app/notification.jar started by appuser in /app)
2026-03-22 01:49:50.697 [main] INFO  c.u.n.NotificationApplicationKt - The following 1 profile is active: "prod"
...
2026-03-22 01:50:02.984 [main] INFO  c.u.n.NotificationApplicationKt - Started NotificationApplicationKt in 13.306 seconds (process running for 14.424)
# ⚠️ 이후 http-nio-8081-exec-* 스레드 로그 없음 — health check 요청 자체가 들어오지 않음
```

### 원인

- MSS Clamping을 `k3s-worker2`에만 적용
- 두 번째 Pod가 **다른 워커 노드**에 스케줄됨
- 해당 노드에는 iptables 룰이 없어서 동일한 MTU 문제 발생
- **k3s에서 iptables 룰은 노드별로 독립적** — 한 노드에 적용한 룰은 다른 노드에 자동 전파되지 않음

### 확인 명령어

```bash
# 각 Pod가 어느 노드에 스케줄됐는지 확인
kubectl get pods -n <namespace> -o wide | grep notification
```

### 해결

모든 워커 노드에 동일한 iptables 룰 적용:

```bash
# 모든 워커 노드에서 실행
sudo iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

적용 후 두 번째 Pod도 정상적으로 Ready 상태 전환 확인.

---

## 7. 영구 적용

### 문제

`iptables` 명령으로 적용한 룰은 메모리에만 존재 → 노드 리부트 시 사라짐.
(`k3s-worker2` SSH 접속 시 `*** System restart required ***` 표시되어 있어 리부트 예정)

### 방법 1: iptables-persistent

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

### 방법 2: k3s flannel MTU 근본 수정

서버 노드(master)의 `/etc/rancher/k3s/config.yaml`:

```yaml
flannel-conf: /etc/rancher/k3s/flannel.json
```

`/etc/rancher/k3s/flannel.json` 생성:

```json
{
  "Network": "10.42.0.0/16",
  "EnableIPv4": true,
  "EnableIPv6": false,
  "Backend": {
    "Type": "vxlan",
    "VNI": 1,
    "Port": 8472,
    "MTU": 1220
  }
}
```

### MTU 계산식

```
Tailscale 터널 MTU: 1280
- VXLAN 오버헤드:     50
- 여유분:             10
────────────────────────
flannel MTU:        1220
```

### 적용 순서

```bash
# 1. 현재 flannel 설정 확인
cat /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json

# 2. config 파일 생성 후 k3s 재시작 (master)
sudo systemctl restart k3s

# 3. 워커 노드 재시작
sudo systemctl restart k3s-agent

# 4. flannel MTU 확인
ip link show flannel.1

# 5. 기존 Pod rollout restart (새 MTU는 신규 Pod에만 적용됨)
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

### 주의사항

- flannel MTU 변경은 **기존 실행 중인 Pod의 veth에 즉시 반영되지 않음**
- 변경 후 기존 Pod들을 rollout restart 해야 새 MTU 적용
- 전체 워크로드에 영향 → **점검 시간에 진행 권장**
- 현재 flannel 백엔드가 VXLAN인지 host-gw인지에 따라 설정이 달라짐 → 사전 확인 필요

---

## 8. 부수적 발견

### Spring Security 자동 생성 패스워드 (prod 프로파일)

모든 로그에서 반복적으로 출력:

```log
2026-03-22 01:21:50.964 [main] INFO  c.u.n.NotificationApplicationKt - The following 1 profile is active: "prod"
...
2026-03-22 01:22:02.477 [main] WARN  o.s.b.a.s.s.UserDetailsServiceAutoConfiguration -
Using generated security password: 063b14ce-21b0-4f3e-a516-816fe3fe35bf
This generated password is for development use only. Your security configuration must be updated before running your application in production.
```

- **prod** 프로파일인데 Spring Security auto-config 기본 패스워드가 생성/노출됨
- `SecurityFilterChain` 설정이 빠져있거나 불완전한 것으로 추정
- prod 환경에서 반드시 처리 필요

### 쿠버네티스 서비스 상태 모니터링 방안 (대화 초반 논의)

장애 감지 → webhook 알림을 위한 도구 후보:

**1. ArgoCD Notifications** (기존 스택 활용, 추가 설치 불필요)
```yaml
trigger.on-health-degraded: |
  - when: app.status.health.status == 'Degraded'
    send: [webhook-alert]
```
- ArgoCD가 관리하는 앱의 health status 변화(Degraded, Missing, Unknown) 감지
- Webhook, Slack 등으로 알림 전송
- `argocd-notifications-cm` ConfigMap에 trigger/template 정의

**2. Datadog Monitor + Webhook** (기존 Datadog 활용)
- 서비스 메트릭 기반 (응답시간, 에러율, Pod restart 등) 조건 설정
- webhook receiver로 알림

**3. Botkube** (가벼운 K8s 이벤트 감시)
- CrashLoopBackOff, OOMKilled, Failed scheduling 등 K8s 네이티브 이벤트 실시간 감지
- Slack/webhook 전송
- ArgoCD 밖의 리소스도 커버

**4. Robusta** (더 rich한 컨텍스트)
- Botkube와 유사하나 troubleshooting 컨텍스트(로그, 이벤트 요약) 포함
- 상대적으로 무거움

---

## 부록: 디버깅 명령어 모음

### MTU 확인

```bash
# DF(Don't Fragment) 비트 설정하여 ping — MTU 초과 시 즉시 실패
ping -M do -s 1400 <target-ip>
ping -M do -s 1200 <target-ip>

# 인터페이스별 MTU 확인
ip link | grep -E 'tailscale|flannel|cni0|vxlan'
ip link show tailscale0
ip link show flannel.1
```

### MSS Clamping 적용

```bash
# 적용
sudo iptables -t mangle -A FORWARD -o tailscale0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# 확인
sudo iptables -t mangle -L FORWARD -v

# 영구 저장
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

### K8s Pod 디버깅

```bash
# Pod 상태 및 노드 확인
kubectl get pods -n <namespace> -o wide | grep <service-name>

# Pod 이벤트 확인 (스케줄링 실패 원인 등)
kubectl describe pod <pod-name> -n <namespace>

# Deployment 롤아웃 전략 확인
kubectl get deploy <deployment-name> -n <namespace> -o jsonpath='{.spec.strategy}'

# Pod에서 Redis 직접 테스트
kubectl exec -it <pod-name> -- redis-cli -h 10.128.168.231 INFO
```

### flannel 설정 확인

```bash
# 현재 flannel 네트워크 설정
cat /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json
```

---

## 부록: 블로그 초안 방향성

- **대상 독자**: 주니어 DevOps/인프라 엔지니어
- **톤**: 실무 회고 + 삽질기 (캐주얼)
- **구성**: 문제 발견 → 원인 분석 → 해결 트러블슈팅 스토리 + MTU/네트워크 이론 설명 비중 높게
- **블로그 톤 참고**: https://www.marsboy.online/ko/posts/
- **코딩 에이전트가 추가할 것**: 내용 검수, 다이어그램 (온프레미스 K8s ↔ Tailscale ↔ AWS 구성도, MTU/패킷 구조 시각화 등)
