# 05 - Helm: Kubernetesパッケージマネージャー

## 📚 この章で学ぶこと

HelmはKubernetesのパッケージマネージャーです。この章では、Helmを**徹底的に**学習し、完全にマスターします。

**重要**: この章を完全に理解してから次の章（06-lens）に進んでください。

## 🎯 学習目標

- [ ] Helmの基本概念を理解する
- [ ] Helmコマンドを使いこなせる
- [ ] 既存のChartをインストール・管理できる
- [ ] 独自のChartを作成できる
- [ ] テンプレート機能を使いこなせる
- [ ] Values.yamlを理解し活用できる
- [ ] Helmリポジトリを管理できる
- [ ] 本番環境でHelmを運用できる

## 📖 目次

1. [Helmとは](#1-helmとは)
2. [Helmのインストール](#2-helmのインストール)
3. [Helmの基本概念](#3-helmの基本概念)
4. [演習1: 既存Chartの使用](#演習1-既存chartの使用)
5. [演習2: Chart作成の基礎](#演習2-chart作成の基礎)
6. [演習3: テンプレート機能](#演習3-テンプレート機能)
7. [演習4: Values管理](#演習4-values管理)
8. [演習5: 依存関係の管理](#演習5-依存関係の管理)
9. [演習6: リポジトリ管理](#演習6-リポジトリ管理)
10. [演習7: 本番運用パターン](#演習7-本番運用パターン)
11. [理解度チェック](#理解度チェック)

## 1. Helmとは

### なぜHelmが必要か

Kubernetesでアプリケーションをデプロイする際：
- 複数のYAMLファイルを管理する必要がある
- 環境ごとに値を変更する必要がある
- バージョン管理が複雑
- ロールバックが困難

**Helmはこれらの問題を解決します**

### Helmの3つの主要概念

1. **Chart**: アプリケーションのパッケージ（YAMLテンプレート集）
2. **Release**: Chartのインストールインスタンス
3. **Repository**: Chartの保管場所

### Helmのアーキテクチャ

```
Helm CLI (あなた)
    ↓
Kubernetes API
    ↓
Kubernetes Cluster
```

Helm v3からTiller（サーバー側コンポーネント）は不要になりました。

## 2. Helmのインストール

### macOS
```bash
brew install helm
```

### Linux
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Windows
```powershell
choco install kubernetes-helm
```

### インストール確認
```bash
helm version
# version.BuildInfo{Version:"v3.x.x", ...}
```

## 3. Helmの基本概念

### Chartの構造

```
mychart/
├── Chart.yaml          # Chartのメタデータ
├── values.yaml         # デフォルト設定値
├── charts/             # 依存するChart
├── templates/          # Kubernetesマニフェストテンプレート
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # テンプレートヘルパー
│   └── NOTES.txt       # インストール後のメッセージ
└── README.md
```

---

## 演習1: 既存Chartの使用

**目標**: 既存のChartを使ってアプリケーションをデプロイする

### ステップ1-1: リポジトリの追加

```bash
# 公式リポジトリを追加
helm repo add bitnami https://charts.bitnami.com/bitnami

# リポジトリ一覧を確認
helm repo list

# リポジトリを更新
helm repo update
```

### ステップ1-2: Chartの検索

```bash
# nginxを検索
helm search repo nginx

# すべてのバージョンを表示
helm search repo nginx --versions

# Chart情報を表示
helm show chart bitnami/nginx
helm show values bitnami/nginx
```

### ステップ1-3: Chartのインストール

```bash
# nginxをインストール
helm install my-nginx bitnami/nginx

# インストール状況を確認
helm list
helm status my-nginx

# Podを確認
kubectl get pods
kubectl get services
```

### ステップ1-4: カスタム値でインストール

```bash
# values.yamlを生成
helm show values bitnami/nginx > nginx-values.yaml

# values.yamlを編集（例: レプリカ数を変更）
# replicaCount: 3

# カスタム値でインストール
helm install my-nginx-custom bitnami/nginx -f nginx-values.yaml

# または、コマンドラインで値を指定
helm install my-nginx-cli bitnami/nginx --set replicaCount=3
```

### ステップ1-5: アップグレードとロールバック

```bash
# アップグレード
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# リビジョン履歴を確認
helm history my-nginx

# ロールバック
helm rollback my-nginx 1

# 特定のリビジョンにロールバック
helm rollback my-nginx 2
```

### ステップ1-6: アンインストール

```bash
# アンインストール
helm uninstall my-nginx

# リリース一覧（削除済みも含む）
helm list --all
```

**✅ チェックポイント**:
- Chartのインストール・アップグレード・ロールバック・削除ができる
- カスタム値を指定してインストールできる

---

## 演習2: Chart作成の基礎

**目標**: シンプルな独自Chartを作成する

### ステップ2-1: Chartのスケルトン作成

```bash
# 新しいChartを作成
helm create myfirstapp

# 構造を確認
tree myfirstapp
cd myfirstapp
```

### ステップ2-2: Chart.yamlの編集

`Chart.yaml`を確認・編集:

```yaml
apiVersion: v2
name: myfirstapp
description: My First Helm Chart
type: application
version: 0.1.0
appVersion: "1.0"
maintainers:
  - name: Your Name
    email: your.email@example.com
```

### ステップ2-3: templatesディレクトリのクリーンアップ

学習のため、既存のテンプレートを削除して一から作成します：

```bash
rm -rf templates/*
```

### ステップ2-4: シンプルなDeploymentテンプレート作成

`templates/deployment.yaml`を作成:

```bash
kubectl apply -f exercises/02-simple-chart/templates/deployment.yaml --dry-run=client -o yaml
```

演習ファイル `exercises/02-simple-chart/` を参照してください。

### ステップ2-5: Chartの検証

```bash
# 構文チェック
helm lint myfirstapp

# テンプレートのレンダリング確認（dry-run）
helm template myfirstapp ./myfirstapp

# インストール前のテスト
helm install --dry-run --debug myfirstapp ./myfirstapp
```

### ステップ2-6: Chartのインストール

```bash
# インストール
helm install myfirstapp ./myfirstapp

# 確認
helm list
kubectl get all
```

**✅ チェックポイント**:
- 独自のChartを作成できる
- テンプレートの基本構造を理解している
- helm lintとhelm templateを使える

---

## 演習3: テンプレート機能

**目標**: Helmのテンプレート機能を使いこなす

### ステップ3-1: Values.yamlの活用

`values.yaml`:
```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25-alpine"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

`templates/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
```

### ステップ3-2: ビルトインオブジェクトの理解

Helmには以下のビルトインオブジェクトがあります：

- `.Release`: リリース情報（Name, Namespace, IsInstall, IsUpgrade等）
- `.Chart`: Chart.yamlの内容
- `.Values`: values.yamlの内容
- `.Files`: Chartファイルへのアクセス
- `.Capabilities`: Kubernetesバージョン情報

### ステップ3-3: テンプレート関数の使用

```yaml
# 大文字変換
name: {{ .Release.Name | upper }}

# デフォルト値
replicas: {{ .Values.replicaCount | default 1 }}

# 引用符で囲む
tag: {{ .Values.image.tag | quote }}

# 条件分岐
{{- if .Values.service.enabled }}
apiVersion: v1
kind: Service
# ...
{{- end }}
```

### ステップ3-4: _helpers.tplの活用

`templates/_helpers.tpl`:
```yaml
{{- define "myfirstapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "myfirstapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
```

使用例:
```yaml
metadata:
  name: {{ include "myfirstapp.fullname" . }}
  labels:
    {{- include "myfirstapp.labels" . | nindent 4 }}
```

詳細は `exercises/03-templates/` を参照してください。

**✅ チェックポイント**:
- テンプレート変数を使用できる
- テンプレート関数を使用できる
- ヘルパーテンプレートを作成できる

---

## 演習4: Values管理

**目標**: 環境ごとの設定を管理する

### ステップ4-1: 環境別のValues作成

```bash
# ベースとなるvalues.yaml
values.yaml

# 開発環境用
values-dev.yaml

# ステージング環境用
values-staging.yaml

# 本番環境用
values-prod.yaml
```

### ステップ4-2: 環境別デプロイ

```bash
# 開発環境
helm install myapp ./myfirstapp -f values-dev.yaml

# ステージング環境
helm install myapp ./myfirstapp -f values-staging.yaml

# 本番環境
helm install myapp ./myfirstapp -f values-prod.yaml
```

### ステップ4-3: 値の優先順位

値は以下の優先順位で適用されます（下が優先）：

1. Chart内のvalues.yaml
2. 親Chartのvalues.yaml
3. `-f`で指定したファイル（複数指定可能）
4. `--set`で指定した値

```bash
# 複数のvaluesファイルを指定
helm install myapp ./myfirstapp \
  -f values.yaml \
  -f values-prod.yaml \
  --set replicaCount=5
```

詳細は `exercises/04-values/` を参照してください。

**✅ チェックポイント**:
- 環境別の設定を管理できる
- 値の優先順位を理解している

---

## 演習5: 依存関係の管理

**目標**: 他のChartに依存するChartを作成する

### ステップ5-1: Chart.yamlで依存関係を定義

`Chart.yaml`:
```yaml
apiVersion: v2
name: webapp
version: 1.0.0
dependencies:
  - name: mysql
    version: 9.x.x
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: 17.x.x
    repository: https://charts.bitnami.com/bitnami
```

### ステップ5-2: 依存Chartのダウンロード

```bash
# 依存関係を更新
helm dependency update ./webapp

# charts/ディレクトリを確認
ls -la webapp/charts/
```

### ステップ5-3: 依存Chartの設定

`values.yaml`:
```yaml
mysql:
  auth:
    rootPassword: mypassword
    database: mydb
  primary:
    persistence:
      enabled: true
      size: 8Gi

redis:
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

### ステップ5-4: インストール

```bash
helm install mywebapp ./webapp
```

詳細は `exercises/05-dependencies/` を参照してください。

**✅ チェックポイント**:
- Chart間の依存関係を定義できる
- 依存Chartの設定をカスタマイズできる

---

## 演習6: リポジトリ管理

**目標**: Helmリポジトリを管理する

### ステップ6-1: 公式リポジトリの利用

```bash
# 主要なリポジトリを追加
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add jetstack https://charts.jetstack.io

# リポジトリ一覧
helm repo list

# リポジトリ更新
helm repo update

# 特定のリポジトリを削除
helm repo remove stable
```

### ステップ6-2: Chart Museumでプライベートリポジトリ

```bash
# Chart Museumをインストール（Kindクラスター推奨）
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm install chartmuseum chartmuseum/chartmuseum --set env.open.DISABLE_API=false

# Chart Museumのサービスを確認
kubectl get svc chartmuseum

# ポートフォワーディング
kubectl port-forward svc/chartmuseum 8080:8080
```

### ステップ6-3: Chartをパッケージ化してアップロード

```bash
# Chartをパッケージ化
helm package ./myfirstapp

# Chart MuseumにアップロードChartMuseum APIを使用）
curl --data-binary "@myfirstapp-0.1.0.tgz" http://localhost:8080/api/charts

# リポジトリとして追加
helm repo add myrepo http://localhost:8080
helm repo update

# 自分のChartを検索
helm search repo myrepo
```

詳細は `exercises/06-repository/` を参照してください。

**✅ チェックポイント**:
- Helmリポジトリを管理できる
- Chartをパッケージ化できる
- プライベートリポジトリを運用できる

---

## 演習7: 本番運用パターン

**目標**: 本番環境でHelmを安全に運用する

### ステップ7-1: Helmのベストプラクティス

#### 1. バージョン管理
```bash
# Chartのバージョンを必ず指定
helm install myapp bitnami/nginx --version 13.2.10

# リリース名を明示的に指定
helm install my-nginx bitnami/nginx
```

#### 2. Namespaceの使用
```bash
# Namespaceを作成
kubectl create namespace production

# Namespace指定でインストール
helm install myapp ./myapp -n production

# Namespaceも自動作成
helm install myapp ./myapp -n production --create-namespace
```

#### 3. Dry-runとDiff
```bash
# インストール前の確認
helm install myapp ./myapp --dry-run --debug

# アップグレード前の差分確認（helm-diffプラグイン）
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myapp ./myapp
```

### ステップ7-2: Secrets管理

```bash
# Helm Secretsプラグイン
helm plugin install https://github.com/jkroepke/helm-secrets

# secrets.yamlを暗号化
helm secrets enc values-secrets.yaml

# 暗号化されたvaluesを使用
helm secrets install myapp ./myapp -f values-secrets.yaml
```

### ステップ7-3: テストの実装

`templates/tests/test-connection.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myfirstapp.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "myfirstapp.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

テストの実行:
```bash
helm test myapp
```

### ステップ7-4: ロールバック戦略

```bash
# アップグレード（失敗時は自動ロールバック）
helm upgrade myapp ./myapp --atomic --timeout 5m

# 手動ロールバック
helm rollback myapp 0  # 直前のバージョンに戻す

# 履歴の確認
helm history myapp

# 古い履歴を削除
helm history myapp --max 5
```

### ステップ7-5: モニタリング

```bash
# リリースの状態確認
helm status myapp

# すべてのリリースを確認
helm list --all-namespaces

# 失敗したリリースを確認
helm list --failed
```

詳細は `exercises/07-production/` を参照してください。

**✅ チェックポイント**:
- 本番環境向けのベストプラクティスを実践できる
- Secretsを安全に管理できる
- テストを実装・実行できる
- ロールバック戦略を理解している

---

## 理解度チェック

以下の質問にすべて答えられますか？

### 基礎
- [ ] Helmの3つの主要概念（Chart、Release、Repository）を説明できる
- [ ] Chart.yamlとvalues.yamlの違いを説明できる
- [ ] Chartの基本構造を説明できる

### 実践
- [ ] 既存のChartをインストール・アップグレード・削除できる
- [ ] 独自のChartを作成できる
- [ ] テンプレート関数を使いこなせる
- [ ] 環境別の設定を管理できる

### 応用
- [ ] Chart間の依存関係を定義できる
- [ ] プライベートリポジトリを構築できる
- [ ] 本番環境でのベストプラクティスを実践できる
- [ ] Secretsを安全に管理できる

## 📝 まとめ

この章では、Helmを徹底的に学習しました：

✅ Helmの基本概念と使い方
✅ Chartの作成とカスタマイズ
✅ テンプレート機能の活用
✅ 環境別の設定管理
✅ 依存関係の管理
✅ リポジトリの運用
✅ 本番運用のベストプラクティス

## 🚀 次のステップ

すべての演習を完了し、理解度チェックに合格したら、次の章に進みましょう：

**次の章**: [06-lens](../06-lens/README.md) - Lens IDEによるクラスター管理

**重要**: この章を完全に理解してから次に進んでください。

## 🔗 参考リンク

- [Helm公式ドキュメント](https://helm.sh/docs/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Artifact Hub](https://artifacthub.io/) - Chart検索
- [Helm GitHub](https://github.com/helm/helm)
