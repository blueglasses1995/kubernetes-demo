# 07 - Linkerd: シンプルなサービスメッシュ

## 📚 この章で学ぶこと

Linkerdは、軽量で使いやすいサービスメッシュです。この章では、Linkerdを**徹底的に**学習し、完全にマスターします。

**重要**: Lensの章（06-lens）を完全に理解してからこの章に進んでください。この章はLinkerd**のみ**に集中します。

## 🎯 学習目標

- [ ] サービスメッシュの概念を理解する
- [ ] Linkerdのインストールと設定ができる
- [ ] アプリケーションをメッシュに追加できる
- [ ] トラフィック管理（リトライ、タイムアウト）ができる
- [ ] 可観測性（メトリクス、トレーシング）を理解する
- [ ] mTLSによるセキュリティを実装できる
- [ ] トラフィック分割（カナリアデプロイ）ができる
- [ ] Linkerd Vizを使いこなせる

## 📖 目次

1. [サービスメッシュとは](#1-サービスメッシュとは)
2. [Linkerdのアーキテクチャ](#2-linkerdのアーキテクチャ)
3. [演習1: Linkerdのインストール](#演習1-linkerdのインストール)
4. [演習2: アプリケーションのメッシュ化](#演習2-アプリケーションのメッシュ化)
5. [演習3: 可観測性とViz](#演習3-可観測性とviz)
6. [演習4: トラフィック管理](#演習4-トラフィック管理)
7. [演習5: セキュリティとmTLS](#演習5-セキュリティとmtls)
8. [演習6: トラフィック分割](#演習6-トラフィック分割)
9. [演習7: 本番運用](#演習7-本番運用)
10. [理解度チェック](#理解度チェック)

## 1. サービスメッシュとは

### マイクロサービスの課題

マイクロサービスアーキテクチャでは以下の課題があります：

- サービス間通信の信頼性（リトライ、タイムアウト）
- セキュリティ（認証、暗号化）
- 可観測性（メトリクス、トレーシング）
- トラフィック制御（カナリアデプロイ、A/Bテスト）

これらをアプリケーションコードに実装すると：
- コードが複雑になる
- 言語ごとに実装が必要
- 変更が困難

### サービスメッシュの解決策

サービスメッシュは、**インフラ層で**これらの機能を提供します：

```
アプリケーション
    ↓↑
サイドカープロキシ（Linkerd Proxy）
    ↓↑
ネットワーク
```

各Podにプロキシを注入し、すべての通信をプロキシ経由にします。

### Linkerdの特徴

- **軽量**: Rustで書かれた超軽量プロキシ
- **シンプル**: 設定が簡単
- **高速**: オーバーヘッドが最小限
- **セキュア**: デフォルトでmTLS
- **CNCF Graduated**: 本番環境で実績あり

## 2. Linkerdのアーキテクチャ

### コントロールプレーン

クラスター全体を管理するコンポーネント：

- **destination**: サービスディスカバリー
- **identity**: 証明書管理（mTLS）
- **proxy-injector**: サイドカー自動注入

### データプレーン

各Podに注入されるプロキシ：

- **linkerd2-proxy**: 超軽量プロキシ（Rust製）
- リクエストのメトリクス収集
- mTLS自動化
- リトライ・タイムアウト処理

---

## 演習1: Linkerdのインストール

**目標**: Linkerdをクラスターにインストールする

### ステップ1-1: CLIのインストール

```bash
# macOS / Linux
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# PATHに追加
export PATH=$PATH:$HOME/.linkerd2/bin

# インストール確認
linkerd version
```

### ステップ1-2: クラスターの準備

Kindで専用クラスターを作成（推奨）：

```bash
kind create cluster --name linkerd-demo --config ../environments/kind/cluster-configs/multi-node-cluster.yaml
kubectl config use-context kind-linkerd-demo
```

### ステップ1-3: 事前チェック

```bash
# クラスターが要件を満たしているか確認
linkerd check --pre

# すべて✓であることを確認
```

出力例：
```
✓ kubernetes-api: can initialize the client
✓ kubernetes-version: is running the minimum Kubernetes API version
✓ pre-kubernetes-setup: control plane namespace does not already exist
...
```

### ステップ1-4: Linkerdのインストール

```bash
# CRDのインストール
linkerd install --crds | kubectl apply -f -

# コントロールプレーンのインストール
linkerd install | kubectl apply -f -

# インストール確認
linkerd check
```

すべて✓になるまで待ちます（1-2分）。

### ステップ1-5: インストールの確認

```bash
# Linkerdのコンポーネントを確認
kubectl get pods -n linkerd

# 出力例：
# linkerd-destination-xxx
# linkerd-identity-xxx
# linkerd-proxy-injector-xxx
```

**✅ チェックポイント**:
- Linkerd CLIをインストールできる
- 事前チェックを実行できる
- Linkerdをインストールできる

---

## 演習2: アプリケーションのメッシュ化

**目標**: サンプルアプリケーションをLinkerdメッシュに追加する

### ステップ2-1: サンプルアプリのデプロイ

```bash
# emojiVotoサンプルアプリをインストール
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -

# Podを確認
kubectl get pods -n emojivoto
```

### ステップ2-2: メッシュ化前の確認

```bash
# サービスにアクセス
kubectl port-forward svc/web-svc -n emojivoto 8080:80

# ブラウザで http://localhost:8080 を開く
# 投票アプリが表示される（一部機能が壊れている）
```

### ステップ2-3: アプリケーションのメッシュ化

```bash
# emojivoto Namespaceにアノテーションを追加してメッシュ化
kubectl get deploy -n emojivoto -o yaml | linkerd inject - | kubectl apply -f -

# または、個別にメッシュ化
kubectl get deploy/web -n emojivoto -o yaml | linkerd inject - | kubectl apply -f -
```

### ステップ2-4: メッシュ化の確認

```bash
# Podを確認（各Podに2つのコンテナ）
kubectl get pods -n emojivoto

# サイドカーが注入されていることを確認
kubectl describe pod -n emojivoto web-xxx

# linkerd-proxyコンテナが追加されている
```

### ステップ2-5: メッシュ化のステータス

```bash
# Namespaceのメッシュ化状況
linkerd -n emojivoto check --proxy

# 統計情報
linkerd -n emojivoto stat deploy
```

**✅ チェックポイント**:
- サンプルアプリをデプロイできる
- アプリケーションをメッシュ化できる
- サイドカープロキシを確認できる

---

## 演習3: 可観測性とViz

**目標**: Linkerd Vizを使って可観測性を実現する

### ステップ3-1: Linkerd Vizのインストール

```bash
# Vizエクステンションをインストール
linkerd viz install | kubectl apply -f -

# インストール確認
linkerd viz check
```

### ステップ3-2: Vizダッシュボードの起動

```bash
# ダッシュボードを開く
linkerd viz dashboard &

# または
linkerd viz dashboard

# ブラウザが自動で開く（http://localhost:50750）
```

### ステップ3-3: ダッシュボードの探索

1. **Namespace一覧**: メッシュ化されたNamespaceを表示
2. **Deployments**: 各Deploymentのメトリクス
3. **サクセスレート**: リクエストの成功率
4. **RPS**: 秒あたりのリクエスト数
5. **レイテンシ**: P50、P95、P99

emojivotoアプリで投票を繰り返し、メトリクスが変化することを確認してください。

### ステップ3-4: Tapでライブトラフィックを観測

```bash
# リアルタイムでトラフィックを観測
linkerd viz tap deploy/web -n emojivoto

# 特定のパスをフィルタ
linkerd viz tap deploy/web -n emojivoto --path /api/vote
```

出力例：
```
req id=0:0 proxy=in  src=10.1.1.1:1234 dst=10.1.1.2:80 tls=true :method=GET
rsp id=0:0 proxy=in  src=10.1.1.1:1234 dst=10.1.1.2:80 tls=true :status=200
```

### ステップ3-5: トップコマンドでトラフィック分析

```bash
# リクエスト元のトップ
linkerd viz top deploy/web -n emojivoto

# リクエスト先のトップ
linkerd viz top deploy/emoji -n emojivoto --to deploy/voting
```

**✅ チェックポイント**:
- Linkerd Vizをインストールできる
- ダッシュボードを使いこなせる
- リアルタイムトラフィックを観測できる

---

## 演習4: トラフィック管理

**目標**: リトライとタイムアウトを設定してトラフィックを制御する

### ステップ4-1: ServiceProfileの作成

ServiceProfileは、サービスのルートとポリシーを定義します：

```bash
# 自動生成
linkerd viz profile -n emojivoto web-svc --tap deploy/web --tap-duration 10s | kubectl apply -f -
```

または、手動で作成 `manifests/serviceprofile.yaml`:

```yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: web-svc.emojivoto.svc.cluster.local
  namespace: emojivoto
spec:
  routes:
  - name: GET /api/vote
    condition:
      method: GET
      pathRegex: /api/vote
    isRetryable: true  # リトライ可能
    timeout: 1s        # タイムアウト1秒
```

### ステップ4-2: リトライポリシー

```yaml
# manifests/retry-policy.yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: voting-svc.emojivoto.svc.cluster.local
  namespace: emojivoto
spec:
  routes:
  - name: POST /emojivoto.v1.VotingService/VoteDoughnut
    condition:
      method: POST
      pathRegex: /emojivoto\.v1\.VotingService/VoteDoughnut
    isRetryable: true
    retryBudget:
      retryRatio: 0.2        # 20%までリトライ許容
      minRetriesPerSecond: 10
      ttl: 10s
```

適用:
```bash
kubectl apply -f manifests/retry-policy.yaml
```

### ステップ4-3: タイムアウトポリシー

```yaml
# 特定のルートにタイムアウトを設定
spec:
  routes:
  - name: GET /api/list
    timeout: 500ms
```

### ステップ4-4: ポリシーの確認

```bash
# ServiceProfileを確認
kubectl get serviceprofile -n emojivoto

# メトリクスでリトライが機能していることを確認
linkerd viz stat -n emojivoto svc/voting-svc --from deploy/web
```

**✅ チェックポイント**:
- ServiceProfileを作成できる
- リトライポリシーを設定できる
- タイムアウトを設定できる

---

## 演習5: セキュリティとmTLS

**目標**: Linkerdの自動mTLSを理解し、検証する

### ステップ5-1: mTLSの確認

Linkerdは**デフォルトで**メッシュ内のすべての通信をmTLSで暗号化します：

```bash
# メッシュ化されたDeploymentを確認
linkerd viz stat deploy -n emojivoto

# "MESHED" と "TLS" 列を確認
# TLSが有効になっている
```

### ステップ5-2: Tap Tapで TLSを確認

```bash
linkerd viz tap deploy/web -n emojivoto

# 出力の "tls=true" を確認
```

### ステップ5-3: mTLS証明書の確認

```bash
# Pod内の証明書を確認
kubectl exec -it -n emojivoto deploy/web -c linkerd-proxy -- ls /var/run/linkerd/identity/end-entity

# end-entity-crt.pem: クライアント証明書
# end-entity-key.pem: 秘密鍵
```

### ステップ5-4: Server Authorization Policy

特定の送信元のみを許可するポリシー：

```yaml
# manifests/server-authorization.yaml
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: voting-server
  namespace: emojivoto
spec:
  podSelector:
    matchLabels:
      app: voting-svc
  port: 8080
  proxyProtocol: HTTP/2
---
apiVersion: policy.linkerd.io/v1alpha1
kind: ServerAuthorization
metadata:
  name: voting-server-authz
  namespace: emojivoto
spec:
  server:
    name: voting-server
  client:
    meshTLS:
      serviceAccounts:
      - name: web        # webサービスアカウントのみ許可
```

適用:
```bash
kubectl apply -f manifests/server-authorization.yaml
```

**✅ チェックポイント**:
- mTLSが自動で有効になることを理解する
- TLS通信を確認できる
- Server Authorization Policyを設定できる

---

## 演習6: トラフィック分割

**目標**: カナリアデプロイメントを実装する

### ステップ6-1: SMI Traffic Splitのインストール

```bash
# SMI extensionをインストール
linkerd smi install | kubectl apply -f -
```

### ステップ6-2: カナリアデプロイメントの準備

新しいバージョンをデプロイ：

```yaml
# manifests/canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
  namespace: emojivoto
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-svc
      version: canary
  template:
    metadata:
      labels:
        app: web-svc
        version: canary
    spec:
      containers:
      - name: web-svc
        image: docker.io/buoyantio/emojivoto-web:v11  # 新バージョン
        ports:
        - containerPort: 8080
```

### ステップ6-3: TrafficSplitの作成

トラフィックを分割：

```yaml
# manifests/traffic-split.yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: web-split
  namespace: emojivoto
spec:
  service: web-svc
  backends:
  - service: web-svc
    weight: 90           # 90%を安定版に
  - service: web-canary
    weight: 10           # 10%をカナリアに
```

適用:
```bash
kubectl apply -f manifests/canary-deployment.yaml
kubectl apply -f manifests/traffic-split.yaml
```

### ステップ6-4: トラフィック分割の確認

```bash
# メトリクスを確認
linkerd viz stat -n emojivoto deploy

# web と web-canary へのトラフィックが 9:1 であることを確認
```

### ステップ6-5: 段階的なロールアウト

問題がなければ、徐々に重みを変更：

```yaml
# 50:50
backends:
- service: web-svc
  weight: 50
- service: web-canary
  weight: 50
```

最終的に100%カナリアに移行。

**✅ チェックポイント**:
- TrafficSplitを作成できる
- カナリアデプロイメントを実装できる
- トラフィックの重み付けができる

---

## 演習7: 本番運用

**目標**: 本番環境でLinkerdを運用するためのベストプラクティス

### ステップ7-1: HA（高可用性）構成

```bash
# HA構成でインストール
linkerd install --ha | kubectl apply -f -

# 各コンポーネントが複数レプリカで実行される
kubectl get deploy -n linkerd
```

### ステップ7-2: 証明書の管理

Linkerdの証明書は90日で期限切れになります：

```bash
# 証明書の期限を確認
linkerd check --proxy

# cert-managerと統合（推奨）
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

# Linkerdのcert-manager統合
linkerd cert install --issuer-name=linkerd-trust-anchor | kubectl apply -f -
```

### ステップ7-3: モニタリング統合

Prometheusにメトリクスをエクスポート：

```bash
# Prometheusをインストール
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# LinkerdメトリクスをPrometheusに統合
# ServiceMonitorを作成
```

### ステップ7-4: アップグレード戦略

```bash
# 現在のバージョンを確認
linkerd version

# 新しいバージョンをダウンロード
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

# CRDをアップグレード
linkerd upgrade --crds | kubectl apply -f -

# コントロールプレーンをアップグレード
linkerd upgrade | kubectl apply -f -

# 確認
linkerd check
```

### ステップ7-5: トラブルシューティング

```bash
# 診断情報を収集
linkerd diagnostics proxy-metrics -n emojivoto deploy/web

# プロキシログを確認
kubectl logs -n emojivoto deploy/web -c linkerd-proxy

# イベントを確認
kubectl get events -n linkerd --sort-by='.lastTimestamp'
```

**✅ チェックポイント**:
- HA構成を理解している
- 証明書管理を実装できる
- モニタリング統合ができる
- アップグレード手順を理解している

---

## 理解度チェック

以下の質問にすべて答えられますか？

### 基礎
- [ ] サービスメッシュの概念を説明できる
- [ ] Linkerdのアーキテクチャを理解している
- [ ] データプレーンとコントロールプレーンの違いを説明できる

### 実践
- [ ] Linkerdをインストールできる
- [ ] アプリケーションをメッシュ化できる
- [ ] Vizダッシュボードを使える
- [ ] ServiceProfileを作成できる

### 応用
- [ ] mTLSを理解し検証できる
- [ ] トラフィック分割ができる
- [ ] 本番環境向けの設定ができる

## 📝 まとめ

この章では、Linkerdを徹底的に学習しました：

✅ サービスメッシュの基本概念
✅ Linkerdのインストールと設定
✅ アプリケーションのメッシュ化
✅ 可観測性とVizダッシュボード
✅ トラフィック管理（リトライ、タイムアウト）
✅ 自動mTLSとセキュリティポリシー
✅ トラフィック分割とカナリアデプロイ
✅ 本番運用のベストプラクティス

## 🚀 次のステップ

すべての演習を完了し、理解度チェックに合格したら、次の章に進みましょう：

**次の章**: [08-service-mesh-istio](../08-service-mesh-istio/README.md) - Istioサービスメッシュ

**重要**: Linkerdを完全に理解してからIstioに進んでください。Istioはより複雑ですが、Linkerdで学んだ概念が活きます。

## 🔗 参考リンク

- [Linkerd公式ドキュメント](https://linkerd.io/docs/)
- [Linkerd GitHub](https://github.com/linkerd/linkerd2)
- [CNCF Linkerd](https://www.cncf.io/projects/linkerd/)
- [Buoyant（Linkerd開発元）](https://buoyant.io/)
