---
title: "GitOpsにおけるSecret管理戦略"
date: 2026-03-26
draft: true
tags: ["gitops", "kubernetes", "vault", "external-secrets", "devops", "security"]
translationKey: "gitops-secret-management"
summary: "GitOps環境でSecretを安全に管理する方法を解説します。Sealed Secrets、SOPS、External Secrets Operator、CSI Driverを比較し、Vault + ESO + ArgoCDの実践アーキテクチャとベストプラクティスを整理します。"
---

## Introduction

[前回の記事]({{< relref "/posts/about-gitops" >}})では、GitOpsの4つのコア原則とArgoCD基盤のPull型デプロイについて説明した。GitOpsの核心はGitリポジトリをSingle Source of Truth（SSOT）として使うことだが、ここで根本的な問題が生じる。**Secretをどう管理するか？**

UOSLIFEチームでもArgoCDがGitHubのgitopsリポジトリを参照するよう設定しており、その中にHashiCorp VaultをHelmチャートのベンダリング方式で立ち上げて運用している。Vaultを最初に初期化した際に5つのUnsealキーを受け取り、そのうち3つを入力してunsealする手順を踏んだのだが、その過程で「VaultのシークレットをArgoCDがどうやって実際の変数として読み取って使えるのか？」という疑問が生まれた。

結論から言えば、**External Secrets Operator**というツールを使って、Vaultに保存されたシークレットをKubernetes Secretに自動変換する仕組みを作ることができる。本記事ではこの経験をもとに、GitOps環境でSecretを安全に管理する方法論と各ツールの特徴を整理する。

```yaml
# このようにGitにプッシュしてはいけない
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # echo -n "password123" | base64
```

このマニフェストをGitにコミットした瞬間、`echo "cGFzc3dvcmQxMjM=" | base64 -d`だけで元のパスワードが露出してしまう。Kubernetes SecretはBase64エンコードに過ぎず、暗号化ではない。リポジトリがprivateであっても、Gitの分散特性上、cloneしたすべてのローカルコピーにパスワードが残るため、一度でも平文でコミットされたら、すでに漏洩したものと見なすべきだ。

---

## 1. GitOpsでSecretが難しい理由

GitOpsの原則を振り返ると、すべてを**宣言的にGitで定義**し、エージェントがそれをPullしてクラスターに適用する。問題は、SecretもKubernetesリソースである以上、GitOpsの原則に従えばGitに存在すべきだという点だ。

しかしセキュリティの原則は逆を要求する。機密情報はバージョン管理システムに平文で保存してはならない。一度でも平文のSecretがコミットされると、`git log`で履歴を追って取り消しても、すでに分散したすべてのコピーにその情報が残ってしまう。

この2つの原則の衝突を解決するために、大きく**2つのアーキテクチャアプローチ**が生まれた。

![alt text](images/secret-access-approach.png)

### アプローチ1: 暗号化したSecretをGitに保存

Secretを暗号化してからGitにコミットする方式だ。クラスター内のコントローラーが復号してKubernetes Secretに変換する。

![alt text](images/secret-gitops.png)

代表的なツールとして**Bitnami Sealed Secrets**と**Mozilla SOPS**がある。

### アプローチ2: Secretの参照（Reference）のみGitに保存

実際のSecret値は外部シークレットマネージャー（HashiCorp Vault、AWS Secrets Managerなど）に保存し、Gitには「どのシークレットマネージャーのどのキーを取得せよ」という参照のみをコミットする方式だ。

![alt text](images/secret-store.png)

代表的なツールとして**External Secrets Operator**（ESO）と**Secrets Store CSI Driver**がある。

> ArgoCD公式ドキュメントでも、**宛先クラスターでSecretを生成する方式**（アプローチ1・2どちらも該当）を推奨している。ArgoCDがマニフェスト生成時にSecretを注入する方式はRedisキャッシュに平文が保存されるセキュリティリスクがあり、Sync操作とSecretの更新が結びつくUX上の問題があるためだ。

## 2. Vaultの基本: Unsealメカニズムを理解する

External Secrets Operatorを扱う前に、Vault自体のセキュリティメカニズムをまず理解しておく必要がある。VaultはStartup時に**Sealed状態**で起動し、この状態ではいかなるシークレットも読み書きできない。

### Shamir's Secret Sharing

VaultのUnsealは**Shamir's Secret Sharing**という暗号学アルゴリズムに基づいている。初期化（`vault operator init`）時にマスターキーをN個の断片（key shares）に分割し、そのうちT個（threshold）を揃えることで復元できるよう設定する。

```bash
# Vault初期化 — 5つのキー断片、3つがthreshold
vault operator init -key-shares=5 -key-threshold=3
```

例えば`key-shares=5, key-threshold=3`であれば、5つのキーのうち3つを入力するとunsealされる。一人がマスターキー全体を持っているとsingle point of compromiseになるため、キー断片をそれぞれ異なる担当者に分散させることが本来の意図だ。

### 開発環境とプロダクションの違い

実際にVaultを初めてセットアップすると、一人が5つのキーをすべて受け取って直接3つを入力してunsealすることになるが、これは開発環境では一般的なものの、プロダクションではそのまま使うべきではない。Shamir's Secret Sharingの分散という意味が失われるからだ。

プロダクションでは通常、2つの方式のいずれかを選択する。

**キーの分散**: init時にキー断片をそれぞれ異なる担当者に渡す。5つのキーを5人が1つずつ持ち、再起動時に3人がそれぞれ自分のキーを入力する方式だ。しかし小規模チームでは、Podが再起動するたびに3人を集めてunsealするのは現実的に難しい。

**Auto Unseal**: AWS KMS、GCP Cloud KMSのような外部KMSにマスターキーの暗号化を委任することで、Vault Podが再起動した際に自動でunsealされる。これが事実上の標準だ。

```yaml
# Vault Helm values.yaml — AWS KMS Auto Unseal設定
server:
  ha:
    enabled: true
    config: |
      seal "awskms" {
        region     = "ap-northeast-2"
        kms_key_id = "arn:aws:kms:ap-northeast-2:123456:key/abcd-1234"
      }
```

Auto Unsealを使うとShamirのキー断片自体が不要になり、セキュリティ境界がKMSのIAMポリシーに移る。オンプレのk3s環境でも、AWS APIさえ呼び出せればIAM Roles Anywhereなどを通じて接続できる。

> なお、初期化時に受け取る**Root Token**は、初期のポリシー設定が完了したら`vault token revoke`で廃棄するのが正しい運用だ。以降はKubernetes Authのようなアイデンティティベースの認証のみを使うべきである。

---

## 3. Bitnami Sealed Secrets

Sealed SecretsはGitOps環境で最も手軽に始められるSecret管理ツールだ。外部のシークレットマネージャーを必要とせず、**公開鍵暗号**を活用してSecretを安全にGitに保存できる。

### 動作原理

Sealed Secretsは非対称暗号化（AES-256-GCM）を使用する。クラスターにインストールされたSealedSecretsコントローラーがキーペア（公開鍵/秘密鍵）を生成し、開発者は`kubeseal` CLIで公開鍵を使ってSecretを暗号化する。復号はクラスター内部のコントローラーのみが行える。

![alt text](images/sealed-secrets.png)

### インストールと使い方

```bash
# SealedSecretsコントローラーのインストール
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# kubeseal CLIのインストール (macOS)
brew install kubeseal
```

通常のKubernetes Secretを作成してから`kubeseal`で暗号化すればよい。

```bash
# 1. 通常のSecretマニフェストを作成
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
stringData:
  username: admin
  password: super-secret-password
EOF

# 2. kubesealで暗号化
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# 3. 元ファイルを削除し、暗号化済みファイルのみGitにコミット
rm secret.yaml
git add sealed-secret.yaml
git commit -m "feat: add encrypted db credentials"
```

生成されたSealedSecretマニフェストは次のようになる。

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO43...  # 암호화된 값
    password: AgCtr8bMOav3wHNEQ14FVg7B2K2...  # 암호화된 값
  template:
    metadata:
      name: db-credentials
      namespace: default
    type: Opaque
```

このファイルは暗号化されているので、Gitに安全に保存できる。クラスターにデプロイされると、SealedSecretsコントローラーが自動的に復号して通常のKubernetes Secretを生成する。

### メリットとデメリット

Sealed Secretsの強みは**外部依存がない**ことだ。Vaultのような別途インフラを用意せずに、コントローラー1つをインストールするだけですぐ使え、GitOpsワークフローとも自然に統合できる。

一方で**クラスターに依存する**という制限がある。各クラスターで固有のキーペアが生成されるため、同じSecretを複数のクラスターにデプロイしたい場合はクラスターごとに個別に暗号化しなければならない。マルチクラスター環境では管理の負担が急増する。Secretのローテーションも手動で再暗号化する必要があり、ランタイムのシークレット管理には適していない。

---

## 4. Mozilla SOPS (Secrets OPerationS)

SOPSはKubernetesに限定されない汎用の暗号化・復号ツールだ。YAML、JSON、ENV、INI、BINARY形式をサポートし、AWS KMS、GCP KMS、Azure Key Vault、age、PGPなど多様なキー管理バックエンドと統合できる。

### Sealed Secretsとの本質的な違い

Sealed Secretsはファイル全体を暗号化するのに対し、SOPSは**値（value）のみを選択的に暗号化**する。暗号化されたファイルの構造がそのまま読めるため、コードレビューや`git diff`での確認がはるかに容易だ。

もう一つの重要な違いは、暗号化・復号が行われる場所だ。Sealed Secretsはサーバーサイド（クラスター）で処理されるが、SOPSは**クライアントサイド**で処理される。CI/CDパイプラインで復号キーへのアクセス権限が必要になるということでもある。

```yaml
# SOPSで暗号化されたファイル — キーは読め、値のみが暗号化されている
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: ENC[AES256_GCM,data:7WgPOw==,iv:...,tag:...,type:str]
  password: ENC[AES256_GCM,data:K8vXzR5u...,iv:...,tag:...,type:str]
sops:
  kms:
    - arn: arn:aws:kms:ap-northeast-2:123456789012:key/xxxx-xxxx
  lastmodified: "2026-03-26T12:00:00Z"
  version: 3.9.0
```

### インストールと使い方

```bash
# SOPSのインストール (macOS)
brew install sops

# ageキーの生成 (PGPの現代的な代替)
age-keygen -o age.key
```

`.sops.yaml`設定ファイルで暗号化対象を細かく制御できる。

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.yaml$
    encrypted_regex: "^(data|stringData)$"
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

```bash
# 暗号化
sops -e secret.yaml > secret.enc.yaml

# インプレース編集（復号 → 編集 → 保存時に自動再暗号化）
sops secret.enc.yaml
```

### ArgoCDとの統合時の注意点

SOPSをArgoCDと組み合わせて使うには**KSOPS**（Kustomize SOPSプラグイン）を活用する必要がある。ArgoCDのカスタムイメージをビルドするか、Kustomizeプラグインを設定する追加作業が必要だ。FluxではSOPSがネイティブサポートされているため別途プラグインは不要だが、ArgoCD環境の場合はこの追加設定コストを考慮しなければならない。

### メリットとデメリット

SOPSの強みは**汎用性**だ。Kubernetes Secretだけでなく、Helmの`values.yaml`やTerraformの`.tfvars`など、あらゆる設定ファイルを暗号化できる。値のみを選択的に暗号化するため、`git diff`で変更内容を構造的に確認できる点も大きなメリットだ。クラウドKMSと統合すればキー管理の負担も軽減できる。

デメリットとしては、ArgoCDとの統合時に追加作業が必要な点、チームやクラスターが増えるにつれてキーの配布と管理が複雑になる点が挙げられる。

---

## 5. External Secrets Operator (ESO)

ESOは現在GitOps環境において**最も推奨される**Secret管理方式だ。UOSLIFEチームでもArgoCD + Vault環境でESOを通じてシークレットを管理している。実際のSecret値は外部シークレットマネージャーに保存し、Gitには「どこから何を取得せよ」という参照のみをコミットする。

### なぜESOが必要か

GitOpsリポジトリに実際のシークレット値を平文で入れることはできないが、ArgoCDはGitにあるマニフェストをそのままクラスターにsyncする仕組みのため、「Vaultから取り出せ」という指示を理解する手段がない。ESOがまさにこのギャップを埋める役割を担う。

全体のチェーンは次のようになる。

```
Git(ExternalSecret CR) → ArgoCD sync → ESO Controllerが検知
  → Vaultからfetch → K8s Secret生成 → PodがSecretをmount/envとして使用
```

![alt text](images/external-secret-operator.png)

### 主要CR（Custom Resource）の整理

ESOは複数のCRDを提供しており、役割によって使い分ける。

**1) SecretStore vs ClusterSecretStore**

どちらも「外部シークレットマネージャーへの接続方法」を定義するが、スコープが異なる。

SecretStoreは特定のnamespaceに紐づく。そのnamespaceのExternalSecretのみがこのstoreを参照できるため、チームごとにnamespaceを分けてアクセス範囲を制限したい場合に適している。

ClusterSecretStoreはクラスター全体を対象とする。どのnamespaceのExternalSecretからも参照できるため、小規模チームでVaultを共用する際に便利だ。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"          # KVシークレットエンジンのマウントパス
      version: "v2"           # KV v2使用時
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-role"     # Vaultに事前作成しておくrole
          serviceAccountRef:
            name: "external-secrets"
            namespace: "external-secrets"
```

ポイントは`auth`ブロックだ。VaultのKubernetes Auth Methodを使えば、ESOのServiceAccountトークンで認証できるため、別途トークンをGitに入れる必要がない。Vault側でこのroleにどのポリシーを付けるかでアクセス範囲を制御する。

**2) ExternalSecret**

「何を取得するか」を宣言するCRだ。最もよく使うリソースになる。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: production
spec:
  refreshInterval: 1h          # Vaultから定期的に再取得
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: my-app-secrets       # 生成されるK8s Secretの名前
    creationPolicy: Owner      # ExternalSecret削除時にSecretも削除
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: secret/data/production/my-app
        property: db_password
    - secretKey: API_KEY
      remoteRef:
        key: secret/data/production/my-app
        property: api_key
```

このマニフェストには**実際のSecret値がまったく含まれていない**。Vaultの`secret/data/production/my-app`パスから`db_password`と`api_key`を取得して`my-app-secrets`というK8s Secretを作成せよという参照があるだけだ。

`dataFrom`を使えば、あるパスにあるすべてのキーをまとめて取得することもできる。

```yaml
spec:
  dataFrom:
    - extract:
        key: secret/data/production/my-app
  # Vaultの該当パスにあるすべてのkey-valueがK8s SecretのDataに入る
```

Vaultにキーが多い場合に一つひとつマッピングしなくて済むので便利だ。

**3) ClusterExternalSecret**

ExternalSecretのクラスター全体版だ。複数のnamespaceに同じSecretをデプロイする必要がある場合に使う。ワイルドカードTLS証明書や共用のイメージプルシークレットを複数のnamespaceに配布する際に便利だ。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterExternalSecret
metadata:
  name: shared-tls-cert
spec:
  namespaceSelector:
    matchLabels:
      needs-tls: "true"       # このラベルがあるすべてのnamespaceに配布
  externalSecretSpec:
    refreshInterval: 24h
    secretStoreRef:
      name: vault-backend
      kind: ClusterSecretStore
    target:
      name: wildcard-tls
    data:
      - secretKey: tls.crt
        remoteRef:
          key: secret/data/shared/tls
          property: certificate
      - secretKey: tls.key
        remoteRef:
          key: secret/data/shared/tls
          property: private_key
```

**4) PushSecret（逆方向）**

逆にK8s Secret → Vaultへ書き込むCRだ。cert-managerが自動発行した証明書をVaultにバックアップするといった用途で使う。頻繁に使うものではないが、知っておくと便利だ。

### Vault Kubernetes Auth設定（前提条件）

ESOがVaultにアクセスするには、Vault側でKubernetes Authを有効化し、適切なroleとpolicyを設定する必要がある。これが全体のチェーンの核心的な前提条件だ。

```bash
# 1. VaultでKubernetes Authを有効化
vault auth enable kubernetes

# 2. K8s APIサーバー情報を登録
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc"

# 3. ESOが使用するroleを作成
vault write auth/kubernetes/role/eso-role \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=my-app-read \
  ttl=1h

# 4. 読み取り専用ポリシーを作成
vault policy write my-app-read - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF
```

この設定があって初めて、ESO → Vaultの認証チェーンが成立する。

---

## 6. マルチプロバイダー: Vault + AWS Secrets Manager併用

ESOの設計が優れている理由の一つは、**Providerを抽象化**していることだ。SecretStoreが「どこから取得するか」を、ExternalSecretが「何を取得するか」を分離しているため、バックエンドをVaultからAWS Secrets Managerに切り替えても、ExternalSecretは`remoteRef.key`の形式が少し変わるだけで、残りはほぼ同じだ。

### AWS Secrets Manager用のClusterSecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

Vault時の`spec.provider.vault`が`spec.provider.aws`に変わっただけだ。EKS環境であればIRSA（IAM Roles for Service Accounts）でスムーズに認証でき、k3sオンプレ環境ではIAM Roles AnywhereまたはaccessKeyID/secretAccessKeyを直接参照する方法を使う。

### ExternalSecretの違い

ExternalSecret側で変わるのは`secretStoreRef.name`と`remoteRef.key`の形式程度だ。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager      # providerのみ変更
    kind: ClusterSecretStore
  target:
    name: my-app-secrets
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/my-app     # AWS SMのシークレット名
        property: db_password      # JSON内のフィールド
```

### ハイブリッド構成

ClusterSecretStoreを複数作成することで、VaultとAWS Secrets Managerを同時に使用できる。ExternalSecretの`secretStoreRef.name`を切り替えるだけで、シークレットごとにどこから取得するかを選択できる。

![alt text](images/hybrid-eso.png)

Vaultをメインプロバイダーとして使いつつ、AWSリソース関連のシークレット（RDSパスワードなど）だけAWS Secrets Managerから直接取得するハイブリッド構成が現実的な選択だ。ESOはこの他にもGCP Secret Manager、Azure Key Vault、1Password、Dopplerなど多様なプロバイダーをサポートしており、構造はすべて同じで`spec.provider`以下のブロックが異なるだけだ。

---

## 7. Secrets Store CSI Driver

Secrets Store CSI Driverは、これまで紹介したツールとはアプローチが異なる。Kubernetes Secretを生成する代わりに、**Podにボリュームとして直接マウント**する方式だ。厳格なコンプライアンス要件によりetcdにSecretを保存すること自体が許可されていない環境で選択されるオプションだ。

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com"
    roleName: "my-app"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/db-credentials"
        secretKey: "password"
```

Podがスケジュールされると、CSI Driverが外部マネージャーからSecretを取得してファイルとしてマウントする。基本的にはボリュームマウント方式のため、環境変数でSecretを注入したりイメージプルシークレットとして使用したりするには別途sync設定が必要だ。DaemonSetとしてデプロイされるため、ノードごとにリソースを消費するという負担もある。

実務では**ESOで大部分のSecretを管理**し、特殊なコンプライアンス要件がある場合にのみCSI Driverを併用するパターンが一般的だ。

---

## 8. ツール比較と選択ガイド

![alt text](images/secret-tool-guide.png)

| 項目 | Sealed Secrets | SOPS | ESO | CSI Driver |
|------|---------------|------|-----|------------|
| **アプローチ** | Gitに暗号化して保存 | Gitに暗号化して保存 | 外部マネージャー参照 | 外部マネージャー参照 |
| **外部依存** | なし | KMS（任意） | 必須 | 必須 |
| **暗号化単位** | ファイル全体 | 値のみ選択的 | 該当なし | 該当なし |
| **Secretローテーション** | 手動再暗号化 | 手動再暗号化 | 自動（refreshInterval） | 自動 |
| **マルチクラスター** | クラスターごとに個別暗号化 | KMSで共有可能 | 同一マネージャーを参照 | 同一マネージャーを参照 |
| **ArgoCD統合** | 自然 | KSOPS必要 | 自然 | 自然 |
| **Flux統合** | 自然 | ネイティブサポート | 自然 | 自然 |
| **初期構築コスト** | 低 | 低 | 中〜高 | 中〜高 |
| **スケーラビリティ** | 低 | 中 | 高 | 高 |
| **マルチプロバイダー** | 不可 | KMS単位 | サポート | サポート |

### 状況別の推奨

**小規模チーム、単一クラスター、すぐに始めたいとき** → **Sealed Secrets**が適している。外部インフラ不要ですぐ使え、学習コストが低い。

**GitOps + IaC統合、Helm valuesの暗号化が必要なとき** → **SOPS**が適している。Kubernetesに限定されない汎用性が強みだ。ただしArgoCD環境ではKSOPS設定に追加の手間がかかる。

**プロダクション環境、マルチクラスター、自動ローテーションが必要なとき** → **External Secrets Operator**が適している。初期構築コストはかかるが、長期的に最も運用負担が少ない。ArgoCD公式ドキュメントでもこの方式を推奨している。

**etcdにSecretを保存してはならないコンプライアンス要件があるとき** → **CSI Driver**を選択するか、ESOと併用する。

---

## 9. 実践アーキテクチャ: ESO + Vault + ArgoCD

プロダクションで最もよく使われる組み合わせを整理する。UOSLIFEチームでもこれと同様の構成で運用している。

### GitOpsリポジトリ構造

```
gitops/
├── vault/                        # Vault Helmチャートのベンダリング
│   ├── Chart.yaml
│   ├── charts/
│   └── values.yaml
├── external-secrets/             # ESO Helmチャートのベンダリング
│   ├── Chart.yaml
│   ├── charts/
│   └── values.yaml
├── cluster-secret-stores/
│   └── vault-backend.yaml        # ClusterSecretStore
└── apps/
    └── my-app/
        ├── base/
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── externalsecret.yaml   # Secret参照マニフェスト
        └── overlays/
            ├── dev/
            │   └── externalsecret-patch.yaml  # Vaultパスをdevに
            └── prod/
                └── externalsecret-patch.yaml  # Vaultパスをprodに
```

VaultもESOも同様にHelmチャートのベンダリング方式でgitopsリポジトリで管理する。環境ごとの分離はKustomize overlayでExternalSecretの`remoteRef.key`パスを変えるだけでよい。`secret/data/dev/my-app` → `secret/data/prod/my-app`のように、構造は同じでパスだけが異なる。

### ArgoCD Applicationのデプロイ順序

ArgoCDでこの構造を使う場合、**デプロイ順序が重要**だ。ESOのCRDが先にインストールされていないとExternalSecretをデプロイできないためだ。

```
ステップ1: ESOのインストール（CRD含む）
ステップ2: ClusterSecretStoreのデプロイ
ステップ3: ExternalSecretを含むアプリのデプロイ
```

ArgoCDの**sync wave**でこの順序を制御できる。SentryやほかのCRDベースのアプリを立ち上げる際に使ったのと同じパターンだ。

```yaml
# ESO ApplicationにSync wave -2を付与
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-2"

# ClusterSecretStoreにSync wave -1を付与
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

# その他のアプリはデフォルト値（0）でデプロイ
```

### デプロイE2Eフロー

1. 開発者がVault UIまたはCLIで`secret/data/production/my-app`にSecret値を保存する。
2. GitOpsリポジトリにExternalSecretマニフェストをコミットする。
3. ArgoCDが変更を検知してExternalSecretをクラスターにsyncする。
4. ESO ControllerがVault Kubernetes Authで認証後、Secret値を取得する。
5. ESOがKubernetes Secret `my-app-secrets`を生成する。
6. Deploymentが該当Secretを`envFrom`で参照する。

Secret値がVaultで変更されると、ESOの`refreshInterval`に応じて自動的にKubernetes Secretが更新される。アプリケーションの再起動が必要な場合は**Stakater Reloader**を合わせて使うことで、Secret変更時に自動でRolling Restartをトリガーできる。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  annotations:
    reloader.stakater.com/auto: "true"  # Secret変更時に自動再起動
spec:
  template:
    spec:
      containers:
        - name: api-server
          envFrom:
            - secretRef:
                name: my-app-secrets
```

---

## 10. ベストプラクティスまとめ

GitOps環境でSecretを管理する際に共通して守るべき原則を整理する。

### Gitリポジトリでの原則

**平文のSecretは絶対にコミットしない。** pre-commitフックやCIのステージで`detect-secrets`のようなツールを使い、平文Secretのコミットを自動的にブロックするとよい。

```yaml
# .pre-commit-config.yaml の例
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

**Secret関連のマニフェストは別ディレクトリに分離する。** RBACやコードレビューポリシーをSecret関連ファイルにより厳格に適用できる。

### 運用での原則

**Secretローテーションポリシーを策定する。** 外部シークレットマネージャーを使う場合は自動ローテーションを設定し、暗号化方式を使う場合は定期的な再暗号化のスケジュールを設ける。

**最小権限の原則を適用する。** VaultのKubernetes Authでroleごとにpolicyを分離し、各アプリケーションが自分に必要なSecretのみにアクセスできるようにする。

```bash
# my-appは自分のパスのみ読み取れるポリシー
vault policy write my-app-read - <<EOF
path "secret/data/production/my-app" {
  capabilities = ["read"]
}
EOF
```

**Vault Root Tokenは初期設定後に必ず廃棄する。** 以降はKubernetes Authのようなアイデンティティベースの認証のみを使う。

### ArgoCD特有のプラクティス

ArgoCD公式ドキュメントでは**宛先クラスターでSecretを管理する方式**を推奨している。理由は3つある。

第一に、ArgoCDがSecret値に直接アクセスする必要がなくなり、セキュリティリスクが減る。ArgoCDはマニフェストをRedisキャッシュに平文で保存するため、マニフェスト生成ステージでSecretを注入するとRedisに平文Secretがキャッシュされてしまう。

第二に、Secretの更新がアプリのSyncと分離される。ESOのようなOperator方式ではSecretが独立して更新されるため、無関係なデプロイ中に意図せずSecretが変更されるリスクがない。

第三に、Rendered Manifestsパターンと互換性がある。このパターンはGitOpsでますます主流になっており、マニフェスト生成ステージでSecretを注入する方式とは互換性がない。

---

## おわりに

GitOpsにおけるSecret管理は、「Gitにすべてを保存する」という原則と「機密情報を保護する」という原則のバランスを見つける問題だ。[前回の記事]({{< relref "/posts/about-gitops" >}})で扱ったGitOpsのコア原則 — 宣言的定義、バージョン管理、Pull型デプロイ、継続的調整 — を維持しながらSecretを安全に扱うためには、適切なツールの選択と運用プラクティスが欠かせない。

まとめると次のようになる。

**小規模でいち早く始めたいならSealed Secrets**、**プロダクションでスケーラビリティと自動化が必要ならExternal Secrets Operator + Vault（またはAWS Secrets Manager）**の組み合わせを推奨する。ArgoCDを使っているなら、公式ドキュメントで推奨されているとおり、Operatorベースの宛先クラスターSecret管理方式を採用することが、長期的に最も安全でメンテナンスしやすい選択だ。

ESOはProvider抽象化のおかげで、Vaultから始めてもその後AWS Secrets ManagerやGCP Secret Managerへの移行やハイブリッド構成が自由にできる。これがESOを選ぶ最大の理由でもある。
