---
title: "Kubernetesネットワークの本質: CNI、VXLAN、そしてPodはどのように通信するのか"
date: 2026-04-02
draft: false
tags: ["kubernetes", "networking", "cni", "vxlan", "devops"]
translationKey: "kubernetes-cni-networking"
summary: "KubernetesのフラットネットワークモデルからCNIスペック、veth pairとブリッジを使った同一ノード内通信、VXLANカプセル化によるノード間通信まで — Podネットワーキングの内部動作を追う。"
---

[前回の記事]({{< relref "/posts/linux-network-internals" >}})では、OSレベルのネットワーク構造を解説した。ネットワークインターフェース、イーサネットフレーム、ルーティングテーブル、Netfilter/iptablesまで — パケットがカーネル内でどのように動くのかを追いかけた。

今回はその一段上に進む。Kubernetesがこのネットワークスタックの上に**何を積み上げることで**、数百・数千のPodがまるで一つのネットワーク上にあるかのように通信できるようにしているのかを掘り下げる。CNIという契約、veth pairという仮想ケーブル、VXLANというトンネル — 結局これらすべては、OSがすでに提供しているネットワークプリミティブを組み合わせたものだ。

---

## Kubernetesのネットワークモデル

Kubernetesはネットワーク実装を**一切提供しない。** その代わり、三つの根本的な要件だけを宣言している。

1. **すべてのPodはNATなしに他のすべてのPodと通信できなければならない**
2. **すべてのノードはNATなしにすべてのPodと通信できなければならない**
3. **Podが自分自身のIPとして認識するアドレスが、他のPodから見えるIPと同じでなければならない**

一言で言えば、**クラスター全体が一つのフラットなL3ネットワークとして見えなければならない**ということだ。

![Kubernetesフラットネットワークモデル](images/k8s-flat-network-model.svg)

なぜこのモデルを選んだのか。Dockerのデフォルトネットワーキングにその答えがある。Dockerのデフォルトモードでは、コンテナが外部と通信する際にホストIPへSNATされる。受信側から見えるソースIPが実際のコンテナIPではなくホストIPになってしまうのだ。こうなるとロギング、セキュリティポリシー、サービスディスカバリがすべて狂ってしまう。Kubernetesはこの問題を根本から排除するため、「NATなしのフラットネットワーク」を要件とした。

しかし現実の物理ネットワークはフラットではない。ノードが異なるサブネットに存在することもあり、途中にルーターがあり、クラウドVPCはPod IPを知る由もない。**この理想と現実のギャップを埋めるのがCNIプラグインの役割だ。**

---

## CNI（Container Network Interface）— 実装ではなく契約

CNIはCNCFプロジェクトで、コンテナのネットワーク接続を設定・解除するための**インターフェーススペック**だ。重要なのは「CNIはネットワーキングソリューションではなく、契約（contract）である」という点だ。vethを使うか、VXLANを使うか、BGPを使うか — CNIスペックはそこには何も規定しない。それはすべてプラグイン実装の領域だ。

### スペックが定義するもの

**バイナリインターフェース:** CNIプラグインは`/opt/cni/bin/`に配置される実行ファイルだ。コンテナランタイム（containerd、CRI-O）がこのバイナリを直接execする。stdinでJSON設定を受け取り、stdoutで結果を返すシンプルな構造だ。

**オペレーション:** `ADD`（コンテナをネットワークに接続）、`DEL`（切断）、`CHECK`（状態確認）、`VERSION` — この四つだけだ。

### Pod生成時のCNI呼び出しフロー

Podが一つ生成されるとき、ネットワークがどのように準備されるかを追ってみよう。

![CNI呼び出しフロー図](images/cni-call-flow.svg)

```
1. kubelet → CRI経由でcontainerdにPod生成を要求
2. containerd → pauseコンテナを作成してnetwork namespaceを確保
3. containerd → /etc/cni/net.d/ からCNI設定ファイルを読み込む
4. CNIバイナリをexec → ADDを呼び出す
5. CNIプラグイン → veth pairを作成し、IPを割り当て、ルーティングを設定
6. 結果（割り当てられたIP、インターフェース情報）をJSONで返す
7. 実際のアプリケーションコンテナがこのnamespaceに参加
```

ここで2番目のステップの**pauseコンテナ**が重要だ。アプリケーションコンテナが再起動されても、network namespaceはpauseコンテナが保持し続けるため、IPが保持される。

### CNI Chaining

一つのPodに複数のCNIプラグインをチェーンとして接続できる。例えば`calico → bandwidth → portmap`のように、メインプラグインがネットワークを構成し、後続のプラグインがQoSやポートマッピングを追加するという形だ。

---

## 同一ノード内のPod通信

Podが生成されると、カーネルは独立した**network namespace**を作成する。[前回の記事]({{< relref "/posts/linux-network-internals" >}})で扱ったネットワークインターフェース、ルーティングテーブル、iptablesルール — これらすべてがnamespaceごとに独立して存在する。Podは文字通り、自分だけのネットワーク世界を持つ。

隔離されたnamespaceをホストと接続するために**veth pair**を使う。veth pairは仮想イーサネットケーブルだ。一方の端（eth0）はPodのnamespace内に、もう一方の端（vethXXXX）はホストのnamespaceに存在する。一方にパケットを入れると、もう一方から出てくるカーネル内部のパイプだ。

![同一ノード内のPod通信](images/same-node-pod-communication.svg)

Flannelの場合、ホストnamespace側のvethたちは**cni0**というLinuxブリッジに接続される。

**同一ノード内のPod A → Pod B通信経路:**

1. Pod Aでパケットを生成（src: 10.42.0.11, dst: 10.42.0.12）
2. Pod Aのeth0 → vethAを経由してホストnamespaceへ
3. cni0ブリッジがMACアドレステーブルを見てvethBへフォワード
4. vethB → Pod Bのeth0

純粋なL2ブリッジ動作なので**カプセル化のオーバーヘッドはゼロだ。** 前回の記事で扱ったイーサネットフレームのMACベースフォワーディングがそのまま機能している。

---

## 異なるノード間のPod通信 — 核心的な問題

同一ノード内ではブリッジ一つで十分だった。しかし別のノードにあるPodと通信しようとすると話は全く変わってくる。

```
[Node 1: 192.168.1.10]              [Node 2: 192.168.1.20]
  Pod A: 10.42.0.11                   Pod C: 10.42.1.15
  Pod B: 10.42.0.12                   Pod D: 10.42.1.16
```

Pod A（10.42.0.11）がPod C（10.42.1.15）へパケットを送ろうとするとき:

- 10.42.1.15というIPはNode 2の内部でのみ意味を持つアドレスだ
- 物理ネットワークのルーターはPod CIDR（10.42.0.0/16）を全く知らない
- パケットがNode 1を出た瞬間、物理ネットワークはこのパケットをどこへ送ればよいかわからない

![ノード間通信の問題](images/cross-node-problem.svg)

解決方法は大きく二つある:

| 方式 | 核心アイデア | 代表実装 |
|------|-------------|----------|
| **オーバーレイ** | 元のパケットをラップして物理ネットワークが理解できるアドレスで転送 | Flannel VXLAN、Cilium Geneve |
| **アンダーレイ** | 物理ネットワークにPod帯域のルーティングを直接教える | Calico BGP |

---

## VXLAN — L2フレームをUDPでラップするトンネル

### 核心アイデア

VXLAN（Virtual eXtensible LAN）の核心はシンプルだ: **L2イーサネットフレームをUDPパケットの中に入れて、L3ネットワークを通じて転送する。**

[前のシリーズ]({{< relref "/posts/tailscale-mtu-troubleshooting" >}})で扱ったWireGuardの「IPパケットをUDPでラップして送る」カプセル化と同じパターンだが、ラップする対象がIPパケットではなく**イーサネットフレーム全体**である点が異なる。

VXLANの本来の目的は「物理的に離れたネットワークを一つのL2セグメントとして見せること」だ。データセンターでVLANの4,096個のID制限を超えるために作られた技術だが、Kubernetesのオーバーレイネットワークに転用されている。

### カプセル化の構造

![VXLANカプセル化の構造](images/vxlan-encapsulation.svg)

外側から見ると: 外部Ethernetヘッダー（src=Node1 MAC、dst=Node2 MAC）、外部IP（src=192.168.1.10、dst=192.168.1.20）、外部UDP（dst=8472、Linux VXLANデフォルトポート）、VXLANヘッダー（VNI=1）、そして内側に内部Ethernet（Pod A MAC → Pod C MAC）、内部IP（10.42.0.11 → 10.42.1.15）、Payloadの順だ。

物理ネットワークの視点では、このパケットは「Node 1がNode 2に送る普通のUDPパケット」だ。内部にイーサネットフレームがまるごと入っているという事実は、知ることも知る必要もない。

### オーバーヘッドの計算

| 構成要素 | サイズ |
|---|---|
| 外部IPヘッダー | 20 bytes |
| 外部UDPヘッダー | 8 bytes |
| VXLANヘッダー | 8 bytes |
| 内部Ethernetヘッダー | 14 bytes |
| **合計** | **50 bytes** |

MTU 1500の環境でVXLANを使うと、内部パケットは**1450バイト**までしか使えない。FlannelがPodインターフェースのMTUを1450に設定する理由がまさにこの50バイトのオーバーヘッドだ。

WireGuardと比較すると:

| カプセル化方式 | オーバーヘッド | 暗号化 | 有効MTU（1500基準） |
|---|---|---|---|
| VXLAN | 50 bytes | なし | 1450 |
| WireGuard | 60 bytes | ChaCha20-Poly1305 | 1440 |
| VXLAN + WireGuard | 110 bytes | あり | 1390 |

### VXLANヘッダーとVNI

![VXLANヘッダーフォーマット](images/vxlan-header-format.svg)

**VNI**（VXLAN Network Identifier）は24ビットで、約1,677万個の論理ネットワークを作成できる。VLANの12ビット（4,096個）と比べて圧倒的だ。Flannelでは通常VNI=1を使用する。

---

## VTEPとFDB — VXLANのアドレス学習メカニズム

VXLANカプセル化を行うには「このPodのパケットを**どのノードへ**送ればよいか」を知る必要がある。この役割を担うのがVTEPとFDBだ。

### VTEP（VXLAN Tunnel End Point）

Flannel環境で各ノードに作成される`flannel.1`デバイスがVTEPだ。このデバイスがカプセル化とデカプセル化を行う。

### FDB（Forwarding Database）

VTEPは「内部MACアドレスをどの外部IPへマッピングするか」を**FDB**（Forwarding Database）で管理する。

![VTEP/FDBマッピング構造](images/vtep-fdb-mapping.svg)

```bash
# FDBの確認
bridge fdb show dev flannel.1
# aa:bb:cc:dd:ee:ff dst 192.168.1.20 self permanent
# → "このMACアドレスを持つVTEPは192.168.1.20にある"
```

WireGuardのcryptokey routingと概念的に対応している:

| | WireGuard | VXLAN |
|---|---|---|
| マッピング | IP帯域 → public key（ピア） | MAC → VTEP IP |
| 管理主体 | Tailscale coordinationサーバー | Flannel flanneld |

### BUMトラフィック問題とFlannelの解決策

**BUM（Broadcast、Unknown unicast、Multicast）** — 通常のL2ネットワークでは、スイッチが宛先MACを知らない場合にすべてのポートにフラッディングする。VXLANでは「すべてのポート」が「すべてのリモートVTEP」を意味することになり、深刻なスケーラビリティ問題が発生する。

純粋なVXLANスペックはマルチキャストグループでBUMトラフィックを伝播するが、ほとんどのクラウド環境はマルチキャストをサポートしていない。

**Flannelの解決策:** flanneldが**コントロールプレーンでFDBとARPエントリをあらかじめ埋めておく（prepopulate）**。ノードがクラスターにジョインすると、flanneldがすべてのノードのFDBとARPテーブルに情報を直接注入する。

```bash
# Flannelが自動管理するARPエントリ
ip neigh show dev flannel.1
# 10.42.1.0 lladdr aa:bb:cc:dd:ee:ff PERMANENT
# → flanneldがあらかじめ設定したもの。実際のARPブロードキャストは不要
```

これは「**データプレーンの問題をコントロールプレーンに引き上げて解決する**」パターンだ。Tailscaleのcoordinationサーバーがピア情報をあらかじめ配布する構造とまったく同じだ。

---

## Flannel + VXLANノード間通信の全体フロー

これまで学んだすべての概念を一つにまとめよう。Pod A（Node 1、10.42.0.11）からPod C（Node 2、10.42.1.15）へパケットを送る全体の経路だ。

![Flannel VXLAN全体通信フロー](images/flannel-vxlan-full-flow.svg)

### Node 1（送信側）

**ステップ1 — Pod内部のルーティング決定:**
Pod Aのnamespaceのルーティングテーブルはシンプルだ。

```
default via 10.42.0.1 dev eth0
```

10.42.1.15はローカルサブネットにないため、デフォルトルートを経由してeth0（vethのPod側の端）から出ていく。

**ステップ2 — ホストnamespaceに到着、ルーティング決定:**
ホストのルーティングテーブルの重要なエントリ:

```
10.42.0.0/24 dev cni0                      # ローカルPod帯域 → ブリッジ
10.42.1.0/24 via 10.42.1.0 dev flannel.1   # Node 2のPod帯域 → VXLANデバイス
```

宛先10.42.1.15は10.42.1.0/24にマッチ → `flannel.1`デバイスへ転送。

**ステップ3 — VXLANカプセル化:**
`flannel.1`（VTEP）にパケットが入ると、カーネルのVXLANモジュールが:

1. FDBを参照 → 「10.42.1.0/24帯域はNode 2（192.168.1.20）にある」
2. 元のパケットを内部イーサネットフレームでラップ
3. VXLANヘッダー（VNI=1）を追加
4. 外部UDPヘッダー（dst port=8472）を追加
5. 外部IPヘッダー（src=192.168.1.10、dst=192.168.1.20）を追加

**ステップ4 — 物理ネットワークへの送信:**
ホストの実際のNIC（eth0）を通じて送信。物理ネットワークは普通のUDPパケットとして処理する。

### Node 2（受信側）

**ステップ5 — VXLANデカプセル化:**
カーネルがUDPポート8472を見てVXLANモジュールへ転送 → 外部ヘッダーを剥がして内部イーサネットフレームを抽出。

**ステップ6 — ホストルーティング → Podへの転送:**
デカプセル化されたパケット（dst=10.42.1.15）は`10.42.1.0/24 dev cni0`のルートを経由してcni0ブリッジ → Pod Cのvethへ転送される。

**ステップ7 — Pod Cの受信:**
Pod Cから見えるソースIPは10.42.0.11（Pod Aの実際のIP）。NATがないため、Kubernetesのネットワークモデルを満たしている。

---

## オーバーレイ vs アンダーレイ

| | オーバーレイ（Flannel VXLANなど） | アンダーレイ（Calico BGP） |
|---|---|---|
| **物理ネットワーク要件** | なし — どこでも動作 | BGPサポートが必須 |
| **カプセル化オーバーヘッド** | 50 bytes（VXLAN） | なし |
| **適した環境** | クラウド、異種インフラ | オンプレミス、BGP利用可能環境 |
| **パフォーマンス** | カプセル化のCPUコストが発生 | 最大パフォーマンス |
| **デバッグ** | パケットキャプチャ時に二重ヘッダー | 通常のルーティングと同じ |

ほとんどのマネージドKubernetes（EKS、GKE、AKS）や軽量ディストリビューション（k3s）ではオーバーレイがデフォルトだ。物理ネットワークに手を加えなくてよい利便性が、わずかなパフォーマンスオーバーヘッドよりもほとんどの場合で価値があるためだ。

### NICハードウェアオフローディング

VXLANは長年の標準であるため、ほとんどのサーバー級NICがハードウェアオフローディングをサポートしている:

```bash
ethtool -k eth0 | grep vxlan
# tx-udp_tnl-segmentation: on        # カプセル化をNICが処理
# tx-udp_tnl-csum-segmentation: on   # チェックサムもNICが処理
```

カプセル化されたパケットに対してもTSO（TCP Segmentation Offload）とGRO（Generic Receive Offload）が機能するため、実際のCPUオーバーヘッドは理論値よりもはるかに小さい。

---

## CNIプラグイン比較: Calico、Cilium、そしてeBPF

ここまでFlannelを例にCNIの基本動作を見てきた。Flannelはオーバーレイネットワークの構成のみを担当し、NetworkPolicyもBGPもL7処理もない。実際のプロダクションではより多くの機能が必要になるが、そこでCalicoとCiliumが登場する。

### Calico — netfilterの上に積み上げた成熟したアーキテクチャ

CalicoはLinuxカーネルのルーティングスタックとiptablesをそのまま活用する。[前回の記事]({{< relref "/posts/linux-network-internals" >}})で扱ったnetfilterの五つのフックのうち、主に`FORWARD`チェーンにルールを挿入してNetworkPolicyを実装する。

**ノード間通信の三つのモード:**

| モード | カプセル化 | オーバーヘッド | 特徴 |
|------|--------|---------|------|
| BGP | なし | 0 bytes | 物理ネットワークにPodルートを直接アドバタイズ |
| VXLAN | L2 over UDP | 50 bytes | クラウド環境で使用 |
| IPIP | IP-in-IP | 20 bytes | VXLANより軽量だが互換性の問題が起こる可能性あり |

各ノードで**Felix**（DaemonSet）がiptablesルールとルートを管理し、**BIRD**がBGPデーモンの役割を担う。Kubernetes標準のNetworkPolicyを完全サポートしながら、`GlobalNetworkPolicy`のような独自CRDによる拡張も可能だ。

### Cilium — eBPFでnetfilterをバイパスする

Ciliumの核心アイデアは「**netfilterをバイパスしよう**」だ。

eBPF（extended Berkeley Packet Filter）は、カーネルのソースを変更せずにカーネル内部でサンドボックス化されたプログラムを実行できるようにする技術だ。CiliumはこのeBPFプログラムを**TC（Traffic Control）フック**と**XDP（eXpress Data Path）フック**に直接アタッチする。これらのフックはnetfilterよりも**はるかに前段**で動作する。

```
従来の経路（iptables）:
  NIC → netfilter PREROUTING → routing → netfilter FORWARD → NIC

Ciliumの経路（eBPF）:
  NIC → XDP/TC eBPFプログラム → 直接redirect → 対象Podのveth
```

iptablesが何千ものルールを**線形探索**（O(n)）するのに対し、Ciliumは**eBPFマップ**（ハッシュテーブル）を使って**O(1)ルックアップ**でポリシーを評価する。Serviceが1,000個あるとき、iptablesは最悪の場合1,000回の比較が必要だが、Ciliumはハッシュ一回で済む。

![iptables経路 vs eBPF経路の比較](images/iptables-vs-ebpf-path.svg)

**Ciliumが追加で提供するもの:**

- **kube-proxyの完全代替**: Service VIP → バックエンドPodのマッピングをeBPFマップに保存し、TCフックでDNATを実行
- **Identityベースのセキュリティ**: IPではなくラベルベースの数値identityでポリシーを適用。Pod IPが変わっても、ラベルが同じであればポリシーを維持
- **Hubble**: eBPFで収集したネットワークフローをL7（HTTP、gRPC、Kafka、DNS）まで観測。専用サイドカーなしにカーネルレベルで観測可能

### 構造的比較

| 観点 | Calico（iptables） | Cilium（eBPF） |
|---|---|---|
| **パケット処理位置** | netfilterフック | TC/XDP eBPFフック |
| **ポリシールックアップ** | O(n)線形探索 | O(1)ハッシュルックアップ |
| **ポリシー更新** | チェーン全体の書き直し | マップエントリのアトミック更新 |
| **kube-proxy** | 別途運用 | 完全代替 |
| **セキュリティモデル** | IPベース | Identity（ラベル）ベース |
| **L7処理** | なし | Envoy内蔵 + Hubble |
| **カーネル要件** | 特別な要求なし | 4.19+（推奨5.10+） |

Cilium一つでFlannel（オーバーレイ）+ Calico（ポリシー）+ kube-proxy（サービスロードバランシング）を代替できる。ただし、カーネルバージョン要件があり、eBPFベースのためデバッグツール（`bpftool`、`cilium monitor`）が異なる点は運用時に考慮が必要だ。

---

## 付録: 環境確認コマンド

```bash
# CNIバイナリの確認
ls /opt/cni/bin/

# CNI設定の確認
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist

# どのCNIが動いているか
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium"

# VXLANデバイスの確認
ip -d link show flannel.1

# FDBの確認
bridge fdb show dev flannel.1

# ARPエントリの確認
ip neigh show dev flannel.1

# VXLANオフロードの確認
ethtool -k eth0 | grep vxlan

# PodインターフェースのMTU確認（1450ならVXLANの50バイトオーバーヘッドが反映されている）
kubectl exec <pod> -- ip link show eth0

# k3sの起動オプション確認
cat /etc/systemd/system/k3s.service
```
