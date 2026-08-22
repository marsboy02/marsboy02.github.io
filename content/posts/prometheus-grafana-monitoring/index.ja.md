---
title: "PrometheusとGrafanaでモニタリングシステムを構築する"
date: 2026-08-22
draft: false
tags: ["monitoring", "prometheus", "grafana", "observability"]
translationKey: "prometheus-grafana-monitoring"
summary: "オブザーバビリティの三本柱から出発してPrometheusのアーキテクチャとexporterの役割を整理し、node-exporterとRedisの監視をdocker-composeで実際に構築します。"
---

モニタリングシステムはサービス運用の土台です。サービスが正常に動いているのか、どんな問題が起きているのかをリアルタイムに把握できる必要があります。この記事ではオブザーバビリティ（Observability）の三本柱から出発し、PrometheusとGrafanaを使ったモニタリングシステムの構築方法を段階的に扱います。node-exporterによるLinuxシステムの監視とRedisの監視まで、実務経験をもとに整理しました。

進め方は次のとおりです。まずオブザーバビリティという概念を理論的に整理し、メトリクスを収集するPrometheusの構造とexporterの役割を見ていきます。その後、node-exporterとRedisを対象に監視環境を実際に構築し、最後にここまでの内容を一つのdocker-composeにまとめます。

## 1. オブザーバビリティの三本柱 (Three Pillars of Observability)

![Observability 3 pillars](images/three-pillars.png "Grafana Blog - What's next for observability?")

オブザーバビリティの三本柱とは、**メトリクス**（Metrics）、**ログ**（Logs）、**トレース**（Traces）を指します。AWS、IBM、Elasticなど各ベンダーの公式ドキュメントでも、この三つの指標の重要性が一貫して強調されています。

まとめると次のようになります。

| 区分             | 定義                                                     | 例                                             |
| ---------------- | -------------------------------------------------------- | ---------------------------------------------- |
| **メトリクス**   | システムの状態を時間に沿った数値の時系列として表現       | CPU使用率、メモリ使用量、ネットワークI/O       |
| **ログ**         | 特定のイベントに紐づくタイムスタンプ付きの構造化／非構造化データ | HTTPリクエストの記録、エラーメッセージ、crontabの実行結果 |
| **トレース**     | 一つのリクエストがシステムを通過する道のり全体           | API呼び出しの経路、サービス間の呼び出し時間    |

### 1.1 メトリクス (Metrics)

メトリクスは **システムの状態を時間に沿った数値の時系列** として表現したものです。主にCPU使用率、メモリ使用量、ネットワーク入出力といった指標を扱い、それぞれ `cpu_utilization`、`memory_utilization`、`network_io` のような名前で毎秒・毎分の数値を残していきます。

![iStat Menusで確認するシステムメトリクス](images/istat-menus.png "iStat Menus")

上の写真はiStat Menusというアプリケーションをインストールして、メニューバーから自分のマシンのCPU使用量を確認したものです。個人のマシンであれサーバーであれ、CPUとメモリをきちんと管理しなければならない点は変わりません。

ただ、どちらも重要だからといって、100%に達したときの結果が同じというわけではありません。CPUが100%になった瞬間にサービスが落ちると思われがちですが、実際にはシステムが非常に遅くなるだけで、メモリほど致命的ではありません。

メモリは事情が違います。メモリは物理的に決まったアドレス空間へデータを書き込むようになっており、あるプロセスが他のプロセスの領域を侵してデータを上書きすると、システム全体が崩れかねません。そのためメモリが100%に達すると、OSは速度を犠牲にする代わりにプロセスそのものを排除する方を選びます。

![プロセスごとに分かれたRAMのアドレス空間 — 自分の領域の外には書き込めない](images/memory-address-isolation.svg)

- **CPU 100%** : 待ち行列が膨れ上がってタイムアウトとエラーが増え、ひどい場合はサーバーが落ちます。
- **Memory 100%** : OSレベルの **OOM Killer**（Out-of-Memory Killer）がSIGKILLでプロセスを強制終了します。

![AWS CloudWatchのメトリクスダッシュボード](images/aws-cloudwatch.png "AWS Cloudwatch")

AWSではEC2やFargateのようなコンピューティングサービスを立ち上げると、CloudWatchがこうしたメトリクスを自動で収集してくれます。オンプレミス環境で同じ役割を担うのが **Prometheus** です。Prometheusは時系列データを蓄えるストアでありデータソースでもあり、設定ファイルに「どこへ行ってどの値を読んでこい」と書いておくと、そのアドレスを定期的に巡回して値を取ってきます。詳しい設定は後ほど扱います。

整理すると、exporterがメトリクスを公開し、Prometheusがそれを収集し、Grafanaが可視化する構造です。メトリクスのストアがPrometheusなら、ログのストアとしてはLokiが対になって語られます。

![exporter - Prometheus - Grafana アーキテクチャ](images/redis-exporter-architecture.png "exporter-Prometheus-Grafana アーキテクチャ")

ここで注目したいのは矢印の向きです。メトリクスはPrometheusがexporterを自分で巡回して値を取ってくる **Pull** 方式です。逆にログは、データを持っている側がストアへ押し込む **Push** 方式です。同じ観測データなのに収集の向きが逆であるという点は、すぐ下でもう一度取り上げます。

### 1.2 ログ (Logs)

ログは **出来事の詳細な記録** です。タイムスタンプ、レベル、メッセージ、フィールドなどで構成され、開発者が最初に触れることになる指標でもあります。

ログの代表的な活用例は次のとおりです。

- **HTTPリクエストの記録**: IP、User-Agent、レスポンスタイムなどを記録し、国別のアクセス傾向やエージェント別のトラフィックを分析
- **バッチ処理の記録**: crontabの実行結果、正常に処理されたかの確認
- **ドメイン別の監査ログ**: 特定のドメイン（例: users）へアクセスした際に必ず記録

![ログデータで構成したダッシュボード](images/log-dashboard.png "ステータスコード別のリクエスト量とエンドポイント別のリクエスト順位")

上のグラフはログをもとに作ったダッシュボードの一部です。ログを残すときは通常、OpenTelemetryというオブザーバビリティの標準に従い、リクエスト元のIP、アクセスされたエンドポイント、レスポンスのステータスコードなどを併せて記録します。

こうして集まった情報から、ステータスコードの分布や、どのエンドポイントにリクエストが集中しているかを把握できます。さらに、すべてのシステムでログのフォーマットを揃えておけば、一つのダッシュボードでサービスだけ切り替えて状態を確認でき、アラートも同じ基準で設定できます。ログシステムの価値は個々のログ一行よりも、この一貫性から生まれます。

### 1.3 メトリクスとログの収集方式の違い

先ほどメトリクスはPrometheusが自分で値を取りに行くPull方式だと述べましたが、ログは逆です。データを持っている側が中央のストアへ押し込むPush方式を使います。

![Lokiのログ収集構造](images/loki-push-architecture.png "https://grafana.com/oss/loki/")

Prometheusが設定ファイルに「どこへ行って値を読んでこい」と書いておくのに対し、ログはサーバーと一緒に立ち上がったAgentがログを読み取り、中央のストアへ送信します。上の図でも、各ノードに常駐するPromtailがログをLokiへ送り、GrafanaとAlertManagerがLokiに溜まったログを参照する構造が見て取れます。ログの保存と保管はこのように行われます。（なお最近ではPromtailではなくAlloyを使います。）

### 1.4 トレース (Traces)

トレースは **一つのリクエストがシステムを通過する道のり全体** を記録します。ユーザーが `/orders` を呼び出したなら、フロント → API → DB/キャッシュ → 外部決済API → メッセージキューといった全区間の時間と因果関係を一目で見せてくれます。

トレースの主要な用語を整理すると次のようになります。

- **Trace**: リクエスト単位の実行フロー全体。固有の `trace_id` で識別されます
- **Span**: Traceを構成する作業単位。「コントローラの処理」「DBクエリ」「外部HTTP呼び出し」のような一区間
- **Root Span**: 最も外側（開始）のスパン。通常は「HTTP POST /orders」のようにユーザーシナリオを代表します
- **Span Kind**: スパンの役割。`SERVER`（受信）、`CLIENT`（外部呼び出し）、`INTERNAL`（内部ロジック）

Spanに含まれる情報（Anatomy）には、名前、開始／終了時刻（duration）、属性（attributes）、イベント（event）、状態（status: OK/ERROR）、関係（親子・リンク）などがあります。

![Trace Waterfallの可視化](images/trace-waterfall.png "Honeycomb - Trace Explorer")

上のようにスパンの親子関係をウォーターフォール構造で可視化すると、各区間にかかった時間を一目で確認できます。これによりボトルネックを素早く特定し、原因を追跡できます。

### 1.5 三つの指標の相互補完関係

三つの指標を組み合わせて活用すると、高い水準のオブザーバビリティを維持できます。

- **メトリクス** で「何かおかしい」ことを検知し
- **ログ** で「どんなイベントが起きたのか」を把握し
- **トレース** で「どの区間で問題が起きたのか」を追跡します

ちなみに、CNCFが管理する **OpenTelemetry**（OTel）は、この三つのシグナルをベンダー中立に計測・収集・転送できるようにする標準（API/SDK + Collector + OTLP）です。DatadogやAWSなどのモニタリングツールも、このOTel標準を基盤に自社製品を組み上げています。

> **注意**: 機微情報（PIIなど）がメトリクス、ログ、トレースに含まれないよう、マスキングまたは削除する必要があります。

---

## 2. Prometheusのアーキテクチャ

オブザーバビリティの概念を整理したので、次は三本柱のうちメトリクスを担うPrometheusを見ていきます。Prometheusはオープンソースのモニタリングツールで、**時系列データ**（time-series data）の収集・保存・クエリに集中しています。単体で使われることは少なく、メトリクスを公開するexporterと、それを可視化するGrafanaと合わせて一つのスタックになります。exporterがまだ馴染みのない方は、2.4で改めて扱うのでそのまま読み進めてください。

### 2.1 メトリクスの種類 (Metric Types)

Prometheusで扱うメトリクスの種類は四つです。

| 種類          | 説明                                  | 例                                                |
| ------------- | ------------------------------------- | ------------------------------------------------- |
| **Counter**   | 増加のみ可能な値                      | リクエスト回数（`http_requests_total`）           |
| **Gauge**     | 増加・減少ともに可能                  | 現在のメモリ使用量（`node_memory_Active_bytes`）  |
| **Histogram** | 観測値をバケットに分けて集計          | レスポンスタイムの分布（`http_request_duration_seconds`） |
| **Summary**   | 一定期間の観測値の要約統計            | リクエスト時間のパーセンタイル                    |

四つとも、exporterがHTTPエンドポイント（`/metrics`）で公開し、Prometheusが定期的に取得します（Pull）。種類は値の意味を示す印でもあります。Counterは累積値なので、値そのものより増加率に意味があり、Gaugeは今この瞬間の値をそのまま読めばよいということです。

### 2.2 アーキテクチャ概要

![Prometheusのアーキテクチャ](images/prometheus-architecture.png "https://prometheus.io/docs/introduction/overview/")

公式ドキュメントのアーキテクチャ図をもとにデータの流れを整理すると、次のようになります。

1. **Exporter** が対象システムのメトリクスを `/metrics` エンドポイントで公開
2. **Prometheus Server** が `prometheus.yml` に設定されたtargetsを定期的にスクレイピングし、時系列DB（TSDB）に保存
3. **PromQL** で保存されたデータをクエリ
4. **Grafana** などの可視化ツールがPromQLを使ってダッシュボードを構成
5. **Alertmanager** を通じて特定の条件に対する通知を送信

肝はここまで繰り返してきた **Pullモデル** です。Prometheusがターゲットを自分で訪ねてデータを取ってくるので、監視対象の追加や削除が設定ファイルを数行直すだけで済みます。その代わり、Prometheusがターゲットへネットワーク的に到達できなければならないという制約が生まれます。

図の左にある **Pushgateway** は、この制約を回避するための仕組みです。バッチ処理のように寿命が短すぎてスクレイピングされる時間すらないプロセスは、終了直前にメトリクスをPushgatewayへ押し込み、PrometheusはPushgatewayを代わりにスクレイピングします。

### 2.3 サービスディスカバリとexporterのエコシステム

ターゲットを設定ファイルに一つずつ書いておく方式は、対象が固定されているうちは楽ですが、オートスケーリングでインスタンスが入れ替わり続ける環境では維持しきれません。そのためPrometheusは、上の図の **Service discovery** のようにKubernetes APIやファイル（`file_sd`）からターゲット一覧を動的に読み込む機能を提供しています。この記事ではノードが固定された環境を扱うので `static_configs` を使いますが、Kubernetes環境ならサービスディスカバリを使うのが事実上の基本です。

Prometheusのもう一つの強みは、幅広いexporterのエコシステムです。DockerHubで `exporter` と検索すれば、redis、node、mongodb、nginxなど、ほとんどのサーバー向けのexporterが見つかります。

![DockerHub Exporters](images/dockerhub-exporters.png "DockerHubで公開されている様々なexporter")

そしてGrafana Labsには、exporterごとにコミュニティが作ったダッシュボードのプリセットが公開されています。

![Grafana Labs Dashboards](images/grafana-labs-dashboards.png "Grafana Labsが提供するexporter別のダッシュボードプリセット")

ダッシュボードを一から作らずIDを入力するだけで取り込めるので、exporterを立ち上げた瞬間に使えるダッシュボードまでほぼ無料で手に入る形です。こうした理由から `exporter → Prometheus → Grafana` の組み合わせは、時系列データにもとづくモニタリングの標準スタックとして定着しました。ただし目的がログ収集なら、ELKスタック（Elasticsearch、Logstash、Kibana）や先に触れたLokiの方が適しています。

### 2.4 exporterとは何か

exporterのエコシステムを眺めてきましたが、肝心のexporterが何なのかはまだきちんと説明していません。exporterとは **監視対象の状態をPrometheusが読める形式に翻訳し、HTTPで公開する小さなサーバー** です。

翻訳が必要な理由は単純です。Redisは `INFO` コマンドで、Linuxカーネルは `/proc` ファイルシステムで、nginxは独自のstatusページで自分の状態を知らせます。どれも形式が違い、Prometheusはその方式を一つひとつ知っているわけではありません。そこで各システムの横にexporterを置き、そのシステム固有の方法で状態を読み取ったうえで、Prometheusの標準形式に変換して `/metrics` エンドポイントに載せておきます。Prometheusは形式が統一されたこのエンドポイントだけを知っていればよいわけです。

#### 公開フォーマット (Exposition Format)

`/metrics` が返すのは特別なプロトコルではなく、ただのプレーンテキストです。`curl` で開くと次のような内容が出てきます。

```
# HELP node_memory_MemAvailable_bytes Memory information field MemAvailable_bytes.
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 5.98016e+09
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 20385.53
node_cpu_seconds_total{cpu="0",mode="system"} 512.44
node_cpu_seconds_total{cpu="1",mode="idle"} 20401.17
```

一行が一つの時系列で、構造は `メトリクス名{ラベル="値"} 数値` と単純です。

- `# HELP` : このメトリクスが何なのかを説明するコメント
- `# TYPE` : 2.1で見た四つの種類のどれに当たるかを明示
- `{cpu="0",mode="idle"}` : **ラベル**（label）。同じ名前のメトリクスを複数の次元に分ける、キーと値のペアです。上の例のようにCPUコア別・モード別に値が別々に存在します。

ここで重要なのは、このレスポンスに **時間の情報がない** ことです。exporterはリクエストを受けた時点の現在値だけを計算して返し、何も保存しません。いつ測られた値なのかを記録し、過去の履歴を積み上げるのは、スクレイピングしたPrometheusの担当です。したがってexporterは状態を持たない軽量なプロセスであり、再起動しても失うデータがありません。

#### exporterを使う場合と直接計測する場合

exporterが必要なのは、対象のコードを自分で直せないときです。Redisやnginxに Prometheus用のコードを入れることはできないので、横にexporterを付けます。反対に自分が作ったアプリケーションなら、exporterを別に立てる理由はありません。各言語のクライアントライブラリを組み込んでアプリケーション自身が `/metrics` を公開する方が自然で、この場合は「注文作成数」「決済失敗率」といったビジネスメトリクスまで一緒に出せます。

整理すると、次のような選択肢があります。

| 状況                              | 方法                                    | 例                                           |
| --------------------------------- | --------------------------------------- | -------------------------------------------- |
| 修正できないサードパーティのシステム | 専用のexporterを横に立てる              | redis-exporter、mysqld-exporter              |
| ホストとOSそのもの                | ノードごとにexporterを常駐させる        | node-exporter                                |
| 自分が作ったアプリケーション      | クライアントライブラリで直接計測        | prometheus-client（Python）、Micrometer（JVM） |
| 外部からの応答可否だけを確認      | 対象に手を入れずプローブのみ実行        | blackbox-exporter                            |
| 寿命の短いバッチ処理              | 終了直前にPush                          | Pushgateway                                  |

結局のところ、Prometheusで何かを監視しようとした瞬間に最初にすべき問いは「この対象に合うexporterがすでにあるか」です。ほとんどの場合すでにあり、なければクライアントライブラリで自分で作ればよいわけです。次の章では、最も広く使われているexporterであるnode-exporterで、Linuxサーバーのメトリクスを実際に収集してみます。

---

## 3. node-exporterでシステムを監視する

node-exporterは、LinuxホストのCPU、メモリ、ディスク、ネットワークの状態を `/metrics` で公開するexporterです。対象がホストそのものなので、監視したいサーバーごとに一つ常駐させるのが原則です。

### 3.1 アーキテクチャ

![node-exporterのアーキテクチャ](images/node-exporter-architecture.svg "node-exporter → Prometheus → Grafana")

`exporter → Prometheus → Grafana` の順にデータが流れる構造です。docker-composeを使って三つのサービスをまとめて立ち上げます。

### 3.2 docker-compose.yml の構成

```yaml
version: "3"

networks:
  monitoring:
    driver: bridge

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - 9090:9090
    command:
      - "--storage.tsdb.path=/prometheus"
      - "--config.file=/etc/prometheus/prometheus.yml"
    restart: always
    networks:
      - monitoring

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    restart: always
    depends_on:
      - prometheus
    networks:
      - monitoring

  node_exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.rootfs=/rootfs"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    ports:
      - "9100:9100"
    networks:
      - monitoring

volumes:
  grafana-data:
  prometheus-data:
```

### 3.3 prometheus.yml の設定

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 15s
  evaluation_interval: 2m

  external_labels:
    monitor: "codelab-monitor"
    query_log_file: query_log_file.log

scrape_configs:
  - job_name: "monitoring-item"
    scrape_interval: 10s
    scrape_timeout: 10s
    metrics_path: "/metrics"
    scheme: "http"

    static_configs:
      - targets: ["prometheus:9090", "node_exporter:9100"]
        labels:
          service: "monitor"
```

ここで注意したいのは、**targetsに `localhost` ではなくdocker-composeのサービス名** を使うことです。同じcomposeネットワークの中ではサービス名で通信できます。ただし、ホストから直接アクセスする場合は `localhost:ポート` で到達できます。

```bash
docker-compose up -d
```

### 3.4 node-exporterが公開するメトリクス

`localhost:9100/metrics` にアクセスすると、node-exporterが出力しているメトリクスを直接確認できます。

![node-exporterのメトリクスエンドポイント](images/node-exporter-web.png)

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 2.1209e-05
go_gc_duration_seconds{quantile="0.25"} 5.3708e-05
...
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 9
```

### 3.5 Grafanaのデータソース接続

Grafana（`localhost:3000`）にアクセスし、デフォルトのアカウント（admin/admin）でログインしたあと、**Connections > Data source** でPrometheusを追加します。

![Grafana Data Sourceの追加](images/grafana-datasource.png "Grafana Data Sourceの設定")

URLを入力するときは、docker-compose環境なので `http://prometheus:9090` と指定します。

![Grafana Data Source Save & Test](images/grafana-datasource-save.png)

### 3.6 ダッシュボードのImport

Grafana Labsでコミュニティが共有しているダッシュボードのプリセットをimportできます。node-exporter向けで有名なダッシュボードIDは **1860** です。

- Grafana Labs Dashboards: https://grafana.com/grafana/dashboards/

**Dashboard > New > Import** でIDに `1860` を入力し、Prometheusのデータソースを選択します。

![Dashboard Import](images/grafana-import-1860.png)

![node-exporterのGrafanaダッシュボード](images/grafana-node-exporter-dashboard.png "完成したnode-exporterのダッシュボード")

### 3.7 リモートのLinuxノードにnode-exporterをインストールする

ローカルではなくリモートのサーバーにnode-exporterをインストールするには、バイナリを直接ダウンロードしてsystemdサービスとして登録します。

```bash
# 1. node-exporterのダウンロードとインストール
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.1.linux-amd64.tar.gz
mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
```

```bash
# 2. systemdサービスファイルの作成
cat <<EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

```bash
# 3. サービスの有効化と起動
systemctl enable node_exporter.service
systemctl start node_exporter.service
systemctl status node_exporter.service
```

リモートノードでnode-exporterが動き出したら、モニタリングサーバーの `prometheus.yml` でそのノードのIPとポートをtargetsに追加することで、メトリクスを収集できます。

---

## 4. Redisの監視

node-exporterでホストを覗いたなら、次はその上で動くミドルウェアを見る番です。前の章と比べて変わるのはexporter一つだけで、PrometheusとGrafanaの扱い方はそのままです。

### 4.1 アーキテクチャ

![Redis監視のアーキテクチャ](images/redis-monitoring-architecture.svg "Redis → redis-exporter → Prometheus → Grafana")

Redis監視の構成は `redis → redis-exporter → Prometheus → Grafana` です。redis-exporterが `INFO` コマンドでRedisの状態を読み取ってメトリクスに変換し `/metrics` で公開し、Prometheusがそれを収集します。3章のnode-exporterと比べると、監視対象がホストからRedisに変わっただけで、後ろの構造はそのままです。

- GitHub Example: https://github.com/marsboy02/redis-exporter-monitoring

### 4.2 docker-compose.yml

```yaml
services:
  redis:
    image: "redis:latest"
    container_name: "redis"
    ports:
      - "6379:6379"

  redis-exporter:
    image: "bitnami/redis-exporter:latest"
    container_name: "redis-exporter"
    environment:
      - REDIS_ADDR=redis:6379
    ports:
      - "9121:9121"
    depends_on:
      - redis

  prometheus:
    image: "prom/prometheus:latest"
    container_name: "prometheus"
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    depends_on:
      - redis-exporter

  grafana:
    image: "grafana/grafana:latest"
    container_name: "grafana"
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana

volumes:
  grafana-storage:
```

コンテナの依存関係は `redis <- redis-exporter <- prometheus <- grafana` の順で構成されます。

### 4.3 prometheus.yml

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "redis"
    static_configs:
      - targets: ["redis-exporter:9121"]
```

`scrape_interval` を5秒と短く設定して、Redisの変化を素早く捉えられるようにします。

### 4.4 redis-exporterのメトリクスを確認する

`localhost:9121/metrics` にアクセスすると、Redisに関する様々なメトリクスを確認できます。

![redis-exporterのメトリクス](images/redis-exporter-metrics.png "redis-exporterが公開するメトリクス")

主なメトリクスは次のとおりです。

```promql
# 接続中のクライアント数
redis_connected_clients

# メモリ使用量
redis_memory_used_bytes

# 秒あたりの処理コマンド数
rate(redis_commands_processed_total[1m])

# キーの個数
redis_db_keys
```

### 4.5 Grafanaダッシュボード

Grafana LabsからRedis向けのダッシュボードプリセット **11835** をimportします。

![Redis Dashboard Import](images/grafana-redis-import-11835.png)

平常時のダッシュボードは次のようになります。

![Redis Dashboard (idle)](images/grafana-redis-dashboard-idle.png)

### 4.6 負荷テスト

Redisに負荷をかけるための簡単なPythonスクリプトを実行してみます。

```python
import redis
import random
import string
import time

def random_string(length=10):
    letters = string.ascii_lowercase
    return ''.join(random.choice(letters) for i in range(length))

def main():
    r = redis.Redis(host='localhost', port=6379, db=0)
    try:
        while True:
            key = random_string(10)
            value = random_string(50)
            r.set(key, value)
            print(f"Set {key} -> {value}")
            time.sleep(0.01)
    except KeyboardInterrupt:
        print("Stopped by user")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    main()
```

```bash
python3 annoying-redis.py
```

![負荷スクリプトの実行](images/annoying-redis-script.png "Redisをいじめるスクリプト")

スクリプトを実行してからGrafanaのダッシュボードを見ると、Redisに負荷がかかったことを視覚的に確認できます。

![Redis Dashboard (負荷状態)](images/grafana-redis-dashboard-load.png "負荷テスト後のダッシュボードの変化")

このようにRedis以外でも、nginxやKafkaなど様々なサービスについて、DockerHubのexporterとGrafana Labsのダッシュボードプリセットを活用すれば、手軽にモニタリング環境を構築できます。

---

## 5. Lab: docker-composeで全体スタックを構成する

ここまで扱った内容をまとめて、Prometheus + Grafana + node-exporter + redis-exporterを一つのdocker-composeで構成する全体スタックの例です。

### 5.1 ディレクトリ構造

```
monitoring-lab/
├── docker-compose.yml
├── prometheus.yml
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml
└── annoying-redis.py
```

### 5.2 docker-compose.yml

```yaml
version: "3"

networks:
  monitoring:
    driver: bridge

services:
  # ---- Prometheus ----
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    command:
      - "--storage.tsdb.path=/prometheus"
      - "--config.file=/etc/prometheus/prometheus.yml"
    restart: always
    networks:
      - monitoring

  # ---- Grafana ----
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    restart: always
    depends_on:
      - prometheus
    networks:
      - monitoring

  # ---- Node Exporter ----
  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.rootfs=/rootfs"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    ports:
      - "9100:9100"
    networks:
      - monitoring

  # ---- Redis ----
  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - monitoring

  # ---- Redis Exporter ----
  redis_exporter:
    image: bitnami/redis-exporter:latest
    container_name: redis_exporter
    environment:
      - REDIS_ADDR=redis:6379
    ports:
      - "9121:9121"
    depends_on:
      - redis
    networks:
      - monitoring

volumes:
  grafana-data:
  prometheus-data:
```

### 5.3 prometheus.yml

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 15s
  evaluation_interval: 2m

scrape_configs:
  - job_name: "node"
    scrape_interval: 10s
    static_configs:
      - targets: ["node_exporter:9100"]
        labels:
          service: "node"

  - job_name: "redis"
    scrape_interval: 5s
    static_configs:
      - targets: ["redis_exporter:9121"]
        labels:
          service: "redis"

  - job_name: "prometheus"
    scrape_interval: 10s
    static_configs:
      - targets: ["prometheus:9090"]
```

### 5.4 Grafanaデータソースの自動プロビジョニング

`grafana/provisioning/datasources/prometheus.yml` ファイルを用意しておくと、Grafanaは起動時にPrometheusのデータソースを自動で接続します。

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

### 5.5 実行

```bash
# スタック全体を起動
docker-compose up -d

# 状態の確認
docker-compose ps

# ログの確認
docker-compose logs -f prometheus
```

実行後に確認できるエンドポイントは次のとおりです。

| サービス       | URL                             | 説明                          |
| -------------- | ------------------------------- | ----------------------------- |
| Prometheus     | `http://localhost:9090`         | PromQLクエリ、ターゲットの状態 |
| Grafana        | `http://localhost:3000`         | ダッシュボード（admin/admin） |
| Node Exporter  | `http://localhost:9100/metrics` | システムメトリクス            |
| Redis Exporter | `http://localhost:9121/metrics` | Redisのメトリクス             |

Grafanaにアクセスし、Dashboard Importで **1860**（node-exporter）と **11835**（Redis）をそれぞれimportすれば、モニタリング環境の構築は完了です。

---

## おわりに

ここまで、オブザーバビリティの三本柱から出発して、exporterとPrometheus、Grafanaを実際に立ち上げながらモニタリングスタックを構成してきました。整理すると構造そのものは単純です。対象の状態を標準形式に翻訳するexporterを立て、Prometheusにそのアドレスを教え、Grafanaでダッシュボードを読み込む。それでほとんどが終わります。

ただし、この記事で使った方式、つまり `prometheus.yml` の `static_configs` に対象を一つずつ書いておく方式は、対象が固定された環境でしか通用しません。KubernetesのようにPodが頻繁に立ち上がっては消える環境では、IPを書いておくこと自体に意味がありません。そのため実際の運用環境では2.3で触れたサービスディスカバリを使い、監視対象が自分自身をアノテーションで知らせるようにします。

とりわけGitOpsでクラスタを管理しているなら、アプリケーションのマニフェストにアノテーションを三行宣言しておくだけで済みます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      containers:
        - name: order-api
          image: registry.example.com/order-api:1.4.2
          ports:
            - containerPort: 8080
```

PrometheusはKubernetes APIでPodの一覧を取得しながら、`prometheus.io/scrape: "true"` が付いたPodだけを選んでターゲットに追加します。アノテーションを読み取って実際のターゲットに変換しているのはPrometheus設定の `kubernetes_sd_config` とrelabelルールであり、kube-prometheus-stackのようにPrometheus Operatorを使う場合は `ServiceMonitor` や `PodMonitor` といったCRDが同じ役割を担います。

ここで得られるのは利便性よりも一貫性です。新しいサービスをデプロイするPRにアノテーションが一緒に入っているので、マージされた瞬間にデプロイと監視の登録が同時に起こり、サービスを落とせばターゲットも一緒に消えます。モニタリングの設定を人が別に覚えて合わせておく必要がなくなるわけです。

モニタリングシステムを構築しながら最も強く感じたのは、オブザーバビリティはツールを付ける問題ではなく、何を見るかを決める問題だということです。exporterを立ててダッシュボードのプリセットをimportするところまでは半日で足りますが、そうして得た数十のグラフのうち、どの値が閾値を超えたときに実際に人を起こすべきなのかを決めるのは、まったく別の問題です。この問いは [SLOとSLI、そしてSLA]({{< ref "/posts/slo-sli-sla-error-budget" >}}) で扱った信頼性指標へそのままつながります。

次は、メトリクスだけを扱ったこの記事の範囲をログとトレースまで広げてみたいと思っています。1章で見た三本柱をLokiとTempoでGrafana一か所に集めれば、ダッシュボードで異常を検知し、同じ画面からログとトレースへ降りていく流れを作れます。この部分は別の記事としてまとめる予定です。

---

### References

- [Elastic] The 3 pillars of observability: https://www.elastic.co/blog/3-pillars-of-observability
- [Grafana] What's next for observability: https://grafana.com/blog/2019/10/21/whats-next-for-observability/
- [OpenTelemetry] What is OpenTelemetry: https://opentelemetry.io/docs/what-is-opentelemetry/
- [Honeycomb] Explore Traces: https://docs.honeycomb.io/investigate/analyze/explore-traces/
- [Prometheus] Metric Types: https://prometheus.io/docs/concepts/metric_types/
- [Prometheus] Architecture Overview: https://prometheus.io/docs/introduction/overview/
- [Prometheus] Kubernetes service discovery: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config
- [Grafana Labs] Dashboards: https://grafana.com/grafana/dashboards/
- [GitHub] node-exporter-monitoring: https://github.com/marsboy02/node-exporter-monitoring
- [GitHub] redis-exporter-monitoring: https://github.com/marsboy02/redis-exporter-monitoring
