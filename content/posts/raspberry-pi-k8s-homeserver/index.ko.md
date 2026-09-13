---
title: "라즈베리파이로 K8s 홈서버 구축하기"
date: 2026-09-13
draft: false
tags: ["kubernetes", "k3s", "raspberry-pi", "homelab"]
translationKey: "raspberry-pi-k8s-homeserver"
summary: "AWS 프리티어 종료를 계기로 라즈베리파이에 Raspberry Pi OS를 올리고, 네트워크를 잡고, K3s로 쿠버네티스 클러스터를 구축하기까지의 과정을 다룬다."
---

![라즈베리파이 로고](images/raspberry-pi-logo.png)

AWS를 통해 다양한 인프라 서비스를 접한 지 거의 일 년이 다 되어 갈 무렵, 프리티어 기간이 슬슬 끝나가고 있었다. 프리티어가 종료되면 매달 고정적인 비용이 발생하는데, 학습 목적으로 인프라를 운영하기에는 부담스러운 금액이었다. 그래서 책상 위에 작은 컴퓨터 몇 대를 올려두고 쿠버네티스 클러스터를 직접 만들어 보기로 했다.

이 글은 그 시작점의 기록이다. 라즈베리파이에 운영체제를 올리고, 네트워크에 붙이고, K3s로 클러스터를 구성해 첫 워크로드가 응답하는 순간까지를 다룬다. 처음 글을 쓴 뒤로 이 클러스터를 계속 운영해 왔기 때문에, 당시에는 몰랐지만 지금은 처음부터 다르게 했을 부분들도 함께 정리했다. Raspberry Pi OS가 Bookworm으로 넘어오면서 바뀐 설정 경로가 대표적이다.

---

## 1. 동기: 왜 온프레미스인가?

프리티어가 끝나는 시점에서 두 가지 선택지가 있었다. 비용을 내고 클라우드에 남거나, 직접 서버를 두거나.

| | 클라우드 (AWS) | 온프레미스 (라즈베리파이) |
|---|---|---|
| **초기 비용** | 없음 (프리티어) | 라즈베리파이 구매 비용 |
| **유지 비용** | 월 과금 (프리티어 종료 후) | 전기세 (미미) |
| **학습 범위** | 관리형 서비스 위주 | 네트워크, OS, 클러스터 전반 |
| **유연성** | 높음 | 물리적 제약 있음 |
| **장애 시 책임** | 클라우드 사업자 | 전부 내 몫 |

마침 학사에 거주하고 있어 공과금을 내지 않는 상황이었기 때문에, 라즈베리파이만 있다면 사실상 무료로 온프레미스 환경을 구축할 수 있었다. 비용보다 더 큰 이유는 학습 범위였다. 관리형 서비스는 편리한 만큼 많은 것을 감춘다. 노드가 어떻게 네트워크에 붙는지, 컨테이너 런타임이 무엇인지, 클러스터 밖에서 들어오는 트래픽이 어떤 경로로 Pod까지 도달하는지는 EKS 콘솔에서는 보이지 않는다. 라즈베리파이 위에서는 그 모든 것을 직접 만져야 한다.

마지막 행은 나중에 추가한 것이다. 정전이 나면 클러스터가 내려가고, 공유기가 재부팅되면 IP가 바뀌고, SD 카드가 수명을 다하면 노드가 사라진다. 이 모든 것이 학습의 일부라는 점은 분명하지만, 무언가를 진지하게 서비스할 계획이라면 미리 알고 시작하는 편이 좋다.

"전기세가 미미하다"는 말도 숫자로 확인해 두자. 라즈베리파이 4는 idle 상태에서 약 3W, 부하 시 6~7W를 소비한다. 두 대를 24시간 켜 두면 넉넉히 잡아 월 10kWh 안팎이고, 국내 가정용 요금으로 환산하면 한 달에 2천 원이 되지 않는다. AWS에서 비슷한 사양의 `t4g.small` 인스턴스 두 대를 프리티어 없이 운영하면 월 3만 원 가까이 나온다. 초기 구매 비용은 몇 달이면 회수된다.

### 1.1 하드웨어 선택

처음 구축할 때는 손에 있던 **라즈베리파이 4**(4GB RAM) 두 대로 시작했다. 다시 고른다면 다음 기준으로 볼 것이다.

- **모델과 메모리**: K3s 서버 노드는 idle 상태에서도 500MB 이상을 쓴다. 4GB면 소규모 클러스터에 충분하지만, 모니터링 스택까지 올릴 계획이면 8GB 모델이 편하다. 라즈베리파이 5는 4 대비 CPU 성능이 2배 이상이고 PCIe로 NVMe SSD를 직접 연결할 수 있어, 새로 산다면 5를 권한다.
- **저장 장치**: SD 카드는 가장 먼저 고장 나는 부품이다. 쿠버네티스는 상태 저장소 쓰기, 컨테이너 이미지 레이어, 로그로 끊임없이 디스크를 두드린다. 라즈베리파이 4는 USB 3.0 SSD 부팅을, 5는 NVMe HAT을 지원하므로 서버 노드만이라도 SSD를 쓰는 것이 좋다. 이 글에서는 SD 카드로 진행한다.
- **전원**: 라즈베리파이는 전원이 불안정하면 저전압 경고와 함께 성능을 낮추거나 재부팅된다. 정품 어댑터 또는 USB PD를 지원하는 어댑터를 사용하고, USB SSD를 붙인다면 전원 소모가 더 커지는 점을 감안해야 한다.
- **냉각**: 방열판 없이 장시간 부하를 주면 80°C 근처에서 스로틀링이 걸린다. 팬 케이스나 최소한 방열판은 붙여 두는 편이 좋다.
- **네트워크**: 가능하다면 유선 LAN이 성능과 안정성 면에서 유리하다. 특히 노드 간 통신이 잦은 K8s 환경에서는 더욱 그렇다. 다만 이번 구축은 서비스 용도가 아닌 학습 목적이므로 무선랜으로 진행했다. 무선을 쓸 경우 뒤에서 다룰 절전 모드 문제를 반드시 확인해야 한다.

전원과 냉각이 충분한지는 OS를 올린 뒤 라즈베리파이 펌웨어가 제공하는 명령으로 확인할 수 있다. 클러스터가 이유 없이 느리거나 노드가 재부팅될 때 가장 먼저 볼 곳이다.

```bash
vcgencmd measure_temp        # temp=52.1'C
vcgencmd get_throttled       # throttled=0x0 이면 정상
```

`get_throttled`가 `0x0`이 아니면 저전압이나 과열로 스로틀링이 걸렸거나 걸린 적이 있다는 뜻이다. `0x50000`처럼 뒤쪽 비트만 켜져 있으면 과거에 발생했다가 지금은 해소된 상태이고, `0x50005`처럼 앞쪽 비트까지 켜져 있으면 지금도 진행 중이다. 어댑터를 바꾸거나 방열판을 붙인 뒤 이 값이 `0x0`으로 돌아오는지 보면 된다.

---

## 2. 라즈베리파이 OS 설정

하드웨어를 골랐다면 이제 운영체제를 올리고 네트워크에 붙일 차례이다. 이 장은 라즈베리파이를 "네트워크에 연결된 리눅스 호스트"로 만드는 과정이다.

### 2.1 인프라 구조

![홈서버 네트워크 구조도](images/homeserver-network-architecture.png)

가정용 공유기를 통해 위와 같은 구조로 네트워크를 구성한다. iptime 공유기를 사용하고 있는데, 대규모 트래픽을 처리할 것이 아니기 때문에 별도의 로드밸런서 없이 공유기 설정만으로 충분하다. 공유기가 DHCP 서버와 NAT 역할을 함께 하고, 라즈베리파이들은 그 아래의 `192.168.0.0/24` 대역에 놓인다.

### 2.2 OS 설치

가장 먼저 라즈베리파이에 운영체제를 설치한다. OS 설치는 SD 카드(또는 SSD)에 운영체제 이미지를 기록하고, 해당 저장 장치로 라즈베리파이를 부팅하는 방식이다.

라즈베리파이 공식 사이트에서 Imager를 다운로드한다.

- **Raspberry Pi Imager** : [https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/)

![라즈베리파이 Imager 실행 화면](images/raspberry-pi-imager.png)

Imager를 실행하면 위와 같은 화면이 나타난다. 저장소는 삽입된 SD 카드를 선택하고, 운영체제는 **Raspberry Pi OS Lite**(64-bit)를 선택한다. 서버로만 쓸 것이므로 데스크톱 환경이 빠진 Lite 버전이면 충분하고, 메모리도 그만큼 절약된다. 64-bit를 골라야 하는 이유는 컨테이너 이미지 때문이다. 대부분의 이미지가 `arm64`로 배포되고, 32-bit(`armv7`) 이미지는 점점 사라지고 있다.

> **주의**: 라즈베리파이용 Ubuntu에 K3s를 설치하면 커널 모듈이나 cgroup 설정 차이로 에러가 발생하는 경우가 있다. Raspberry Pi OS가 가장 시행착오가 적었다. Ubuntu를 꼭 사용해야 한다면 K3s 공식 문서의 [Raspberry Pi 요구사항](https://docs.k3s.io/installation/requirements#raspberry-pi)에 나온 추가 패키지(`linux-modules-extra-raspi`)를 먼저 설치해야 한다.

OS와 SD 카드를 선택한 후 설정 옵션에서 **SSH 허용**과 **무선랜 설정**을 반드시 진행해야 한다. 모니터와 키보드 없이 운영할 것이므로, 이 단계에서 SSH를 열어두지 않으면 부팅 후 접속할 방법이 없다.

![Imager 고급 옵션 - SSH 및 사용자 설정](images/raspberry-pi-imager-ssh-settings.png)

이 화면에서 SSH를 허용하고, 접속에 사용할 사용자 이름과 비밀번호를 설정한다. 예전에는 기본값으로 `pi` / `raspberry`가 만들어졌지만, Bullseye 이후로는 보안상의 이유로 기본 계정이 없어졌으므로 반드시 직접 만들어야 한다. 호스트 이름도 여기서 `k3s-master`, `k3s-worker1`처럼 역할이 드러나게 정해두면 나중에 `kubectl get nodes` 출력에서 바로 구분된다. 무선랜도 함께 설정하여 홈서버 공유기의 Wi-Fi 정보를 입력해 둔다.

가능하다면 비밀번호 대신 **공개키 인증**을 선택하고 자신의 SSH 공개키를 등록한다. 뒤에서 포트 포워딩으로 SSH를 외부에 노출하는 순간부터 비밀번호 로그인은 무차별 대입 공격의 대상이 된다.

### 2.3 첫 부팅 후 기본 설정

SD 카드에 이미지를 기록하고 라즈베리파이에 삽입하여 부팅하면, 라즈베리파이에 운영체제가 설치된다. 이제 이 작은 컴퓨터는 네트워크에 연결된 하나의 호스트가 되었다. 본격적인 설정에 앞서 모든 노드에서 공통으로 해 두면 좋은 것들이 있다.

먼저 64-bit 커널로 부팅되었는지 확인한다. 32-bit 이미지를 잘못 골랐다면 뒤에서 `arm64` 컨테이너 이미지가 실행되지 않으므로, 여기서 잡는 것이 가장 싸다.

```bash
uname -m
# aarch64
```

시간대와 패키지를 맞춘다. 쿠버네티스는 인증서 만료와 노드 간 통신에 시간을 사용하므로, 노드들의 시계가 어긋나 있으면 이유를 알기 어려운 문제가 생긴다.

```bash
sudo timedatectl set-timezone Asia/Seoul
timedatectl                       # NTP service: active 확인

sudo apt update && sudo apt full-upgrade -y
```

마지막으로 **스왑을 끈다**. Raspberry Pi OS는 기본적으로 SD 카드 위에 스왑 파일을 만들어 두는데, 쿠버네티스는 메모리 부족을 스왑으로 버티는 대신 Pod를 죽이고 다시 스케줄하는 방식을 전제로 설계되어 있다. 스왑이 켜져 있으면 노드가 느려지기만 할 뿐 문제가 드러나지 않고, SD 카드 수명만 깎아 먹는다. K3s는 스왑이 있어도 설치를 거부하지는 않지만, 끄는 편이 동작을 예측하기 쉽다.

```bash
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo systemctl disable --now dphys-swapfile

free -h                           # Swap 행이 0B이면 완료
```

### 2.4 SSH 접속 설정

호스트가 준비되었으니 이제 밖에서 들어갈 길을 만든다. 같은 네트워크, 다른 네트워크, 그리고 노드 사이의 세 가지 경로를 차례로 다룬다.

#### 사설 IP 확인 및 고정

공유기 관리 페이지(`192.168.0.1`)에 접속하여 **고급 설정 > 네트워크 관리 > DHCP 서버 설정**으로 이동한다.

![iptime DHCP 서버 설정 화면](images/iptime-dhcp-settings.png)

**DHCP**(Dynamic Host Configuration Protocol)는 네트워크에 연결된 호스트에게 IP 주소를 동적으로 할당하는 프로토콜이다. 문제는 연결할 때마다 다른 IP가 할당될 수 있다는 점이다. 서버로 사용할 라즈베리파이의 IP는 반드시 고정해야 한다. 특히 K3s는 워커 노드가 서버 노드의 IP로 접속하도록 설정되므로, 서버 노드의 IP가 바뀌면 클러스터가 통째로 깨진다.

IP를 고정하는 방법은 두 가지이다. 공유기에서 MAC 주소 기준으로 항상 같은 IP를 내려주는 **DHCP 고정 할당**과, 라즈베리파이 자체에 **정적 IP**를 설정하는 방법이다. 둘 중 하나만 해도 되지만, 공유기 쪽에서 고정하는 편이 관리 지점이 한 곳으로 모여 편하다. **사용 중인 IP 주소 정보** 섹션에서 라즈베리파이의 IP를 클릭하고 **수동 등록** 버튼을 눌러 IP를 고정시킨다.

라즈베리파이 쪽에서 정적 IP를 주고 싶다면 주의할 점이 있다. 예전 자료에는 `/etc/dhcpcd.conf`를 수정하라고 되어 있지만, **Raspberry Pi OS Bookworm(2023년 10월 이후)부터는 NetworkManager가 네트워크를 관리**하므로 그 파일은 더 이상 읽히지 않는다. `nmcli`로 설정한다.

```bash
# 연결 이름 확인 (Imager로 설정한 Wi-Fi는 보통 "preconfigured")
nmcli connection show

# 정적 IP, 게이트웨이, DNS 설정
sudo nmcli connection modify "preconfigured" \
  ipv4.method manual \
  ipv4.addresses 192.168.0.128/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns "1.1.1.1 8.8.8.8"

sudo nmcli connection up "preconfigured"
```

`ipv4.gateway`에는 공유기 주소(`192.168.0.1`)를 넣는다. 유선이라면 연결 이름이 `Wired connection 1`이다.

#### 같은 네트워크에서 접속

같은 Wi-Fi를 사용하는 환경에서는 사설 IP로 바로 SSH 접속이 가능하다.

```bash
ssh marsboy@192.168.0.128
```

패스워드 또는 등록한 키로 라즈베리파이에 정상적으로 접근할 수 있다.

#### 다른 네트워크에서 접속 (포트 포워딩)

외부 네트워크에서 접속하려면 공유기에 **포트 포워딩** 설정이 필요하다.

![포트 포워딩 구조도 - 공유기를 기준으로 내부/외부 네트워크 분리](images/port-forwarding-diagram.png)

공유기 내부의 호스트들은 DHCP에 의해 `192.168.0.*` 대역의 사설 IP를 할당받는다. 하지만 외부에서는 이 사설 IP를 직접 식별할 수 없다. 따라서 공유기의 공인 IP를 통해 들어오는 특정 포트의 연결을 내부 사설 IP로 전달하는 **포트 포워딩** 설정이 필요하다.

구체적으로는 **외부의 22번 포트로 들어오는 연결을 내부 192.168.0.128의 22번 포트로 전달**하도록 설정한다. 이후에는 공유기의 공인 IP를 통해 SSH 접속이 가능해진다.

```bash
ssh marsboy@<공유기 공인 IP>
```

이 방식은 동작은 하지만 두 가지 문제가 있다. 첫째, 가정용 인터넷의 공인 IP는 유동적이라 언제든 바뀔 수 있다. 둘째, SSH 포트를 인터넷에 직접 노출하면 몇 시간 안에 로그인 시도 로그가 쌓이기 시작한다. 장기적으로는 Tailscale 같은 VPN으로 대체하는 것이 좋고, 실제로 이 클러스터도 그렇게 바꿨다. 그 전까지 임시로 사용한다면 최소한 비밀번호 로그인을 끄고 키 인증만 허용해 두자.

```bash
# /etc/ssh/sshd_config.d/hardening.conf
PasswordAuthentication no
PermitRootLogin no
```

```bash
sudo systemctl restart ssh
```

#### 다른 라즈베리파이로의 접근

![홈서버 SSH 접근 경로](images/homeserver-ssh-access-diagram.png)

포트 포워딩을 통해 첫 번째 라즈베리파이(192.168.0.128)에 접속한 후, 해당 호스트에서 같은 내부 네트워크에 있는 다른 라즈베리파이로 SSH 연결할 수 있다. 이미 홈서버 네트워크 안에 있으므로 사설 IP만으로 접근이 가능하다. 즉 첫 번째 노드가 일종의 bastion 역할을 하는 셈이다.

```bash
# 192.168.0.128에 접속한 상태에서
ssh marsboy@192.168.0.129
```

매번 두 번 접속하는 대신 노트북의 `~/.ssh/config`에 **ProxyJump**를 설정해 두면, 워커 노드에 한 번에 들어갈 수 있다. `scp`나 `kubectl`이 사용하는 SSH 터널에도 그대로 적용된다.

```
# ~/.ssh/config
Host k3s-master
    HostName 192.168.0.128
    User marsboy

Host k3s-worker1
    HostName 192.168.0.129
    User marsboy
    ProxyJump k3s-master
```

```bash
ssh k3s-worker1   # 노트북에서 바로 워커 노드로
```

### 2.5 무선랜 절전 모드 끄기

무선으로 구성했다면 한 가지를 더 해야 한다. 라즈베리파이의 Wi-Fi 드라이버는 기본적으로 **절전 모드**가 켜져 있어, 트래픽이 없으면 무선 칩이 잠들었다가 깨어난다. 데스크톱 용도라면 문제가 없지만 서버에서는 SSH 세션이 간헐적으로 멈추고, K3s 노드가 `NotReady`로 깜빡이는 원인이 된다.

```bash
# 현재 상태 확인
iw dev wlan0 get power_save

# NetworkManager 설정으로 영구 비활성화 (2 = disable)
sudo nmcli connection modify "preconfigured" wifi.powersave 2
sudo nmcli connection up "preconfigured"
```

유선을 쓴다면 이 단계는 건너뛰어도 된다.

---

## 3. K3s 클러스터 구축

![K3s 로고](images/k3s-logo.png)

> *Lightweight Kubernetes. Easy to install, half the memory, all in a binary of less than 100MB.*

호스트가 준비되었으니 이제 그 위에 쿠버네티스를 올린다. 라즈베리파이 같은 저사양 장비에서 kubeadm으로 정식 쿠버네티스를 구성하는 것도 가능하지만, 메모리와 설치 복잡도 면에서 K3s가 훨씬 현실적이다.

### 3.1 K3s란?

K3s는 쿠버네티스와 완전히 호환되면서도 가볍게 사용할 수 있도록 설계된 경량 쿠버네티스 배포판이다. Kubernetes가 10글자이기 때문에 절반으로 줄여 K3s라고 이름 붙였다고 한다. CNCF 인증을 받은 정식 배포판이므로, K3s에서 배운 것은 그대로 EKS나 GKE에서도 통한다.

![K3s 아키텍처](images/k3s-architecture.svg)

K3s는 위와 같은 구조를 가진다. 정식 쿠버네티스와의 차이는 크게 세 가지이다.

1. **단일 바이너리**: API 서버, 스케줄러, 컨트롤러 매니저, kubelet, kube-proxy가 하나의 프로세스로 실행된다. 노드 역할은 `server`(컨트롤 플레인)와 `agent`(워커)로 나뉜다.
2. **가벼운 저장소**: etcd 대신 기본값으로 **SQLite**를 사용한다. 서버 노드가 한 대인 소규모 클러스터에서는 충분하며, 고가용성이 필요하면 embedded etcd로 전환할 수 있다.
3. **기본 내장 컴포넌트**: 정식 쿠버네티스에서는 직접 설치해야 하는 것들이 처음부터 들어 있다.

| 내장 컴포넌트 | 역할 | 비고 |
|---|---|---|
| **containerd** | 컨테이너 런타임 | Docker 설치 불필요 |
| **Flannel** | CNI (Pod 네트워크, VXLAN) | 다른 CNI로 교체 가능 |
| **CoreDNS** | 클러스터 DNS | |
| **Traefik** | Ingress Controller | `--disable traefik`으로 제외 가능 |
| **ServiceLB** (Klipper) | `LoadBalancer` 타입 Service 구현 | 노드 포트를 직접 점유 |
| **local-path-provisioner** | 기본 StorageClass | 노드 로컬 디스크 사용 |
| **metrics-server** | `kubectl top` 지원 | |

이 표를 미리 알아두면 설치 옵션에서 무엇을 끄고 무엇을 남길지 판단할 수 있다.

한 가지 더 알아둘 차이가 있다. kubeadm으로 구성한 클러스터는 컨트롤 플레인 노드에 `NoSchedule` taint를 붙여 일반 Pod가 올라가지 못하게 하지만, **K3s의 서버 노드는 기본적으로 워커 역할도 겸한다**. 라즈베리파이 두 대짜리 클러스터에서는 한 대를 놀리기 아까우므로 이 기본값이 오히려 반갑다. 노드가 충분해져서 컨트롤 플레인을 보호하고 싶어지면 서버 설정에 `node-taint`를 추가하면 된다.

#### 컨테이너 런타임은 containerd

"쿠버네티스를 쓰려면 Docker를 먼저 설치해야 한다"는 인식이 아직 남아 있지만, 지금은 그렇지 않다. 쿠버네티스는 1.24부터 kubelet이 Docker 데몬과 직접 통신하던 **dockershim**을 제거했고, 이후로는 CRI(Container Runtime Interface)를 구현한 런타임만 사용한다. 그 표준 자리를 차지한 것이 **containerd**이다. K3s도 처음부터 containerd를 내장하고 있어, 별도의 런타임 설치 없이 바로 Pod를 실행할 수 있다.

Docker로 빌드해 Docker Hub에 올린 이미지는 OCI 표준 이미지이므로 containerd가 그대로 내려받아 실행한다. 이미지를 만드는 도구와 실행하는 런타임이 달라도 문제가 없다는 뜻이다. 이 글에서는 containerd 기본 구성으로 진행한다.

노드에서 컨테이너를 직접 확인하고 싶을 때는 `docker ps` 대신 K3s에 포함된 `crictl`을 쓰면 된다.

```bash
sudo k3s crictl ps
sudo k3s crictl images
```

### 3.2 노드 공통 준비: cgroup 설정

K3s는 메모리 cgroup이 활성화되어 있어야 한다. Raspberry Pi OS는 기본적으로 꺼져 있으므로 **모든 노드**에서 커널 부트 파라미터에 추가하고 재부팅한다.

```bash
# Bookworm 이후: /boot/firmware/cmdline.txt
# Bullseye 이전: /boot/cmdline.txt
sudo sed -i '$ s/$/ cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
sudo reboot
```

`cmdline.txt`는 **한 줄**이어야 한다. `echo >>`로 추가하면 줄바꿈이 들어가 부팅 시 무시되는 경우가 있어 `sed`로 기존 줄 끝에 이어 붙였다. 재부팅 후 다음 명령으로 확인한다.

```bash
cat /proc/cgroups | grep memory
# memory  0  ...  1   <- 마지막 열이 1이면 활성화
```

이어서 K3s 설치에 필요한 패키지를 넣는다. 나중에 NFS 볼륨을 붙일 가능성을 생각해 `nfs-common`도 함께 설치했다.

```bash
sudo apt install -y curl nfs-common
```

### 3.3 서버 노드 설치

K3s는 설치 옵션을 커맨드라인 플래그 대신 `/etc/rancher/k3s/config.yaml`에 적어둘 수 있다. 플래그를 길게 나열하는 것보다 파일로 남겨두는 편이 나중에 "이 클러스터를 어떤 옵션으로 만들었더라"를 확인하기 쉽다.

```yaml
# /etc/rancher/k3s/config.yaml (server)
node-name: k3s-master
write-kubeconfig-mode: "0644"
tls-san:
  - 192.168.0.128
disable:
  - servicelb
```

각 옵션의 의미는 다음과 같다.

- `node-name`: 클러스터에 표시될 노드 이름이다. 호스트 이름과 맞춰 두면 혼동이 없다.
- `write-kubeconfig-mode`: 기본값은 root만 읽을 수 있는 `0600`이라 일반 사용자로 `kubectl`을 쓸 수 없다. 개인 클러스터이므로 `0644`로 완화했다.
- `tls-san`: API 서버 인증서에 포함할 추가 주소이다. 노트북에서 어떤 주소로 API 서버에 접속하든, 그 주소가 여기 들어 있어야 인증서 검증을 통과한다. 나중에 VPN IP나 도메인으로 접속할 계획이 있다면 그때 추가하고 K3s를 재시작하면 된다.
- `disable: servicelb`: ServiceLB는 `LoadBalancer` 타입 Service를 만들 때 **노드의 실제 포트를 점유**하는 방식으로 동작한다. 처음에는 "왜 80 포트가 이미 사용 중이지"라는 혼란을 주기 쉬워 껐다. 나중에 Istio Gateway를 올리면서 MetalLB로 대체했는데, 그 이야기는 [Ingress에서 Istio Gateway까지]({{< ref "/posts/kubernetes-networking-ingress-to-istio" >}})에서 다룬다. 단순히 NodePort로 시작할 계획이라면 켜 두어도 무방하다.

설정 파일을 만들었으면 설치 스크립트를 실행한다.

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<'EOF'
node-name: k3s-master
write-kubeconfig-mode: "0644"
tls-san:
  - 192.168.0.128
disable:
  - servicelb
EOF

curl -sfL https://get.k3s.io | sh -
```

설치가 끝나면 K3s가 systemd 서비스로 등록되어 부팅 시 자동 시작된다. `kubectl`도 함께 설치되며, kubeconfig는 `/etc/rancher/k3s/k3s.yaml`에 생성된다. 매번 `--kubeconfig`를 붙이지 않도록 환경 변수로 지정해 둔다.

```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc

kubectl get nodes
kubectl get pods -A
```

`kube-system` 네임스페이스에 CoreDNS, Traefik, local-path-provisioner, metrics-server Pod가 `Running`이 되면 서버 노드는 준비된 것이다. 라즈베리파이 4에서는 모두 뜨기까지 1~2분 정도 걸린다. 무언가 뜨지 않는다면 K3s 자체의 로그를 본다. 쿠버네티스 컴포넌트가 한 프로세스에 들어 있으므로 로그도 한 곳에 모인다.

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -f
```

#### 버전 고정

위 명령은 설치 시점의 최신 stable 버전을 받아 온다. 학습용이라면 상관없지만, 워커 노드를 나중에 추가하거나 노드를 다시 설치할 때 서버와 다른 버전이 잡히면 곤란하다. 쿠버네티스는 워커가 서버보다 마이너 버전이 높은 것을 허용하지 않는다. 채널이나 정확한 버전을 지정해 두면 이런 일을 피할 수 있다.

```bash
# 마이너 버전 채널 고정
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.31 sh -

# 정확한 버전 고정
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.4+k3s1 sh -
```

업그레이드는 같은 스크립트를 새 버전으로 다시 실행하면 된다. 바이너리만 교체되고 `config.yaml`과 클러스터 데이터는 그대로 남는다. 순서는 **서버 노드 먼저, 그다음 워커 노드**이다.

워커 노드 설치를 위해 서버 노드의 토큰을 확인해 둔다.

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
# K10xxxx...::server:xxxx
```

### 3.4 워커 노드 설치

워커 노드에서는 앞서 한 cgroup 설정과 패키지 설치가 끝났다는 전제에서, 서버 주소와 토큰을 지정하여 agent 모드로 설치한다.

```bash
# 서버 노드에서 확인한 값으로 치환
export K3S_URL=https://192.168.0.128:6443
export K3S_TOKEN=K10xxxx...::server:xxxx

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent --node-name k3s-worker1" sh -
```

설치가 완료되면 서버 노드에서 클러스터 상태를 확인한다.

```bash
kubectl get nodes -o wide
```

![kubectl get nodes 결과 - master와 worker 노드가 Ready 상태](images/kubectl-get-nodes-result.png)

두 노드가 모두 **Ready** 상태로 표시되면 K3s 클러스터 구축이 완료된 것이다. `-o wide`를 붙이면 각 노드의 `OS-IMAGE`, `KERNEL-VERSION`, `CONTAINER-RUNTIME` 열에서 containerd가 런타임으로 잡혀 있는 것도 확인할 수 있다.

워커 노드의 `ROLES` 열은 `<none>`으로 비어 있다. 쿠버네티스는 서버 노드에만 `control-plane` 라벨을 자동으로 붙이고, 워커의 역할은 사용자에게 맡기기 때문이다. 라벨을 하나 달아 두면 출력이 읽기 쉬워지고, 나중에 `nodeSelector`로 워크로드를 워커에만 배치할 때도 쓸 수 있다.

```bash
kubectl label node k3s-worker1 node-role.kubernetes.io/worker=worker
kubectl get nodes
# NAME          STATUS   ROLES                  AGE   VERSION
# k3s-master    Ready    control-plane,master   20m   v1.31.4+k3s1
# k3s-worker1   Ready    worker                 3m    v1.31.4+k3s1
```

#### 워커가 조인되지 않을 때

워커 노드가 `kubectl get nodes`에 나타나지 않거나 `NotReady`에 머문다면, 워커에서 `sudo journalctl -u k3s-agent -f`로 로그를 보면서 다음을 순서대로 확인한다. 실제로 겪어 본 원인은 대부분 이 넷 중 하나였다.

| 증상 | 원인 | 확인 방법 |
|---|---|---|
| `401 Unauthorized` | 토큰 불일치 | 서버의 `node-token`을 다시 복사했는지 확인 |
| `connection refused` / 타임아웃 | 서버 IP나 6443 포트 접근 불가 | 워커에서 `curl -k https://192.168.0.128:6443` |
| 조인은 되나 Pod 간 통신 실패 | Flannel VXLAN(UDP 8472) 차단 | 방화벽(ufw 등)을 켰다면 아래 포트 허용 |
| 노드가 `Ready`와 `NotReady`를 오감 | 무선랜 절전 모드 | 2.5의 설정을 워커에도 적용했는지 확인 |

Raspberry Pi OS는 기본적으로 방화벽이 꺼져 있어 포트 문제는 드물지만, 보안을 위해 `ufw`를 켰다면 K3s가 사용하는 포트를 열어야 한다.

| 포트 | 프로토콜 | 방향 | 용도 |
|---|---|---|---|
| 6443 | TCP | 워커 → 서버 | Kubernetes API |
| 8472 | UDP | 모든 노드 간 | Flannel VXLAN |
| 10250 | TCP | 모든 노드 간 | kubelet (로그, `exec`, 메트릭) |

노드를 더 추가하고 싶다면 같은 절차를 반복하면 된다. 실제로 이 클러스터는 이후 워커 노드가 세 대까지 늘어났고, [MTU 문제 해결기]({{< ref "/posts/tailscale-mtu-troubleshooting" >}})에서 등장하는 `k3s-worker2`, `k3s-worker3`가 그 노드들이다.

### 3.5 노트북에서 kubectl 사용하기

매번 서버 노드에 SSH로 들어가 `kubectl`을 치는 것은 불편하다. kubeconfig를 노트북으로 가져와 사용한다.

```bash
# 노트북에서 실행
scp marsboy@192.168.0.128:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-home.yaml

# server 주소를 127.0.0.1에서 서버 노드 IP로 변경
sed -i '' 's/127.0.0.1/192.168.0.128/' ~/.kube/k3s-home.yaml

export KUBECONFIG=~/.kube/k3s-home.yaml
kubectl get nodes
```

`k3s.yaml`의 `server` 필드는 기본적으로 `https://127.0.0.1:6443`으로 되어 있으므로 반드시 서버 노드 주소로 바꿔야 한다. 이 주소가 앞서 `tls-san`에 넣어둔 값과 일치해야 인증서 검증을 통과한다. 이 구성은 같은 Wi-Fi 안에서만 동작한다. 집 밖에서도 쓰려면 앞서 말한 VPN이 필요하다.

### 3.6 잘못됐을 때: 깨끗하게 지우기

K3s의 장점 중 하나는 제거가 쉽다는 점이다. 설치 옵션을 바꾸고 싶거나 클러스터가 꼬였을 때, 설치 스크립트가 함께 만들어 둔 제거 스크립트를 실행하면 처음 상태로 돌아간다.

```bash
# 서버 노드
/usr/local/bin/k3s-uninstall.sh

# 워커 노드
/usr/local/bin/k3s-agent-uninstall.sh
```

처음 구축할 때 이 스크립트를 대여섯 번은 실행했다. 실험 비용이 낮다는 것이 홈서버의 가장 큰 장점이므로 주저하지 말고 지우고 다시 만들어 보자.

### 3.7 첫 워크로드로 검증하기

노드가 `Ready`라는 것은 kubelet이 API 서버와 통신하고 있다는 뜻일 뿐, Pod가 실제로 뜨고 밖에서 접근되는지까지 보장하지는 않는다. 클러스터 구축의 마지막 단계로 가장 단순한 워크로드를 하나 올려 전체 경로를 확인한다.

```bash
kubectl create deployment hello --image=nginx --replicas=2
kubectl expose deployment hello --type=NodePort --port=80
kubectl get pods -o wide
kubectl get svc hello
```

`-o wide`로 보면 두 Pod가 서버와 워커에 하나씩 나뉘어 스케줄된 것을 볼 수 있다. 앞서 말한 대로 서버 노드도 워크로드를 받는다. `nginx` 공식 이미지는 `arm64`를 포함한 멀티 아키텍처 이미지라 라즈베리파이에서 그대로 실행된다. 직접 만든 이미지를 올릴 때는 이 점을 반드시 확인해야 하는데, amd64 노트북에서 빌드한 이미지는 라즈베리파이에서 `exec format error`로 죽는다. 이 문제는 CI/CD 파이프라인을 다루는 다음 글에서 이어서 다룰 예정이다.

`NodePort` 타입 Service는 모든 노드의 같은 포트를 연다. 어느 노드의 IP로 접근하든 Service를 거쳐 Pod에 도달하며, Pod가 없는 노드로 들어가도 Flannel이 다른 노드로 넘겨준다.

```bash
# 노트북에서 실행. 포트 번호는 kubectl get svc 출력의 30000번대 값
curl -s http://192.168.0.128:31234 | grep title
curl -s http://192.168.0.129:31234 | grep title
# <title>Welcome to nginx!</title>
```

두 노드 모두에서 응답이 오면 스케줄링, 이미지 풀, Pod 네트워크, Service 라우팅이 전부 동작하는 것이다. 마지막으로 내장된 metrics-server로 노드에 여유가 얼마나 남았는지 확인해 두자. 라즈베리파이 4에서 K3s 서버 노드는 아무것도 올리지 않은 상태에서 메모리를 600~700MB 정도 쓴다.

```bash
kubectl top nodes
# NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# k3s-master    186m         4%     712Mi           18%
# k3s-worker1   62m          1%     318Mi           8%
```

확인이 끝났으면 정리한다.

```bash
kubectl delete svc,deployment hello
```

---

## 마치며

이 글에서는 AWS 프리티어 종료를 계기로, 라즈베리파이 위에 K3s 클러스터를 구축하는 과정을 다루었다.

1. **하드웨어와 OS** — 모델과 저장 장치 선택, Raspberry Pi OS Lite 설치, 첫 부팅 후 시간대와 스왑 설정
2. **네트워크** — 고정 IP, 포트 포워딩과 SSH 강화, ProxyJump, 무선랜 절전 모드
3. **K3s 클러스터** — containerd 기반 서버/워커 노드 설치, 설정 파일과 버전 관리, 조인 트러블슈팅, 노트북에서의 `kubectl`, 첫 워크로드 검증

처음 이 글을 썼을 때는 "클러스터가 떴다"가 목표였다. 지금 돌아보면 클러스터를 띄우는 데 걸린 시간보다, 그 뒤에 만난 문제들에서 배운 것이 더 많았다. 클라우드였다면 만날 일이 없었을 문제들이고, 그래서 관리형 서비스가 어떤 일을 대신해 주고 있었는지를 몸으로 알게 되었다. 비용 면에서도 초기 라즈베리파이 구매 비용 외에는 거의 들지 않기 때문에, 인프라를 밑바닥부터 배우고 싶은 사람에게는 여전히 가장 좋은 선택지라고 생각한다.

클러스터가 떴다면 이제 그 위에 무언가를 올릴 차례이다. 이 클러스터는 이후 CI/CD와 GitOps, VPN을 통한 원격 접속, 서비스 메시, 모니터링을 하나씩 얹으면서 계속 모습을 바꿔 왔다. 배포 방식은 [GitOps란 무엇인가]({{< ref "/posts/about-gitops" >}})에서, 외부 서비스와 연결하며 만난 네트워크 문제는 [Tailscale 터널에서 Redis 헬스체크가 실패하는 이유]({{< ref "/posts/tailscale-mtu-troubleshooting" >}})에서 이어진다.

### References

- [Raspberry Pi 공식 사이트](https://www.raspberrypi.com/software/)
- [Raspberry Pi OS Bookworm 릴리스 노트 — NetworkManager 전환](https://www.raspberrypi.com/news/bookworm-the-new-version-of-raspberry-pi-os/)
- [K3s 공식 문서 — Requirements (Raspberry Pi)](https://docs.k3s.io/installation/requirements#raspberry-pi)
- [K3s 공식 문서 — Configuration File](https://docs.k3s.io/installation/configuration#configuration-file)
- [K3s 공식 문서 — Networking Services (ServiceLB, Traefik)](https://docs.k3s.io/networking/networking-services)
- [Kubernetes — Dockershim removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)
- [core_kubernetes — 쿠버네티스 설치](https://github.com/bjpublic/core_kubernetes/tree/master/chapters/03)
