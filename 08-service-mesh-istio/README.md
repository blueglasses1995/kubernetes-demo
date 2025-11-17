# 08 - Istio: エンタープライズサービスメッシュ

## 📚 この章で学ぶこと

Istioは、最も機能豊富で強力なサービスメッシュです。Linkerdで学んだ概念を基に、Istioの高度な機能を**徹底的に**学習します。

**重要**: Linkerdの章（07-service-mesh-linkerd）を完全に理解してからこの章に進んでください。

## 🎯 学習目標

- [ ] Istio Ambientモードを理解する
- [ ] Istioのインストールと設定ができる
- [ ] Gateway APIを使いこなせる
- [ ] 高度なトラフィック管理（A/Bテスト、ミラーリング）ができる
- [ ] Kiali、Jaeger、Grafanaを使った可観測性を実装できる
- [ ] セキュリティポリシーを設定できる
- [ ] マルチクラスター構成を理解する

## 📖 目次

1. [Istioとは](#1-istioとは)
2. [Ambient vs Sidecar](#2-ambient-vs-sidecar)
3. [演習1: Istioのインストール（Ambient）](#演習1-istioのインストールambient)
4. [演習2: Bookinfo アプリケーション](#演習2-bookinfo-アプリケーション)
5. [演習3: Gateway APIとIngress](#演習3-gateway-apiとingress)
6. [演習4: 高度なトラフィック管理](#演習4-高度なトラフィック管理)
7. [演習5: Kiali、Jaeger、Grafana](#演習5-kialiaegerrafana)
8. [演習6: セキュリティポリシー](#演習6-セキュリティポリシー)
9. [演習7: 本番運用](#演習7-本番運用)
10. [理解度チェック](#理解度チェック)

## 1. Istioとは

### Linkerd vs Istio

| 特徴 | Linkerd | Istio |
|------|---------|-------|
| 複雑さ | シンプル | 複雑 |
| オーバーヘッド | 最小限 | やや大きい |
| 機能 | 基本的 | 豊富 |
| 拡張性 | 限定的 | 高い |
| 学習曲線 | 緩やか | 急 |

### Istioの主な機能

1. **Ambient Mesh**: サイドカーレスアーキテクチャ（新機能）
2. **高度なトラフィック制御**: A/Bテスト、ミラーリング、フォルトインジェクション
3. **豊富な可観測性**: Kiali、Jaeger、Prometheus、Grafana統合
4. **強力なセキュリティ**: 認可ポリシー、外部認証
5. **マルチクラスター**: 複数クラスター間のメッシュ

## 2. Ambient vs Sidecar

### 従来のSidecarモード

```
Pod
├── アプリケーションコンテナ
└── Envoy プロキシ（サイドカー）
```

**欠点**:
- Podごとにプロキシが必要（リソース消費）
- アプリケーションの再起動が必要

### 新しいAmbientモード（推奨）

```
Pod
└── アプリケーションコンテナ

ノード上のztunnel（共有プロキシ）
```

**利点**:
- リソース効率的
- アプリケーションの変更不要
- 段階的な導入が容易

---

## 演習1: Istioのインストール（Ambient）

**目標**: Istio Ambientモードをインストールする

### ステップ1-1: istioctl のインストール

```bash
# Istio CLIのインストール
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# バージョン確認
istioctl version
```

### ステップ1-2: クラスターの準備

Kind推奨（Ambientモード対応）：

```bash
kind create cluster --name istio-demo --config ../environments/kind/cluster-configs/multi-node-cluster.yaml
kubectl config use-context kind-istio-demo
```

### ステップ1-3: Istio Ambientのインストール

```bash
# Ambientプロファイルでインストール
istioctl install --set profile=ambient -y

# インストール確認
kubectl get pods -n istio-system

# 以下が実行されている：
# istiod: コントロールプレーン
# ztunnel: L4プロキシ（各ノード）
```

### ステップ1-4: 検証

```bash
# Istioの状態を確認
istioctl verify-install

# Ambient statusを確認
kubectl get daemonset -n istio-system
```

**✅ チェックポイント**:
- istioctl をインストールできる
- Istio Ambient をインストールできる
- コンポーネントを確認できる

---

## 演習2: Bookinfo アプリケーション

**目標**: Istioのサンプルアプリケーションをデプロイする

### ステップ2-1: アプリケーションのデプロイ

```bash
# Bookinfoアプリケーションをデプロイ
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# Podが起動するのを待つ
kubectl get pods
```

Bookinfoアプリの構成：
- **productpage**: フロントエンド（Python）
- **details**: 書籍詳細（Ruby）
- **reviews**: レビュー（Java） - 3つのバージョン
  - v1: レーティングなし
  - v2: 黒い星
  - v3: 赤い星
- **ratings**: レーティング（Node.js）

### ステップ2-2: Ambientメッシュに追加

```bash
# Namespaceにラベルを追加してAmbient meshに参加
kubectl label namespace default istio.io/dataplane-mode=ambient

# 確認
kubectl get ns default --show-labels
```

### ステップ2-3: Gatewayのセットアップ

```bash
# Ingress Gatewayを作成
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Gatewayを確認
kubectl get gateway
```

### ステップ2-4: アプリケーションへのアクセス

```bash
# Gatewayのアドレスを取得（Kind環境）
kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80

# ブラウザで http://localhost:8080/productpage にアクセス
# ページを何度かリロードすると、reviewsのバージョンが変わる
```

**✅ チェックポイント**:
- Bookinfoアプリケーションをデプロイできる
- Ambientメッシュに追加できる
- Gateway経由でアクセスできる

---

## 演習3: Gateway APIとIngress

**目標**: Istio Gateway APIを使いこなす

### ステップ3-1: VirtualServiceの作成

すべてのトラフィックをreviews v1に固定：

```yaml
# manifests/reviews-v1.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

### ステップ3-2: DestinationRuleの作成

```yaml
# manifests/destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

適用：
```bash
kubectl apply -f manifests/reviews-v1.yaml
kubectl apply -f manifests/destination-rule.yaml

# リロードしても常にv1（星なし）が表示される
```

**✅ チェックポイント**:
- VirtualServiceを作成できる
- DestinationRuleを作成できる
- トラフィックルーティングができる

---

## 演習4: 高度なトラフィック管理

**目標**: A/Bテスト、カナリア、ミラーリングを実装する

### ステップ4-1: ユーザーベースルーティング

特定のユーザーのみ v2 にルーティング：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

Bookinfoで "jason" としてログインすると v2 が表示される。

### ステップ4-2: トラフィック分割（カナリア）

50%ずつに分割：

```yaml
spec:
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

### ステップ4-3: トラフィックミラーリング

本番トラフィックをテスト環境にミラー：

```yaml
spec:
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
    mirror:
      host: reviews
      subset: v2
    mirrorPercentage:
      value: 100.0
```

### ステップ4-4: フォルトインジェクション

意図的に遅延やエラーを注入してテスト：

```yaml
spec:
  http:
  - fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
```

**✅ チェックポイント**:
- ユーザーベースルーティングができる
- カナリアデプロイメントを実装できる
- トラフィックミラーリングができる
- フォルトインジェクションができる

---

## 演習5: Kiali、Jaeger、Grafana

**目標**: 可観測性ツールを使いこなす

### ステップ5-1: アドオンのインストール

```bash
# Kiali、Prometheus、Grafana、Jaegerをインストール
kubectl apply -f samples/addons/

# Podが起動するのを待つ
kubectl get pods -n istio-system
```

### ステップ5-2: Kialiダッシュボード

```bash
# Kialiを開く
istioctl dashboard kiali

# 機能：
# - サービスグラフ
# - トラフィックフロー
# - エラー率
# - レイテンシ
```

### ステップ5-3: Jaegerトレーシング

```bash
# Jaegerを開く
istioctl dashboard jaeger

# 分散トレーシング：
# - リクエストの経路
# - 各サービスの処理時間
# - ボトルネックの特定
```

### ステップ5-4: Grafanaメトリクス

```bash
# Grafanaを開く
istioctl dashboard grafana

# Istio専用ダッシュボード：
# - Service Dashboard
# - Workload Dashboard
# - Performance Dashboard
```

**✅ チェックポイント**:
- Kialiでサービスグラフを確認できる
- Jaegerで分散トレーシングができる
- Grafanaでメトリクスを確認できる

---

## 演習6: セキュリティポリシー

**目標**: Istioのセキュリティ機能を実装する

### ステップ6-1: Peer Authentication（mTLS）

```yaml
# manifests/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # 強制mTLS
```

### ステップ6-2: Authorization Policy

特定のサービスへのアクセスを制限：

```yaml
# manifests/authz-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ratings-viewer
  namespace: default
spec:
  selector:
    matchLabels:
      app: ratings
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
```

### ステップ6-3: Request Authentication（JWT）

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-example
spec:
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.20/security/tools/jwt/samples/jwks.json"
```

**✅ チェックポイント**:
- mTLSを強制できる
- Authorization Policyを設定できる
- JWT認証を実装できる

---

## 演習7: 本番運用

**目標**: 本番環境でIstioを運用する

### ステップ7-1: HA構成

```bash
# 本番環境用プロファイル
istioctl install --set profile=production -y
```

### ステップ7-2: リソース管理

```yaml
# istiodのリソース設定
spec:
  values:
    pilot:
      resources:
        requests:
          cpu: 500m
          memory: 2048Mi
```

### ステップ7-3: アップグレード戦略

```bash
# カナリアアップグレード
istioctl install --set revision=1-20-0

# 段階的に移行
kubectl label namespace default istio.io/rev=1-20-0 --overwrite

# 古いバージョンを削除
istioctl uninstall --revision=1-19-0
```

**✅ チェックポイント**:
- HA構成を理解している
- カナリアアップグレードができる
- 本番運用のベストプラクティスを理解している

---

## 理解度チェック

### 基礎
- [ ] Ambient vs Sidecarの違いを説明できる
- [ ] VirtualServiceとDestinationRuleの違いを理解している

### 実践
- [ ] Istioをインストールできる
- [ ] トラフィックルーティングができる
- [ ] 可観測性ツールを使える

### 応用
- [ ] 高度なトラフィック管理ができる
- [ ] セキュリティポリシーを設定できる
- [ ] 本番運用を理解している

## 📝 まとめ

✅ Istio Ambientモードの理解
✅ 高度なトラフィック管理
✅ Kiali、Jaeger、Grafanaによる可観測性
✅ セキュリティポリシー
✅ 本番運用のベストプラクティス

## 🚀 次のステップ

**次の章**: [09-advanced-orchestration](../09-advanced-orchestration/README.md) - KarpenterとCrossplane

**重要**: Istioを完全に理解してから次に進んでください。

## 🔗 参考リンク

- [Istio公式ドキュメント](https://istio.io/latest/docs/)
- [Istio Ambient Mesh](https://istio.io/latest/docs/ops/ambient/)
- [Kiali](https://kiali.io/)
- [Jaeger](https://www.jaegertracing.io/)
