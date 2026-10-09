---
title: "Raspberry PiでK3sホームサーバーを構築する"
date: 2026-09-13
draft: false
tags: ["kubernetes", "k3s", "raspberry-pi", "homelab"]
translationKey: "raspberry-pi-k8s-homeserver"
summary: "AWS無料枠の終了をきっかけに、Raspberry PiにRaspberry Pi OSを載せ、ネットワークを整え、K3sでKubernetesクラスタを構築するまでの過程を扱います。"
---

![Raspberry Piのロゴ](images/raspberry-pi-logo.png)

AWSを通じてさまざまなインフラサービスに触れてから、そろそろ一年が経とうとしていた頃、無料枠の期間が終わりに近づいていました。無料枠が終了すると毎月固定の費用が発生しますが、学習目的でインフラを運用するには負担の大きい金額でした。そこで、机の上に小さなコンピュータを何台か置いて、Kubernetesクラスタを自分で作ってみることにしました。

この記事はその出発点の記録です。Raspberry PiにOSを載せ、ネットワークにつなぎ、K3sでクラスタを構成して最初のワークロードが応答する瞬間までを扱います。最初に記事を書いてからもこのクラスタを運用し続けてきたので、当時は分からなかったけれど今なら最初から違うやり方にしていた部分も一緒にまとめました。Raspberry Pi OSがBookwormに移行して変わった設定パスが代表例です。

---

## 1. 動機: なぜオンプレミスなのか?

無料枠が終わる時点で選択肢は二つありました。費用を払ってクラウドに残るか、自分でサーバーを置くかです。

| | クラウド (AWS) | オンプレミス (Raspberry Pi) |
|---|---|---|
| **初期費用** | なし (無料枠) | Raspberry Piの購入費用 |
| **維持費用** | 月額課金 (無料枠終了後) | 電気代 (ごくわずか) |
| **学習範囲** | マネージドサービス中心 | ネットワーク、OS、クラスタ全般 |
| **柔軟性** | 高い | 物理的な制約あり |
| **障害時の責任** | クラウド事業者 | すべて自分 |

ちょうど学生寮に住んでいて光熱費を払わない状況だったので、Raspberry Piさえあれば事実上無料でオンプレミス環境を構築できました。費用よりも大きな理由は学習範囲でした。マネージドサービスは便利な分だけ多くのものを隠します。ノードがどうやってネットワークにつながるのか、コンテナランタイムが何なのか、クラスタの外から入ってくるトラフィックがどの経路でPodまで到達するのかは、EKSのコンソールからは見えません。Raspberry Piの上ではそのすべてを自分の手で触ることになります。

最後の行はあとから追加したものです。停電が起きればクラスタは落ち、ルーターが再起動すればIPは変わり、SDカードが寿命を迎えればノードは消えます。これらすべてが学習の一部であることは確かですが、何かを本気でサービスとして運用する計画があるなら、あらかじめ知ったうえで始めるほうがよいでしょう。

「電気代がごくわずか」という話も数字で確認しておきましょう。Raspberry Pi 4はidle状態で約3W、負荷時に6〜7Wを消費します。二台を24時間つけっぱなしにしても多めに見積もって月10kWh前後で、韓国の家庭用料金に換算すると一か月で2,000ウォンにもなりません。AWSで同程度のスペックの `t4g.small` インスタンスを二台、無料枠なしで運用すると月3万ウォン近くかかります。初期の購入費用は数か月あれば回収できます。

### 1.1 ハードウェアの選択

最初に構築したときは、手元にあった **Raspberry Pi 4**（4GB RAM）二台から始めました。もう一度選ぶなら、次の基準で見るはずです。

- **モデルとメモリ**: K3sのサーバーノードはidle状態でも500MB以上を使います。4GBあれば小規模なクラスタには十分ですが、モニタリングスタックまで載せる計画があるなら8GBモデルが楽です。Raspberry Pi 5は4に比べてCPU性能が2倍以上あり、PCIeでNVMe SSDを直接接続できるので、新しく買うなら5をおすすめします。
- **ストレージ**: SDカードは最初に壊れる部品です。Kubernetesは状態ストアへの書き込み、コンテナイメージのレイヤ、ログで絶えずディスクを叩きます。Raspberry Pi 4はUSB 3.0 SSDからのブートを、5はNVMe HATをサポートしているので、せめてサーバーノードだけでもSSDを使うのがよいでしょう。この記事ではSDカードで進めます。
- **電源**: Raspberry Piは電源が不安定だと低電圧警告とともに性能を下げたり再起動したりします。純正アダプタまたはUSB PD対応のアダプタを使い、USB SSDをつなぐなら消費電力がさらに大きくなる点を考慮する必要があります。
- **冷却**: ヒートシンクなしで長時間負荷をかけると80°C付近でスロットリングがかかります。ファン付きケース、少なくともヒートシンクは付けておくほうがよいです。
- **ネットワーク**: 可能なら有線LANのほうが性能と安定性の面で有利です。特にノード間の通信が頻繁なK8s環境ではなおさらです。ただし今回の構築はサービス用途ではなく学習目的なので、無線LANで進めました。無線を使う場合は、後述する省電力モードの問題を必ず確認しなければなりません。

電源と冷却が十分かどうかは、OSを載せたあとにRaspberry Piのファームウェアが提供するコマンドで確認できます。クラスタが理由もなく遅かったり、ノードが再起動したりするときに最初に見るべき場所です。

```bash
vcgencmd measure_temp        # temp=52.1'C
vcgencmd get_throttled       # throttled=0x0 なら正常
```

`get_throttled` が `0x0` でなければ、低電圧や過熱でスロットリングがかかっている、あるいはかかったことがあるという意味です。`0x50000` のように後ろのビットだけが立っていれば過去に発生して今は解消された状態で、`0x50005` のように前のビットまで立っていれば今も進行中です。アダプタを替えたりヒートシンクを付けたりしたあと、この値が `0x0` に戻るかを見ればよいでしょう。

---

## 2. Raspberry Pi OSの設定

ハードウェアを選んだら、次はOSを載せてネットワークにつなぐ番です。この章はRaspberry Piを「ネットワークに接続されたLinuxホスト」にする過程です。

### 2.1 インフラ構成

![ホームサーバーのネットワーク構成図](images/homeserver-network-architecture.png)

家庭用ルーターを通じて上のような構成でネットワークを組みます。iptime（韓国のルーターメーカー）のルーターを使っていますが、大規模なトラフィックを処理するわけではないので、別途ロードバランサを置かずルーターの設定だけで十分です。ルーターがDHCPサーバーとNATの役割を兼ね、Raspberry Piたちはその下の `192.168.0.0/24` 帯域に置かれます。

### 2.2 OSのインストール

まずはRaspberry PiにOSをインストールします。OSのインストールは、SDカード（またはSSD）にOSイメージを書き込み、そのストレージからRaspberry Piを起動する方式です。

Raspberry Piの公式サイトからImagerをダウンロードします。

- **Raspberry Pi Imager** : [https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/)

![Raspberry Pi Imagerの実行画面](images/raspberry-pi-imager.png)

Imagerを実行すると上のような画面が表示されます。ストレージには挿入したSDカードを選び、OSには **Raspberry Pi OS Lite**（64-bit）を選びます。サーバーとしてのみ使うのでデスクトップ環境の入っていないLite版で十分ですし、メモリもその分節約できます。64-bitを選ぶべき理由はコンテナイメージにあります。ほとんどのイメージが `arm64` で配布されており、32-bit（`armv7`）のイメージは次第に姿を消しつつあります。

> **注意**: Raspberry Pi用のUbuntuにK3sをインストールすると、カーネルモジュールやcgroup設定の違いでエラーが発生することがあります。Raspberry Pi OSが最も試行錯誤が少なくて済みました。どうしてもUbuntuを使いたい場合は、K3s公式ドキュメントの [Raspberry Piの要件](https://docs.k3s.io/installation/requirements#raspberry-pi) に書かれている追加パッケージ（`linux-modules-extra-raspi`）を先にインストールする必要があります。

OSとSDカードを選んだあと、設定オプションで **SSHの許可** と **無線LANの設定** を必ず行っておかなければなりません。モニタもキーボードもなしで運用するので、この段階でSSHを開けておかないと起動後に接続する手段がありません。

![Imagerの詳細オプション - SSHおよびユーザー設定](images/raspberry-pi-imager-ssh-settings.png)

この画面でSSHを許可し、接続に使うユーザー名とパスワードを設定します。以前はデフォルトで `pi` / `raspberry` が作られていましたが、Bullseye以降はセキュリティ上の理由でデフォルトアカウントがなくなったので、必ず自分で作る必要があります。ホスト名もここで `k3s-master`、`k3s-worker1` のように役割が分かるように決めておくと、あとで `kubectl get nodes` の出力からすぐ見分けられます。無線LANも一緒に設定して、ホームサーバーのルーターのWi-Fi情報を入力しておきます。

可能であればパスワードの代わりに **公開鍵認証** を選び、自分のSSH公開鍵を登録します。あとでポートフォワーディングによってSSHを外部に公開した瞬間から、パスワードログインは総当たり攻撃の標的になります。

### 2.3 初回起動後の基本設定

SDカードにイメージを書き込んでRaspberry Piに挿入し起動すると、Raspberry PiにOSがインストールされます。これでこの小さなコンピュータはネットワークに接続された一つのホストになりました。本格的な設定の前に、すべてのノードで共通してやっておくとよいことがあります。

まず64-bitカーネルで起動しているかを確認します。32-bitイメージを誤って選んでいた場合、あとで `arm64` のコンテナイメージが実行できなくなるので、ここで捕まえるのが一番安上がりです。

```bash
uname -m
# aarch64
```

タイムゾーンとパッケージを揃えます。Kubernetesは証明書の有効期限やノード間の通信に時刻を使うので、ノードの時計がずれていると原因の分かりにくい問題が起こります。

```bash
sudo timedatectl set-timezone Asia/Seoul
timedatectl                       # NTP service: active を確認

sudo apt update && sudo apt full-upgrade -y
```

最後に **スワップを切ります**。Raspberry Pi OSはデフォルトでSDカード上にスワップファイルを作りますが、Kubernetesはメモリ不足をスワップで耐えるのではなく、Podを落として再スケジュールする方式を前提に設計されています。スワップが有効だとノードが遅くなるだけで問題が表に出てこず、SDカードの寿命を削るだけです。K3sはスワップがあってもインストールを拒否はしませんが、切っておくほうが挙動を予測しやすくなります。

```bash
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo systemctl disable --now dphys-swapfile

free -h                           # Swapの行が0Bなら完了
```

### 2.4 SSH接続の設定

ホストが準備できたので、次は外から入る道を作ります。同じネットワーク、別のネットワーク、そしてノード同士という三つの経路を順に扱います。

#### プライベートIPの確認と固定

ルーターの管理ページ（`192.168.0.1`）にアクセスし、**高度な設定 > ネットワーク管理 > DHCPサーバー設定** に移動します。

![iptimeのDHCPサーバー設定画面](images/iptime-dhcp-settings.png)

**DHCP**（Dynamic Host Configuration Protocol）は、ネットワークに接続したホストにIPアドレスを動的に割り当てるプロトコルです。問題は、接続するたびに違うIPが割り当てられる可能性がある点です。サーバーとして使うRaspberry PiのIPは必ず固定しなければなりません。特にK3sはワーカーノードがサーバーノードのIPへ接続する設定になるので、サーバーノードのIPが変わるとクラスタが丸ごと壊れます。

IPを固定する方法は二つあります。ルーター側でMACアドレスを基準に常に同じIPを割り当てる **DHCP固定割り当て** と、Raspberry Pi自体に **静的IP** を設定する方法です。どちらか一方だけでも構いませんが、ルーター側で固定するほうが管理ポイントが一か所にまとまって楽です。**使用中のIPアドレス情報** のセクションでRaspberry PiのIPをクリックし、**手動登録** ボタンを押してIPを固定します。

Raspberry Pi側で静的IPを与えたい場合は注意点があります。古い資料には `/etc/dhcpcd.conf` を編集するように書かれていますが、**Raspberry Pi OS Bookworm（2023年10月以降）からはNetworkManagerがネットワークを管理する** ため、そのファイルはもう読まれません。`nmcli` で設定します。

```bash
# 接続名の確認 (Imagerで設定したWi-Fiは通常 "preconfigured")
nmcli connection show

# 静的IP、ゲートウェイ、DNSの設定
sudo nmcli connection modify "preconfigured" \
  ipv4.method manual \
  ipv4.addresses 192.168.0.128/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns "1.1.1.1 8.8.8.8"

sudo nmcli connection up "preconfigured"
```

`ipv4.gateway` にはルーターのアドレス（`192.168.0.1`）を入れます。有線なら接続名は `Wired connection 1` です。

#### 同じネットワークからの接続

同じWi-Fiを使う環境では、プライベートIPで直接SSH接続ができます。

```bash
ssh marsboy@192.168.0.128
```

パスワードまたは登録した鍵でRaspberry Piに問題なくアクセスできます。

#### 別のネットワークからの接続 (ポートフォワーディング)

外部のネットワークから接続するには、ルーターに **ポートフォワーディング** の設定が必要です。

![ポートフォワーディングの構成図 - ルーターを境に内部/外部ネットワークを分離](images/port-forwarding-diagram.png)

ルーター内部のホストはDHCPによって `192.168.0.*` 帯域のプライベートIPを割り当てられます。しかし外部からはこのプライベートIPを直接識別できません。そのため、ルーターのグローバルIPを通じて入ってくる特定ポートの接続を内部のプライベートIPへ転送する **ポートフォワーディング** の設定が必要になります。

具体的には、**外部の22番ポートに入ってくる接続を内部の192.168.0.128の22番ポートへ転送する** ように設定します。これ以降はルーターのグローバルIPを通じてSSH接続ができるようになります。

```bash
ssh marsboy@<ルーターのグローバルIP>
```

この方式は動作はしますが、二つの問題があります。第一に、家庭用インターネットのグローバルIPは動的なのでいつでも変わり得ます。第二に、SSHポートをインターネットへ直接公開すると、数時間のうちにログイン試行のログが溜まり始めます。長期的にはTailscaleのようなVPNに置き換えるのがよく、実際このクラスタもそのように変えました。それまで一時的に使うのであれば、少なくともパスワードログインを切って鍵認証だけを許可しておきましょう。

```bash
# /etc/ssh/sshd_config.d/hardening.conf
PasswordAuthentication no
PermitRootLogin no
```

```bash
sudo systemctl restart ssh
```

#### 別のRaspberry Piへのアクセス

![ホームサーバーのSSHアクセス経路](images/homeserver-ssh-access-diagram.png)

ポートフォワーディングを通じて一台目のRaspberry Pi（192.168.0.128）に接続したあと、そのホストから同じ内部ネットワークにある別のRaspberry PiへSSH接続できます。すでにホームサーバーのネットワークの中にいるので、プライベートIPだけでアクセスできます。つまり一台目のノードが一種のbastionの役割を果たすわけです。

```bash
# 192.168.0.128に接続した状態で
ssh marsboy@192.168.0.129
```

毎回二度接続する代わりに、ノートPCの `~/.ssh/config` に **ProxyJump** を設定しておけば、ワーカーノードへ一度で入れます。`scp` や `kubectl` が使うSSHトンネルにもそのまま適用されます。

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
ssh k3s-worker1   # ノートPCから直接ワーカーノードへ
```

### 2.5 無線LANの省電力モードを切る

無線で構成したなら、もう一つやるべきことがあります。Raspberry PiのWi-Fiドライバはデフォルトで **省電力モード** が有効になっていて、トラフィックがないと無線チップが眠り、また目を覚まします。デスクトップ用途なら問題ありませんが、サーバーではSSHセッションが断続的に止まったり、K3sのノードが `NotReady` に点滅したりする原因になります。

```bash
# 現在の状態を確認
iw dev wlan0 get power_save

# NetworkManagerの設定で恒久的に無効化 (2 = disable)
sudo nmcli connection modify "preconfigured" wifi.powersave 2
sudo nmcli connection up "preconfigured"
```

有線を使うならこのステップは飛ばして構いません。

---

## 3. K3sクラスタの構築

![K3sのロゴ](images/k3s-logo.png)

> *Lightweight Kubernetes. Easy to install, half the memory, all in a binary of less than 100MB.*

ホストが準備できたので、次はその上にKubernetesを載せます。Raspberry Piのような低スペック機器でkubeadmを使って正式なKubernetesを構成することも可能ですが、メモリとインストールの複雑さの面ではK3sのほうがはるかに現実的です。

### 3.1 K3sとは?

K3sはKubernetesと完全に互換性を保ちながら軽量に使えるよう設計された、軽量Kubernetesディストリビューションです。Kubernetesが10文字なので、半分に縮めてK3sと名付けたそうです。CNCF認証を受けた正式なディストリビューションなので、K3sで学んだことはそのままEKSやGKEでも通用します。

![K3sのアーキテクチャ](images/k3s-architecture.svg)

K3sは上のような構造を持ちます。正式なKubernetesとの違いは大きく三つあります。

1. **単一バイナリ**: APIサーバー、スケジューラ、コントローラマネージャ、kubelet、kube-proxyが一つのプロセスとして実行されます。ノードの役割は `server`（コントロールプレーン）と `agent`（ワーカー）に分かれます。
2. **軽量なストア**: etcdの代わりにデフォルトで **SQLite** を使います。サーバーノードが一台の小規模なクラスタには十分で、高可用性が必要ならembedded etcdへ切り替えられます。
3. **標準で内蔵されたコンポーネント**: 正式なKubernetesでは自分でインストールする必要があるものが最初から入っています。

| 内蔵コンポーネント | 役割 | 備考 |
|---|---|---|
| **containerd** | コンテナランタイム | Dockerのインストール不要 |
| **Flannel** | CNI (Podネットワーク、VXLAN) | 別のCNIに置き換え可能 |
| **CoreDNS** | クラスタDNS | |
| **Traefik** | Ingress Controller | `--disable traefik` で除外可能 |
| **ServiceLB** (Klipper) | `LoadBalancer` タイプのService実装 | ノードのポートを直接占有 |
| **local-path-provisioner** | デフォルトのStorageClass | ノードのローカルディスクを使用 |
| **metrics-server** | `kubectl top` のサポート | |

この表をあらかじめ知っておくと、インストールオプションで何を切って何を残すかを判断できます。

もう一つ知っておきたい違いがあります。kubeadmで構成したクラスタはコントロールプレーンノードに `NoSchedule` のtaintを付けて一般のPodが載らないようにしますが、**K3sのサーバーノードはデフォルトでワーカーの役割も兼ねます**。Raspberry Pi二台のクラスタでは一台を遊ばせるのはもったいないので、このデフォルト値はむしろありがたいです。ノードが十分に増えてコントロールプレーンを保護したくなったら、サーバーの設定に `node-taint` を追加すればよいでしょう。

#### コンテナランタイムはcontainerd

「Kubernetesを使うにはまずDockerをインストールしなければならない」という認識がまだ残っていますが、今はそうではありません。Kubernetesは1.24からkubeletがDockerデーモンと直接通信していた **dockershim** を削除し、以降はCRI（Container Runtime Interface）を実装したランタイムだけを使います。その標準の座を占めたのが **containerd** です。K3sも最初からcontainerdを内蔵しているので、別途ランタイムをインストールせずにすぐPodを実行できます。

DockerでビルドしてDocker Hubに上げたイメージはOCI標準イメージなので、containerdがそのまま取得して実行します。イメージを作るツールと実行するランタイムが違っても問題ないということです。この記事ではcontainerdのデフォルト構成で進めます。

ノード上でコンテナを直接確認したいときは、`docker ps` の代わりにK3sに含まれている `crictl` を使えばよいです。

```bash
sudo k3s crictl ps
sudo k3s crictl images
```

### 3.2 ノード共通の準備: cgroupの設定

K3sはメモリcgroupが有効になっている必要があります。Raspberry Pi OSはデフォルトで無効なので、**すべてのノード** でカーネルのブートパラメータに追加して再起動します。

```bash
# Bookworm以降: /boot/firmware/cmdline.txt
# Bullseye以前: /boot/cmdline.txt
sudo sed -i '$ s/$/ cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
sudo reboot
```

`cmdline.txt` は **一行** でなければなりません。`echo >>` で追加すると改行が入って起動時に無視されることがあるので、`sed` で既存の行末に続けて書き足しました。再起動後、次のコマンドで確認します。

```bash
cat /proc/cgroups | grep memory
# memory  0  ...  1   <- 最後の列が1なら有効
```

続いてK3sのインストールに必要なパッケージを入れます。あとでNFSボリュームをつなぐ可能性を考えて `nfs-common` も一緒にインストールしました。

```bash
sudo apt install -y curl nfs-common
```

### 3.3 サーバーノードのインストール

K3sはインストールオプションをコマンドラインフラグの代わりに `/etc/rancher/k3s/config.yaml` に書いておけます。フラグを長く並べるよりファイルとして残しておくほうが、あとで「このクラスタはどのオプションで作ったんだったか」を確認しやすくなります。

```yaml
# /etc/rancher/k3s/config.yaml (server)
node-name: k3s-master
write-kubeconfig-mode: "0644"
tls-san:
  - 192.168.0.128
disable:
  - servicelb
```

各オプションの意味は次のとおりです。

- `node-name`: クラスタに表示されるノード名です。ホスト名と揃えておくと混乱がありません。
- `write-kubeconfig-mode`: デフォルト値はrootだけが読める `0600` なので、一般ユーザーで `kubectl` を使えません。個人のクラスタなので `0644` に緩めました。
- `tls-san`: APIサーバー証明書に含める追加のアドレスです。ノートPCからどのアドレスでAPIサーバーに接続するにせよ、そのアドレスがここに入っていないと証明書の検証を通りません。あとでVPNのIPやドメインで接続する予定があるなら、そのときに追加してK3sを再起動すればよいです。
- `disable: servicelb`: ServiceLBは `LoadBalancer` タイプのServiceを作るときに **ノードの実ポートを占有する** かたちで動作します。最初は「なぜ80番ポートがすでに使用中なんだ」という混乱を招きやすいので切りました。あとでIstio Gatewayを載せる際にMetalLBへ置き換えたのですが、その話は [IngressからIstio Gatewayまで]({{< ref path="/posts/kubernetes-networking-ingress-to-istio" lang="en" >}}) で扱います。単純にNodePortから始める計画なら有効のままでも差し支えありません。

設定ファイルを作ったら、インストールスクリプトを実行します。

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

インストールが終わるとK3sがsystemdサービスとして登録され、起動時に自動で開始されます。`kubectl` も一緒にインストールされ、kubeconfigは `/etc/rancher/k3s/k3s.yaml` に生成されます。毎回 `--kubeconfig` を付けなくて済むよう、環境変数で指定しておきます。

```bash
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc

kubectl get nodes
kubectl get pods -A
```

`kube-system` ネームスペースでCoreDNS、Traefik、local-path-provisioner、metrics-serverのPodが `Running` になれば、サーバーノードは準備完了です。Raspberry Pi 4ではすべて立ち上がるまで1〜2分ほどかかります。何かが立ち上がらないときはK3s自体のログを見ます。Kubernetesのコンポーネントが一つのプロセスに入っているので、ログも一か所にまとまります。

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -f
```

#### バージョンの固定

上のコマンドはインストール時点で最新のstableバージョンを取得します。学習用なら構いませんが、ワーカーノードをあとから追加したりノードを入れ直したりするときに、サーバーと違うバージョンが入ると困ります。Kubernetesはワーカーがサーバーよりマイナーバージョンが高いことを許容しません。チャンネルや正確なバージョンを指定しておけば、こうした事態を避けられます。

```bash
# マイナーバージョンのチャンネルを固定
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.31 sh -

# 正確なバージョンを固定
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.4+k3s1 sh -
```

アップグレードは同じスクリプトを新しいバージョンで再実行すればよいです。バイナリだけが入れ替わり、`config.yaml` とクラスタのデータはそのまま残ります。順序は **サーバーノードが先、そのあとワーカーノード** です。

ワーカーノードのインストールに備えて、サーバーノードのトークンを確認しておきます。

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
# K10xxxx...::server:xxxx
```

### 3.4 ワーカーノードのインストール

ワーカーノードでは、先ほど行ったcgroup設定とパッケージのインストールが済んでいる前提で、サーバーのアドレスとトークンを指定してagentモードでインストールします。

```bash
# サーバーノードで確認した値に置き換える
export K3S_URL=https://192.168.0.128:6443
export K3S_TOKEN=K10xxxx...::server:xxxx

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="agent --node-name k3s-worker1" sh -
```

インストールが完了したら、サーバーノードでクラスタの状態を確認します。

```bash
kubectl get nodes -o wide
```

![kubectl get nodesの結果 - masterとworkerノードがReady状態](images/kubectl-get-nodes-result.png)

二つのノードがどちらも **Ready** 状態で表示されれば、K3sクラスタの構築は完了です。`-o wide` を付けると、各ノードの `OS-IMAGE`、`KERNEL-VERSION`、`CONTAINER-RUNTIME` の列から、containerdがランタイムとして使われていることも確認できます。

ワーカーノードの `ROLES` 列は `<none>` で空のままです。Kubernetesはサーバーノードにだけ `control-plane` のラベルを自動で付け、ワーカーの役割はユーザーに委ねるからです。ラベルを一つ付けておくと出力が読みやすくなり、あとで `nodeSelector` でワークロードをワーカーにだけ配置するときにも使えます。

```bash
kubectl label node k3s-worker1 node-role.kubernetes.io/worker=worker
kubectl get nodes
# NAME          STATUS   ROLES                  AGE   VERSION
# k3s-master    Ready    control-plane,master   20m   v1.31.4+k3s1
# k3s-worker1   Ready    worker                 3m    v1.31.4+k3s1
```

#### ワーカーがjoinできないとき

ワーカーノードが `kubectl get nodes` に現れなかったり `NotReady` のままだったりする場合は、ワーカーで `sudo journalctl -u k3s-agent -f` によってログを見ながら、次を順に確認します。実際に経験した原因は、たいていこの四つのどれかでした。

| 症状 | 原因 | 確認方法 |
|---|---|---|
| `401 Unauthorized` | トークンの不一致 | サーバーの `node-token` を改めてコピーしたか確認 |
| `connection refused` / タイムアウト | サーバーIPや6443番ポートにアクセスできない | ワーカーで `curl -k https://192.168.0.128:6443` |
| joinはできるがPod間の通信が失敗 | Flannel VXLAN(UDP 8472)の遮断 | ファイアウォール(ufwなど)を有効にしているなら下記のポートを許可 |
| ノードが `Ready` と `NotReady` を行き来する | 無線LANの省電力モード | 2.5の設定をワーカーにも適用したか確認 |

Raspberry Pi OSはデフォルトでファイアウォールが無効なのでポートの問題はまれですが、セキュリティのために `ufw` を有効にしているなら、K3sが使うポートを開ける必要があります。

| ポート | プロトコル | 方向 | 用途 |
|---|---|---|---|
| 6443 | TCP | ワーカー → サーバー | Kubernetes API |
| 8472 | UDP | 全ノード間 | Flannel VXLAN |
| 10250 | TCP | 全ノード間 | kubelet (ログ、`exec`、メトリクス) |

ノードをさらに追加したい場合は、同じ手順を繰り返せばよいです。実際このクラスタはその後ワーカーノードが三台まで増え、[TailscaleトンネルでRedisヘルスチェックが失敗する理由]({{< ref "/posts/tailscale-mtu-troubleshooting" >}}) に登場する `k3s-worker2`、`k3s-worker3` がそのノードたちです。

### 3.5 ノートPCからkubectlを使う

毎回サーバーノードにSSHで入って `kubectl` を叩くのは不便です。kubeconfigをノートPCに持ってきて使います。

```bash
# ノートPCで実行
scp marsboy@192.168.0.128:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-home.yaml

# serverのアドレスを127.0.0.1からサーバーノードのIPに変更
sed -i '' 's/127.0.0.1/192.168.0.128/' ~/.kube/k3s-home.yaml

export KUBECONFIG=~/.kube/k3s-home.yaml
kubectl get nodes
```

`k3s.yaml` の `server` フィールドはデフォルトで `https://127.0.0.1:6443` になっているので、必ずサーバーノードのアドレスに書き換える必要があります。このアドレスが先ほど `tls-san` に入れておいた値と一致していないと証明書の検証を通りません。この構成は同じWi-Fiの中でのみ動作します。家の外でも使いたいなら、先に述べたVPNが必要です。

### 3.6 うまくいかないとき: きれいに消す

K3sの利点の一つは、削除が簡単な点です。インストールオプションを変えたいときやクラスタがこじれたときは、インストールスクリプトが一緒に作っておいたアンインストールスクリプトを実行すれば最初の状態に戻ります。

```bash
# サーバーノード
/usr/local/bin/k3s-uninstall.sh

# ワーカーノード
/usr/local/bin/k3s-agent-uninstall.sh
```

最初に構築したときは、このスクリプトを五、六回は実行しました。実験のコストが低いことがホームサーバーの最大の利点なので、ためらわずに消してまた作ってみましょう。

### 3.7 最初のワークロードで検証する

ノードが `Ready` であることは、kubeletがAPIサーバーと通信しているという意味にすぎず、Podが実際に立ち上がって外からアクセスできるかまでを保証するものではありません。クラスタ構築の最終ステップとして、最も単純なワークロードを一つ載せて経路全体を確認します。

```bash
kubectl create deployment hello --image=nginx --replicas=2
kubectl expose deployment hello --type=NodePort --port=80
kubectl get pods -o wide
kubectl get svc hello
```

`-o wide` で見ると、二つのPodがサーバーとワーカーに一つずつ分かれてスケジュールされているのが分かります。先ほど述べたとおり、サーバーノードもワークロードを受け取ります。`nginx` の公式イメージは `arm64` を含むマルチアーキテクチャイメージなので、Raspberry Piでもそのまま実行できます。自分で作ったイメージを載せるときはこの点を必ず確認しなければならず、amd64のノートPCでビルドしたイメージはRaspberry Piでは `exec format error` で落ちます。この問題は、CI/CDパイプラインを扱う次の記事で引き続き取り上げる予定です。

`NodePort` タイプのServiceは、すべてのノードで同じポートを開きます。どのノードのIPにアクセスしてもServiceを経由してPodに到達し、Podのないノードに入ってもFlannelが別のノードへ渡してくれます。

```bash
# ノートPCで実行。ポート番号は kubectl get svc の出力にある30000番台の値
curl -s http://192.168.0.128:31234 | grep title
curl -s http://192.168.0.129:31234 | grep title
# <title>Welcome to nginx!</title>
```

両方のノードから応答が返ってくれば、スケジューリング、イメージのpull、Podネットワーク、Serviceのルーティングがすべて動作しているということです。最後に、内蔵のmetrics-serverでノードにどれだけ余裕が残っているかを確認しておきましょう。Raspberry Pi 4では、K3sのサーバーノードは何も載せていない状態でメモリを600〜700MBほど使います。

```bash
kubectl top nodes
# NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# k3s-master    186m         4%     712Mi           18%
# k3s-worker1   62m          1%     318Mi           8%
```

確認が終わったら片付けます。

```bash
kubectl delete svc,deployment hello
```

---

## おわりに

この記事では、AWS無料枠の終了をきっかけに、Raspberry Piの上へK3sクラスタを構築する過程を扱いました。

1. **ハードウェアとOS** — モデルとストレージの選択、Raspberry Pi OS Liteのインストール、初回起動後のタイムゾーンとスワップの設定
2. **ネットワーク** — 固定IP、ポートフォワーディングとSSHの堅牢化、ProxyJump、無線LANの省電力モード
3. **K3sクラスタ** — containerdベースのサーバー/ワーカーノードのインストール、設定ファイルとバージョン管理、joinのトラブルシューティング、ノートPCからの `kubectl`、最初のワークロードの検証

最初にこの記事を書いたときは「クラスタが立ち上がった」が目標でした。今振り返ると、クラスタを立ち上げるのにかかった時間よりも、そのあとに出会った問題から学んだことのほうが多かったです。クラウドだったら出会うことのなかった問題であり、だからこそマネージドサービスがどんな仕事を代わりにやってくれていたのかを身をもって知ることになりました。費用の面でも初期のRaspberry Piの購入費用以外はほとんどかからないので、インフラを土台から学びたい人にとっては今でも最良の選択肢だと思っています。

クラスタが立ち上がったなら、次はその上に何かを載せる番です。このクラスタはその後、CI/CDとGitOps、VPNを通じたリモートアクセス、サービスメッシュ、モニタリングを一つずつ重ねながら姿を変え続けてきました。デプロイの方式は [GitOpsについて]({{< ref "/posts/about-gitops" >}}) で、外部サービスと接続する際に出会ったネットワークの問題は [TailscaleトンネルでRedisヘルスチェックが失敗する理由]({{< ref "/posts/tailscale-mtu-troubleshooting" >}}) で続きます。

### References

- [Raspberry Pi公式サイト](https://www.raspberrypi.com/software/)
- [Raspberry Pi OS Bookwormのリリースノート — NetworkManagerへの移行](https://www.raspberrypi.com/news/bookworm-the-new-version-of-raspberry-pi-os/)
- [K3s公式ドキュメント — Requirements (Raspberry Pi)](https://docs.k3s.io/installation/requirements#raspberry-pi)
- [K3s公式ドキュメント — Configuration File](https://docs.k3s.io/installation/configuration#configuration-file)
- [K3s公式ドキュメント — Networking Services (ServiceLB, Traefik)](https://docs.k3s.io/networking/networking-services)
- [Kubernetes — Dockershim removal FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)
- [core_kubernetes — Kubernetesのインストール](https://github.com/bjpublic/core_kubernetes/tree/master/chapters/03)
