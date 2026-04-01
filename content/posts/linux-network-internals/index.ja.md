---
title: "OSネットワーク内部構造：インターフェースからNetfilterまで"
date: 2026-03-31
draft: false
tags: ["networking", "linux", "kernel", "iptables", "devops"]
translationKey: "linux-network-internals"
summary: "ネットワークインターフェース、イーサネットフレーム、ルーティングテーブル、Netfilter/iptablesまで — OSレベルでパケットが実際にどのように動くのかを、ifconfig、ip route、iptablesコマンドとともに解説します。"
---

開発をしていると、ネットワークはいつも「なんとなく動くもの」のように感じられます。`curl`を打てばレスポンスが返ってきますし、Docker Composeでコンテナを立ち上げれば互いに通信できます。しかし、VPNトンネルで大きなパケットだけドロップされたり、コンテナネットワークが丸ごと通信不能になったりする瞬間が訪れると、「パケットが実際にどのように動いているのか」を知らなければ原因を突き止めることはできません。

この記事では、OSレベルでパケットが実際に通る経路をたどります。`ifconfig`の出力を読む方法から始めて、イーサネットフレーム、ルーティングテーブル、Netfilter/iptablesまで — ネットワークトラブルシューティングの基礎となる内容を一つの記事にまとめます。

## ネットワークインターフェース — カーネルが作った出入口

ネットワークインターフェースは、**オペレーティングシステムのカーネルがネットワークと通信するために作った抽象化された出入口**です。

複数のアプリケーションが同時にモニターを占有できないためカーネルを通じて画面を使用するように、ネットワークも同様です。プログラムがWi-Fiチップに直接命令を出したり、NICメモリにアクセスしたりすることは不可能で、必ずカーネルを経由する必要があります。インターフェースはカーネルが提供するこの出入口であり、プログラムはソケットを通じてここにデータを渡すだけです。

`ifconfig`コマンドで現在のシステムのネットワークインターフェースを確認できます。

```
➜  ~ ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
        options=1203<RXCSUM,TXCSUM,TXSTATUS,SW_TIMESTAMP>
        inet 127.0.0.1 netmask 0xff000000
        inet6 ::1 prefixlen 128
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
        nd6 options=201<PERFORMNUD,DAD>

# ... gif0, stf0, anpi0-3, en1-6 など非アクティブなインターフェースは省略 ...

en0: flags=88e3<UP,BROADCAST,SMART,RUNNING,NOARP,SIMPLEX,MULTICAST> mtu 1500
        options=6460<TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
        ether aa:bb:cc:dd:ee:01
        inet6 fe80::xxxx:xxxx:xxxx:xxxx%en0 prefixlen 64 secured scopeid 0xe
        inet 192.168.0.10 netmask 0xffffff00 broadcast 192.168.0.255
        nd6 options=201<PERFORMNUD,DAD>
        media: autoselect
        status: active

bridge0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        options=63<RXCSUM,TXCSUM,TSO4,TSO6>
        ether aa:bb:cc:dd:ee:02
        member: en1 flags=3<LEARNING,DISCOVER>
        member: en2 flags=3<LEARNING,DISCOVER>
        member: en3 flags=3<LEARNING,DISCOVER>
        nd6 options=201<PERFORMNUD,DAD>
        media: <unknown type>
        status: inactive

awdl0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        options=6460<TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
        ether aa:bb:cc:dd:ee:03
        inet6 fe80::xxxx:xxxx:xxxx:xxxx%awdl0 prefixlen 64 scopeid 0x10
        nd6 options=201<PERFORMNUD,DAD>
        media: autoselect
        status: active
```

出力はかなり長いですが、主要なインターフェースだけを抽出すると構造が見えてきます。

- **`lo0`** — ループバック。外部に出ない自分自身との通信（`127.0.0.1`）用です。MTUが16384と大きいのは、物理的な転送がないため制約が緩いからです。
- **`en0`** — Wi-Fi。`status: active`の唯一の物理インターフェースで、実際のインターネットトラフィックがこの出入口を通過します。`ether`フィールドにMACアドレスが、`inet`に割り当てられたIPが表示されます。
- **`bridge0`** — Thunderbolt Bridge。`en1`、`en2`、`en3`をメンバーとして束ねるL2仮想スイッチです。
- **`awdl0`** — Apple Wireless Direct Link。AirDropなどAppleデバイス間のP2P通信に使用されます。

残りの`anpi*`、`en1-6`、`gif0`、`stf0`などは、現在非アクティブ（`inactive`）状態のインターフェースです。Thunderboltポート、トンネルインターフェースなど、macOSがあらかじめ作成しているものです。

ここで注目すべき点は、インターフェースごとに`ether`（MACアドレス）、`inet`（IP）、`mtu`、`flags`といった情報がレイヤーごとに並んでいることです。これらの各フィールドが何を意味するのかを理解するには、まずイーサネットフレームの構造を知る必要があります。

### 重要な原則：インターフェースの背後に必ずしも物理ハードウェアがあるとは限らない

![ネットワークインターフェースの抽象化構造](images/network-interface-abstraction.svg)

| インターフェース | 出入口の先にあるもの | 説明 |
|---|---|---|
| `en0` | Wi-Fiチップ（ハードウェア） | パケットが電波に変換されて物理的に送信される |
| `lo0` | カーネルメモリ | パケットが外部に出ず、カーネル内で即座に折り返す |
| `utun` / `wg0` | ユーザースペースプロセス | VPNアプリ（Tailscaleなど）がパケットを受け取り、暗号化して再度物理インターフェースへ送信 |
| `bridge0` | 複数のインターフェースを束ねる仮想スイッチ | L2レベルでフレームをフォワーディング |
| `veth` | コンテナ/Podのネットワークネームスペース | 一方から入れたパケットが反対側から出てくるパイプ |

カーネルの観点では、これらすべてのインターフェースが同一の抽象化として扱われます。「パケットを入れればどこかへ送られる」というインターフェースの契約（contract）は、物理でも仮想でも同じです。Dockerの`veth`でも、Tailscaleの`utun`でも、カーネルは同じ方法でパケットを転送します。

---

## イーサネット — 同じネットワーク内での通信ルール

### イーサネットとは

イーサネットは、**同じネットワーク内でデバイス同士がデータをやり取りするルール**です。OSIモデルにおいてL1（物理層）とL2（データリンク層）を定義するIEEE 802.3規格です。

名前の由来は、19世紀の物理学で光が伝わる媒質と信じられていた「エーテル（Ether）」で、ネットワークという見えない媒質を通じてデータが伝達されるという比喩です。

### イーサネットフレームの構造

イーサネットの基本的な転送単位は**フレーム**（Frame）です。

![イーサネットフレームの構造](images/ethernet-frame-structure.svg)

各フィールドの役割：

- **Preamble（8バイト）**：受信側NICのクロック同期のためのパターン（`10101010...`）。最後の1バイト（SFD）は`10101011`で「ここからフレームが始まる」という信号です。
- **Dst/Src MAC（各6バイト）**：宛先と送信元のMACアドレス。`ff:ff:ff:ff:ff:ff`ならブロードキャストです。
- **Type（2バイト）**：ペイロードに含まれる上位プロトコルの識別子。`0x0800` = IPv4、`0x86DD` = IPv6、`0x0806` = ARP。
- **Payload（46～1500バイト）**：上位レイヤーのデータ（IPパケットなど）。最大1500バイトが標準MTUです。
- **FCS（4バイト）**：Frame Check Sequence。CRC-32アルゴリズムによるフレーム完全性検証です。

L2レイヤーで使用するデータ単位はフレームであり、L3のIPレベルではパケットと呼びます。したがって、フレームはIPパケットをペイロードとして、ヘッダーを付けて梱包し送信準備をします。

当然ながら「どのIPに送るか」という内容を含んでいるIPパケットは、フレームのペイロードの中にあるため、フレームのヘッダーではMACアドレスを通じて「どのMACアドレスに送るか」という内容を定義することになります。

### MACアドレス

MAC（Media Access Control）アドレスは48ビット（6バイト）のハードウェア識別子です。先頭3バイトはメーカー（OUI）、後ろ3バイトは固有番号です。

```
96:60:2d:e2:2f:03
├──────┤├──────┤
  OUI    固有番号
(メーカー)
```

MACアドレスは同一ネットワークセグメント（L2）内でのみ意味を持ちます。**ルーターを越えると、MACは次のホップのものに置き換えられます。**

### Wi-Fiとイーサネットの関係

Wi-Fi（802.11）は無線イーサネットです。同一のフレーム構造を共有し、MACアドレスベースで通信します。macOSでWi-Fiインターフェースが`en0`（Ethernetの略）である理由がこれです。

---

## ifconfigの出力を読む方法

### 基本構造

```
インターフェース名: flags=値<フラグ一覧> mtu 値
    options=値<オプション一覧>
    ether MACアドレス          ← L2 (Data Link)
    inet IPv4アドレス          ← L3 (Network)
    inet6 IPv6アドレス         ← L3 (Network)
    media: ...             ← L1 (Physical)
    status: active/inactive
```

この構造自体がOSIレイヤーの順序 — Physical → Data Link → Network の順に情報が並んでいます。

### 主要なflagsの解釈

| フラグ | 意味 |
|---|---|
| `UP` | カーネルによりインターフェースが有効化されている |
| `RUNNING` | L1レベルでリンクが生きている。`UP`なのに`RUNNING`がなければ「ドライバーは起動したがケーブルが抜けている」状態 |
| `BROADCAST` | ブロードキャスト可能（イーサネット系） |
| `LOOPBACK` | `lo0`専用。自分自身へのトラフィック専用 |
| `POINTOPOINT` | トンネルインターフェース（`utun`）。両端点のみを接続する1:1リンク |
| `PROMISC` | プロミスキャスモード。自分のMACでないパケットもすべて受信。bridgeメンバーで有効化 |

実際に`ifconfig`コマンドで確認すると、以下のようにさまざまなフラグがあることがわかります。

```
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
```

フラグの内容に応じて、上記のような情報を確認できます。

### NICオフロード機能（options）

| オプション | 意味 |
|---|---|
| `RXCSUM` / `TXCSUM` | 受信/送信チェックサムをNICが計算（CPU負荷軽減） |
| `TSO4` / `TSO6` | TCP Segmentation Offload。カーネルが大きなセグメントをNICに渡すと、NICがMTUサイズに分割 |
| `CHANNEL_IO` | Apple独自の高性能I/Oチャネルベースのパケット処理パス |

`TSO`が有効な場合、tcpdumpでMTU（1500）より大きなパケットが見えるのは正常です。NICが分割する前の大きなセグメントをキャプチャしているためです。

---

## macOSとLinuxのインターフェース命名規則

`ifconfig`コマンドを確認するとさまざまなインターフェースがありますが、それぞれが何を意味するのかを見ると次のとおりです。macOSとLinux系ではインターフェースを指すキーワードが若干異なります。

### macOS（BSD系）

macOSのインターフェース名は**ドライバー名 + インスタンス番号**のパターンです。

| プレフィックス | 意味 | 例 |
|---|---|---|
| `en` | Ethernet（有線 + Wi-Fi含む） | `en0` = Wi-Fi、`en1-3` = Thunderbolt |
| `lo` | Loopback | `lo0` = 127.0.0.1 |
| `utun` | User-space Tunnel | `utun1` = Tailscale（WireGuard） |
| `bridge` | L2 Bridge | `bridge0` = Thunderbolt Bridge |
| `awdl` | Apple Wireless Direct Link | `awdl0` = AirDrop |

Apple Silicon Macでは`en0`が常にWi-Fiです。Intel Macでは有線が`en0`、Wi-Fiが`en1`でしたが、逆転しました。

また、en1からen3まではThunderboltに属します。表の中ほどにbridge0がThunderbolt Bridgeと書かれていますが、文字通りThunderboltインターフェースのブリッジ役割を果たしており、詳細は後述します。

### Linux（systemd）

Linuxは物理的な位置ベースの名前を使用します：`enp3s0`（PCIバス3、スロット0）、`ens1`（ホットプラグスロット1）。ハードウェアを変更しても名前が変わらないという利点があります。

| 項目 | macOS | Linux |
|---|---|---|
| 命名規則 | 検出順序ベース（`en0`、`en1`） | 物理位置ベース（`enp3s0`） |
| ネットワークスタック | XNU（BSD系 `ifnet`） | Linux固有（`net_device`） |
| ファイアウォール | pf（Packet Filter） | netfilter（iptables/nftables） |

---

## Bridgeインターフェース

Bridgeは、複数の物理（または仮想）インターフェースを一つのL2ブロードキャストドメインにまとめる仮想スイッチです。実際のスイッチ機器がなくても、カーネルがソフトウェアで同じ役割を果たします。

![macOS bridge0 — Thunderbolt Bridge](images/bridge-macos.svg)

### macOS bridge0

先ほどの`ifconfig`出力で、`bridge0`が`en1`、`en2`、`en3`をメンバーとして持っていることを確認しました。この3つのインターフェースはそれぞれMacのThunderboltポートに対応します。`bridge0`はこれらを一つのL2セグメントにまとめる仮想スイッチの役割を果たします。

例えば、Mac A（自分のコンピュータ）のThunderboltポートにMac BとMac Cがそれぞれケーブルで接続されているとしましょう。`bridge0`がなければ、`en1`と`en2`は完全に別のネットワークセグメントです。Mac BがMac Cにパケットを送るにはIPレベルのルーティング（L3）が必要です。

しかし`bridge0`が`en1`と`en2`を束ねているため、Mac B → Mac Cのトラフィックはブリッジ内部でMACアドレスベースで直接フォワーディングされます。まるで3台のMacが同じイーサネットスイッチに接続されているのと同じ効果です。これがL2ブリッジングの本質です — IPを経由せずフレームレベルで直接転送します。

### KubernetesにおけるBridge

この概念はKubernetesのネットワーキングでも同様に登場します。FlannelなどのCNIプラグインは、各ワーカーノードに`cni0`というブリッジを作成し、同じノードにあるPodのvethインターフェースをこのブリッジに接続します。

結果として、同じノードのPodは`cni0`ブリッジを通じてL2レベルで直接通信します。macOSで`bridge0`がThunderboltで接続されたMac同士を一つのネットワークにするように、`cni0`は同じノードのPod同士を一つのネットワークにするのです。規模と用途が異なるだけで、原理は同じです。

---

## カーネルネットワークスタック

### なぜネットワークがカーネルにあるのか

カーネルはハードウェアとプログラムの間の唯一の仲介者です。これはモニター、キーボード、マウスと同じ原則です。

1. **NICはハードウェア** → カーネルのみが制御可能
2. **複数のアプリが同時にネットワークを使用** → 誰かが仲裁する必要がある → カーネルの役割
3. **ルーティング、ファイアウォールなどの共通ポリシー** → すべてのアプリに一貫して適用 → カーネルが一か所で処理

### Linuxカーネルネットワークスタックの構造

ネットワークを使用するにはカーネルを制御する必要があると述べましたが、ソケットを通じてカーネルレイヤーにアクセスし、OSI 7レイヤーに対応するようにさまざまなレイヤーを通過します。図で表すと以下のとおりです。

![Linuxカーネルネットワークスタック](images/kernel-network-stack.svg)

各層の役割：

- **Socket layer**：アプリがカーネルと対話する唯一の窓口。`socket()`システムコールでソケットfdを生成し、`send`/`recv`でデータを交換します。
- **Transport layer**：TCPならシーケンス番号、フロー制御、再送、輻輳制御。UDPならほぼパススルーです。
- **IP layer + Routing + Netfilter**：ルーティングテーブルを参照して出口インターフェースを決定し、Netfilterフックに登録されたルールを実行します。
- **Network device subsystem**：インターフェース一覧、MTU、状態（UP/DOWN）、キューイング規則（tc/qdisc）などを管理します。
- **Device drivers**：ハードウェア制御（NICドライバー）、ユーザースペース接続（TUNドライバー）、コンテナ接続（vethドライバー）。

アプリケーションがネットワークへデータを送信する際、データは一塊りで出るわけではありません。レイヤーを下るほどIPヘッダー（20B）、TCPヘッダー（20B）などが付加されるため、L2で定められた最大転送単位であるMTU（通常1500B）を超えないようにデータを事前に分割する必要があります。

このときTCPが使用する単位が**MSS（Maximum Segment Size）**です。MSS = MTU - IPヘッダー - TCPヘッダーなので、標準環境では1500 - 20 - 20 = 1460バイトとなります。アプリケーションがソケットにどれだけ大きなデータを書き込んでも、TCPがMSSサイズに切り分けて下位に送ります。

### macOS vs Linux

この記事で扱うネットワークスタック構造はLinuxカーネル基準ですが、macOSも大枠では同じ役割を持つコンポーネントを備えています。名前と実装は異なりますが、レイヤー構造自体は同じと考えて問題ありません。

```
macOS XNU カーネル                   Linux カーネル
├─ ifnet (BSD Network Stack)      ├─ net_device (Linuxネットワークスタック)
├─ pf (Packet Filter)             ├─ netfilter (+ iptables/nftables)
├─ utun (NetworkExtension)        ├─ tun/tap, wireguard モジュール
└─ bridge (ifnet bridging)        └─ bridge, veth, vxlan モジュール
```

---

## ルーティングテーブル — 「このパケットをどこに送るか？」

先ほどカーネルネットワークスタックの構造を見ました。アプリケーションがソケットにデータを書き込むと、TCP/UDP（L4）を経てIPレイヤー（L3）に到達します。まさにこのIPレイヤーでカーネルが最初に行うのがルーティングテーブルの参照です — 「このパケットをどのインターフェースで、どこに向けて送り出すか？」を決定するのです。

ルーティングテーブルはカーネルが管理する経路情報データベースです。宛先IPアドレスを見て最も具体的に一致する経路を選択しますが、このアルゴリズムを**Longest Prefix Match（LPM）**と呼びます。

### ルーティングテーブルの確認

オペレーティングシステムによって、ルーティングテーブルを確認する方法は以下のとおりです。

**Linux：**

```bash
ip route show
```

```
default via 192.168.1.1 dev eth0          # デフォルトゲートウェイ
192.168.1.0/24 dev eth0 scope link        # ローカルサブネットはeth0で直接配送
10.0.0.0/8 via 10.1.1.1 dev wg0          # 10.x帯域はWireGuardトンネルへ
```

**macOS：**

```bash
netstat -rn                    # ルーティングテーブル全体の表示
route -n get default           # デフォルトゲートウェイの詳細情報
route -n get 10.0.5.3          # 特定IPへの経路を照会
```

### 主要な要素

上記のコマンドを解釈するために知っておくべき要素は以下のとおりです。

| 要素 | 説明 |
|---|---|
| Destination | 宛先ネットワーク（CIDR表記） |
| Gateway (via) | 次のホップアドレス（直接接続の場合はなし） |
| Device (dev / Netif) | 出口ネットワークインターフェース |
| Metric | 同じ宛先に対して複数の経路がある場合の優先順位 |
| Scope | link（ローカルサブネット）、global（リモート）など |

### macOSルーティングテーブルの解釈

macOS環境でnetstatを使って確認すると、Internetに応じてさまざまな情報が以下のように表示されます。

```
Destination        Gateway            Flags               Netif Expire
default            192.168.20.1       UGScg                 en0
127                127.0.0.1          UCS                   lo0
192.168.20/23      link#14            UCS                   en0      !
192.168.20.1       ac:71:2e:f:ad:88   UHLWIir               en0   1198
224.0.0/4          link#14            UmCS                  en0      !
```

各経路の意味：

- **`default → 192.168.20.1`**：一致する経路がなければ必ずゲートウェイ（ルーター）に送ります。インターネットへ出るすべてのトラフィックです。
- **`127.0.0.0/8`**：localhostトラフィック。ネットワーク外に出ず`lo0`で自分自身に返ってきます。
- **`192.168.20/23`**：ローカルサブネット。ゲートウェイなしで`en0`から直接通信します。
- **`192.168.20.1 → MACアドレス`**：ARPキャッシュ連動ホスト経路。Expireの数値はARPキャッシュ期限切れまでの残り秒数です。
- **`224.0.0.0/4`**：マルチキャスト帯域。mDNS（Bonjour — AirDrop、AirPlay）、SSDP（UPnP）などです。

### Flagsの意味

| Flag | 意味 |
|---|---|
| `U` | Up（経路アクティブ） |
| `G` | Gateway経由（直接接続ではない） |
| `H` | Host経路（/32、特定のホスト1台） |
| `S` | Static（手動またはシステム設定） |
| `L` | Link-layerアドレスあり（MAC確認済み） |
| `W` | Was cloned（C経路から複製された） |
| `m` | Multicast |

### パケットフローの例

- **ローカルサブネット通信**（`192.168.20.7`へping）：サブネットマッチ → ゲートウェイなしで`en0`から直接配送 → ARPキャッシュでMAC確認 → イーサネットフレーム送信
- **インターネット通信**（`8.8.8.8`へping）：サブネットマッチなし → default経路マッチ → ゲートウェイ`192.168.20.1`へ転送 → ルーターがインターネットへルーティング

---

## Netfilterとiptables — 「このパケットをどう処理するか？」

ルーティングテーブルが「どこに送るか？」を決定するなら、Netfilterは「送ってよいか？変更するか？」を決定します。

| 区分 | ルーティングテーブル | iptables / Netfilter |
|---|---|---|
| 核心的な問い | 「どこに送るか？」 | 「送ってよいか？変更するか？」 |
| 判断基準 | 宛先IP | 送信元/宛先IP、ポート、プロトコル、状態など |
| 動作 | 経路選択 | 許可/遮断/アドレス変換/パケット変更 |

両者は独立したシステムですが密接に連動します。Netfilterがパケットを改変するとルーティング判断が変わり、ルーティング判断の結果に応じて通過するNetfilterフックが変わります。

### iptablesの役割

iptablesはカーネルのNetfilterに「このようなパケットが来たらこう処理せよ」と**ルールを登録するユーザースペースツール（userspace tool）**です。実際のパケット処理はカーネル内のNetfilterが行います。ルールが一度登録されれば、iptablesプロセスが終了してもルールはカーネルに残り続けます。

```
iptables (ユーザースペース CLI)
    │
    │ netlinkソケットでカーネルにルールを伝達
    ▼
netfilter (カーネルフレームワーク)
    ├─ PREROUTING hook: 登録されたルールをチェック
    ├─ INPUT hook: 登録されたルールをチェック
    ├─ FORWARD hook: 登録されたルールをチェック
    ├─ OUTPUT hook: 登録されたルールをチェック
    └─ POSTROUTING hook: 登録されたルールをチェック
```

---

## Netfilterの5つのHook — パケットフローの検問所

### フックの概念

フック（Hook）とは「ある処理フローの特定の地点に割り込むことができるエントリーポイント」です。Netfilterはカーネルネットワークスタックのパケット処理フローの途中に、**「ここで少し待って、登録されたルールがあれば実行する」**というチェックポイントを5つ設けています。

### 全体のパケットフロー

カーネルの視点でパケットが到着すると、必ず一つの問いを立てます：**「このパケットの宛先は自分か、そうでないか？」** この問いの答えに応じて経路が分岐し、5つのフックはこの経路上の異なる地点に配置されています。

![Netfilterの5つのHookとパケットフロー](images/netfilter-hooks.svg)

### 各フックの役割

**① PREROUTING — パケットが到着した直後、ルーティング判断の前**

パケットがNICを通じてカーネルに入った直後、最初に通過するフックです。ここで宛先アドレスを変更すると（DNAT）、後続のルーティング判断結果が変わります。

```bash
# 外部8080 → 内部192.168.1.10:80へDNAT
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 192.168.1.10:80
```

**② ルーティング判断（Routing Decision）**

Netfilterフックではなくカーネル IPスタックの動作ですが、フローの理解に不可欠です。宛先IPを見てルーティングテーブルを参照し、INPUT経路とFORWARD経路に分けます。

**③ INPUT — このホスト宛てのパケットがローカルプロセスに渡される直前**

サーバーファイアウォールの要です。このホストが最終宛先であるパケットにのみ適用されます。

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT     # SSH許可
iptables -A INPUT -p tcp --dport 80 -j ACCEPT     # HTTP許可
iptables -A INPUT -j DROP                          # 残りは遮断
```

INPUTを通過した後も、カーネルのソケット層で「このポートをlistenしているプロセスがあるか？」を確認します。この2つの段階は独立しています — iptablesで80番をACCEPTしても、nginxが起動していなければRSTが発生します。

**④ FORWARD — このホストを経由するパケット**

このホストがルーター役割を果たす場合にのみ意味があります。Dockerコンテナネットワーキング、VPNサーバー、Linuxルーターが代表的なケースです。LinuxではデフォルトでIPフォワーディングが無効になっているため、`sysctl net.ipv4.ip_forward=1`で有効化する必要があります。

**⑤ OUTPUT — このホストで生成されたパケットが出る直前**

ローカルプロセスが作成したパケットがカーネルネットワークスタックに降りてきた直後に通過するフックです。

```bash
# 外部SMTP（25番ポート）送信を遮断
iptables -A OUTPUT -p tcp --dport 25 -j DROP
```

**⑥ POSTROUTING — パケットが最終的にNICから出る直前**

外部に出るすべてのパケットが最後に通過するフックです。SNAT/MASQUERADEがここで行われます。

```bash
# 内部ネットワークの送信元をグローバルIPに変換
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

### 各フックがその位置にある理由

| フック | 核心的な理由 |
|---|---|
| PREROUTING | ルーティング判断に影響を与える必要があるから（DNAT） |
| INPUT | プロセスに到達する前に遮断する必要があるから |
| FORWARD | 他のパケットを無条件に通過させてはならないから |
| OUTPUT | 出てはいけないトラフィックを捕捉する必要があるから |
| POSTROUTING | すべての判断が終わった後にアドレスを変換する必要があるから |

このように、Netfilterで押さえておくべき基本的なフックは5つで構成されています。このうちFORWARDを除けば残りはペアを成しているため、図を描いてみると容易に理解できます。

---

## iptablesの構造：テーブル、チェーン、ルール

iptablesは**テーブル（Table） > チェーン（Chain） > ルール（Rule）**の3段階層構造を持ちます。

### テーブル（Table）

一つのフックで性質の異なる複数の処理を分離するためにテーブルが存在します。

![iptablesのテーブル](images/iptables-tables.svg)

**filter** — 最も基本的で最もよく使われるテーブルです。パケットを許可または遮断します。

```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # SSH許可
iptables -A INPUT -j DROP                         # 残りは遮断
```

**nat** — 送信元または宛先のIP/ポートを変換します。ルーターNAT、ポートフォワーディング、Dockerコンテナネットワーキングなどがここで行われます。natテーブルは**接続の最初のパケットにのみ適用**され、以降は**conntrack**（接続追跡）が自動的に同じ変換を適用します。

```bash
# DNAT（宛先変換） — PREROUTING
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 192.168.1.10:80

# MASQUERADE（送信元変換） — POSTROUTING
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

**mangle** — パケットのIPヘッダーフィールドを直接変更します。TTL変更、TOS変更、MARK（カーネル内部タグ）設定などです。

**raw** — 特定のパケットをconntrack追跡対象から除外します。大規模トラフィックサーバーでのconntrackテーブルオーバーフロー防止用です。他のすべてのテーブルより最初に実行されます。

### チェーン（Chain）

チェーンは「特定のフックで、特定のテーブルのルールを順番に格納したリスト」です。一つのフックで複数のテーブルのチェーンが**固定された順序（raw → mangle → nat → filter）**で実行されます。どのテーブルがどのフック（チェーン）で動作するかは、以下のマッピング表のとおりです。

![テーブル-チェーンマッピング](images/iptables-chain-mapping.svg)

ユーザー定義チェーンも作成できます。ルールが多くなった際の整理用で、プログラミングの関数呼び出しに似ています。

```bash
# ユーザー定義チェーンの作成と使用
iptables -N WEB_TRAFFIC
iptables -A WEB_TRAFFIC -p tcp --dport 80 -j ACCEPT
iptables -A WEB_TRAFFIC -p tcp --dport 443 -j ACCEPT
iptables -A WEB_TRAFFIC -j DROP

# INPUTからこのチェーンに分岐
iptables -A INPUT -p tcp -j WEB_TRAFFIC
```

### ルール（Rule）

ルールは**マッチ条件（Match）**と**ターゲット（Target）**で構成されます。チェーン内のルールは上から下へ順番に評価され、終了ターゲットにマッチすると即座に決定されます。

![ルール構造の例](images/iptables-rule-structure.svg)

**終了ターゲット（Terminating）** — 次のルールに進まない：

| ターゲット | 動作 |
|---|---|
| `ACCEPT` | パケット通過 |
| `DROP` | 無応答で破棄（相手はタイムアウトまで待つ） |
| `REJECT` | 拒否 + ICMPエラー応答を送信 |
| `DNAT` | 宛先アドレス変換 |
| `SNAT` / `MASQUERADE` | 送信元アドレス変換 |

**非終了ターゲット（Non-terminating）** — 処理後も次のルールの評価を続行：

| ターゲット | 動作 |
|---|---|
| `LOG` | カーネルログに記録して次のルールへ続行 |
| `MARK` | 内部マークを設定して次のルールへ続行 |

```bash
# LOG（非終了）の後にDROP（終了） — ログ記録後に遮断
iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "SSH attempt: "
iptables -A INPUT -p tcp --dport 22 -s !10.0.0.0/8 -j DROP
```

どのルールにもマッチしない場合、チェーンの**デフォルトポリシー（Policy）**が適用されます：

```bash
iptables -P INPUT DROP    # INPUTチェーンのデフォルトポリシーをDROPに
```

---

## パケットの実際の旅路 — curl一行の重み

`curl https://api.example.com`を実行すると、一つのHTTPリクエストが以下の階層を通過します。

![curl実行時のパケットの旅路](images/packet-journey-curl.svg)

1. **L7 Application**：curlがHTTPリクエスト文字列を生成します。`socket()`システムコールでカーネルにソケットfdを要求します。
2. **L6-5 TLS/Session**：TLSハンドシェイクが行われます。HTTPデータがAES-GCMで暗号化されます。
3. **L4 Transport（TCP）**：TCPヘッダー付与 — 送信元ポート（エフェメラル）、宛先ポート（443）、シーケンス番号、チェックサム。
4. **L3 Network（IP）**：IPヘッダー付与 + ルーティングテーブル参照 → 出口インターフェース決定。
5. **L3 Netfilter Hooks**：OUTPUT → POSTROUTINGの順にiptablesルールをチェック。
6. **L2 Data Link**：ARPキャッシュでゲートウェイMAC参照 → イーサネットヘッダー + FCS付与。
7. **L1 Physical**：NICがフレームを電気/Wi-Fi信号に変換して送信。

L1で電気信号になったパケットはルーター（ゲートウェイ）に転送されます。ルーターはNATを実行してプライベートIPをグローバルIPに変換し、ISPネットワークに送り出します。その後、ISP間のBGPルーティングに従いIX（Internet Exchange）を経由して宛先サーバーのあるISPに到達し、最終的にサーバーのNICに到着します。

レスポンスはこの全経路を逆順にたどります。サーバーでL7まで上がってから再びL1に下り、同じ物理経路を経て自分のNICに到着し、カーネルがL1 → L7の順にヘッダーを剥がしていき、最終的にcurlがHTTPレスポンスを出力します。

---

## 実践的なサーバーファイアウォール設定パターン

ほとんどのサーバーはルーターではなく最終宛先であるため、INPUTチェーンがサーバーファイアウォールの要です。

```bash
# デフォルトポリシー：すべてのインバウンドを遮断
iptables -P INPUT DROP

# すでに確立されたセッションのパケットは許可（stateful）
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# ループバックは許可
iptables -A INPUT -i lo -j ACCEPT

# SSH、HTTP、HTTPSのみ許可
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 残りはデフォルトポリシー（DROP）により遮断される
```

これはAWSセキュリティグループでインバウンドルールとして「SSH 22番許可、HTTP 80番許可」とするのと本質的に同じです。

---

## 付録：ネットワークコマンド集

### macOS

```bash
# ルーティング
netstat -rn                              # ルーティングテーブル全体
route -n get default                     # デフォルトゲートウェイ詳細
route -n get <IP>                        # 特定IPの経路照会

# インターフェース
ifconfig                                 # インターフェース一覧とIP

# ARP / NDP
arp -a                                   # ARPテーブル (IP ↔ MAC マッピング)
ndp -a                                   # IPv6 Neighbor Discoveryテーブル

# DNS
scutil --dns                             # DNS設定確認
networksetup -listallhardwareports       # 物理インターフェース一覧

# ファイアウォール (PF)
sudo pfctl -sr                           # 現在のアクティブなファイアウォールルール
sudo pfctl -sa                           # 全体の状態
```

### Linux

```bash
# ルーティング
ip route show                            # ルーティングテーブル
ip route get 10.0.0.5                    # 特定IPの経路
ip rule show                             # Policy Routingルール

# iptablesルール（テーブル別）
sudo iptables -L -v -n --line-numbers    # filterテーブル
sudo iptables -t nat -L -v -n            # natテーブル
sudo iptables-save                       # 全ルールのダンプ

# nftables（iptablesの後継）
sudo nft list ruleset

# k8s関連ルールのみフィルタリング
sudo iptables-save | grep -i "KUBE\|FLANNEL\|CALICO"

# conntrack（NATマッピング状態）
sudo conntrack -L -d 10.96.0.1

# リアルタイムパケットトレース
sudo iptables -t raw -A PREROUTING -s 192.168.1.100 -j TRACE
dmesg -w | grep TRACE
```

### macOS PFとLinux iptablesの文法比較

```bash
# Linux iptables: 80番ポート遮断
iptables -A INPUT -p tcp --dport 80 -j DROP

# macOS PF: 同じ動作 (/etc/pf.conf)
block in proto tcp from any to any port 80
```

---

## おわりに

ここまで、ネットワークに関する基本的な内容を整理しました。この記事の目標は、ifconfig、ip route、iptablesといった基本的なシェルコマンドのすべての意味や詳細なロジックを理解できるように整理することです。

私自身もよく使っていながら、詳しく理解できていませんでした。しかし、新しいネットワークツールがなぜ登場したのかを理解するには、既存のツールの限界を知る必要があります。例えば、先ほど扱ったiptablesはチェーン内のルールを上から下へ**順番に評価**します。ルールが数十個程度なら問題ありませんが、KubernetesクラスターのようにServiceが数百～数千に増えるとiptablesのルールも爆発的に増加し、すべてのパケットがこの長いリストを線形探索しなければならないため、深刻なパフォーマンスボトルネックとなります。CiliumなどのツールがeBPFを活用してNetfilterをバイパスするアプローチを取った背景がまさにこれです。

次の投稿では、ネットワークにおけるこのような基本的なネットワークインターフェースの上に構築されたVLANやVXLAN、あるいは新しく登場したネットワークソリューションについて取り上げたいと思います。