# 06 - Lens: Kubernetes IDE

## 📚 この章で学ぶこと

Lens（旧Lens Desktop）は、Kubernetesクラスターを管理するための強力なIDEです。この章では、Lensを**徹底的に**使いこなせるようになります。

**重要**: Helmの章（05-helm）を完全に理解してからこの章に進んでください。

## 🎯 学習目標

- [ ] Lensのインストールと初期設定ができる
- [ ] クラスターの追加と切り替えができる
- [ ] リソースの可視化と監視ができる
- [ ] リソースの作成・編集・削除ができる
- [ ] ログとメトリクスの確認ができる
- [ ] ターミナル機能を使いこなせる
- [ ] トラブルシューティングができる
- [ ] 拡張機能を活用できる

## 📖 目次

1. [Lensとは](#1-lensとは)
2. [Lensのインストール](#2-lensのインストール)
3. [演習1: クラスター接続](#演習1-クラスター接続)
4. [演習2: リソース可視化](#演習2-リソース可視化)
5. [演習3: リソース操作](#演習3-リソース操作)
6. [演習4: ログとメトリクス](#演習4-ログとメトリクス)
7. [演習5: ターミナルとコマンド実行](#演習5-ターミナルとコマンド実行)
8. [演習6: トラブルシューティング](#演習6-トラブルシューティング)
9. [演習7: 拡張機能](#演習7-拡張機能)
10. [理解度チェック](#理解度チェック)

## 1. Lensとは

### なぜLensが必要か

kubectlコマンドラインだけでは：
- リソースの全体像が把握しづらい
- 複数のコマンドを覚える必要がある
- ログやメトリクスの確認が煩雑
- 複数クラスターの管理が困難

**LensはこれらをGUIで解決します**

### Lensの主な機能

1. **マルチクラスター管理**: 複数のクラスターを一元管理
2. **リソース可視化**: すべてのリソースを視覚的に表示
3. **リアルタイムモニタリング**: CPU、メモリ使用率をグラフ表示
4. **統合ターミナル**: IDE内でkubectlやshellを実行
5. **Prometheusメトリクス**: 組み込みメトリクス収集
6. **拡張機能**: プラグインでカスタマイズ可能

## 2. Lensのインストール

### ダウンロード

公式サイトからダウンロード:
https://k8slens.dev/

### macOS
```bash
# Homebrewでインストール
brew install --cask lens

# または、DMGファイルをダウンロードしてインストール
```

### Linux
```bash
# AppImage版をダウンロード
wget https://api.k8slens.dev/binaries/Lens-<version>.AppImage
chmod +x Lens-<version>.AppImage
./Lens-<version>.AppImage

# または、Snapでインストール
sudo snap install kontena-lens --classic
```

### Windows
```powershell
# Chocolateyでインストール
choco install lens

# または、.exeインストーラーをダウンロード
```

### 初回起動

1. Lensを起動
2. 利用規約に同意
3. オプションでアカウント作成（スキップ可能）

---

## 演習1: クラスター接続

**目標**: Lensにクラスターを追加して接続する

### ステップ1-1: kubeconfigの確認

Lensはkubeconfigを使用してクラスターに接続します：

```bash
# kubeconfigの場所を確認
echo $KUBECONFIG
# デフォルト: ~/.kube/config

# 現在のコンテキストを確認
kubectl config get-contexts
```

### ステップ1-2: クラスターの追加

1. Lensを起動
2. "Catalog" タブをクリック
3. "+" ボタンまたは "Add Cluster" をクリック
4. 以下のいずれかの方法で追加：
   - **From kubeconfig**: ~/.kube/configを選択
   - **Paste as text**: kubeconfigの内容を貼り付け

### ステップ1-3: クラスターへの接続

1. Catalog画面でクラスターをクリック
2. クラスターに接続されると、ダッシュボードが表示される

### ステップ1-4: 複数クラスターの管理

Minikube と Kind の両方を追加してみましょう：

```bash
# Minikubeクラスターを起動
minikube start

# Kindクラスターを作成
kind create cluster --name lens-demo
```

Lensで両方のクラスターを追加し、簡単に切り替えられることを確認してください。

**✅ チェックポイント**:
- Lensにクラスターを追加できる
- 複数のクラスターを管理できる
- クラスター間を切り替えられる

---

## 演習2: リソース可視化

**目標**: Lensでリソースを可視化して理解する

### ステップ2-1: ダッシュボードの理解

クラスターに接続すると、以下が表示されます：

1. **Cluster**: クラスター全体の概要
2. **Nodes**: ノード一覧とリソース使用状況
3. **Workloads**: Pod、Deployment、StatefulSet等
4. **Config**: ConfigMap、Secret等
5. **Network**: Service、Ingress等
6. **Storage**: PersistentVolume、StorageClass等

### ステップ2-2: ノードの可視化

1. 左メニューから "Nodes" を選択
2. ノード一覧を確認
3. ノードをクリックして詳細を表示：
   - CPU・メモリ使用率（グラフ）
   - Podの一覧
   - ラベルとアノテーション
   - イベント

### ステップ2-3: Workloadsの可視化

1. "Workloads" > "Pods" を選択
2. すべてのPodを一覧表示
3. Podの状態を色で確認：
   - 緑: Running
   - 黄: Pending
   - 赤: Error/CrashLoopBackOff

### ステップ2-4: テスト用リソースのデプロイ

CLIまたはLensから以下をデプロイ：

```bash
kubectl create deployment nginx --image=nginx:1.25-alpine --replicas=3
kubectl expose deployment nginx --port=80 --type=NodePort
```

Lensで以下を確認：
- Deployment の状態
- ReplicaSet の自動作成
- 3つのPodが異なるノードに配置されている様子
- Serviceの作成

**✅ チェックポイント**:
- ダッシュボードを使いこなせる
- リソースを視覚的に把握できる
- リソース間の関係を理解できる

---

## 演習3: リソース操作

**目標**: Lensを使ってリソースを作成・編集・削除する

### ステップ3-1: リソースの作成

1. "Workloads" > "Deployments" を選択
2. "+" ボタンをクリック
3. YAMLエディタが開く
4. 以下のYAMLを入力：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-lens
  labels:
    app: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: nginx
        image: nginx:1.25-alpine
        ports:
        - containerPort: 80
```

5. "Create & Close" をクリック

### ステップ3-2: リソースの編集

1. 作成した "hello-lens" Deploymentをクリック
2. 右上の "Edit" ボタン（鉛筆アイコン）をクリック
3. `replicas: 2` を `replicas: 5` に変更
4. "Save & Close" をクリック
5. Podsが5つにスケールアップされることを確認

### ステップ3-3: リソースのスケーリング（GUI）

1. Deploymentを選択
2. 右上の "..." メニューをクリック
3. "Scale" を選択
4. レプリカ数を入力してスケール

### ステップ3-4: リソースの削除

1. Deploymentを選択
2. 右上の "..." メニューをクリック
3. "Delete" を選択
4. 確認ダイアログで "Remove" をクリック

**✅ チェックポイント**:
- Lensでリソースを作成できる
- YAMLエディタを使いこなせる
- リソースをスケーリングできる
- リソースを削除できる

---

## 演習4: ログとメトリクス

**目標**: ログとメトリクスを確認してアプリケーションを監視する

### ステップ4-1: Podログの確認

1. テスト用Podをデプロイ：
```bash
kubectl run busybox --image=busybox --restart=Never -- sh -c 'for i in $(seq 1 100); do echo "Log line $i"; sleep 1; done'
```

2. Lensで "Workloads" > "Pods" から "busybox" を選択
3. 下部の "Logs" タブを表示
4. リアルタイムでログが流れることを確認
5. 検索ボックスでログをフィルタリング

### ステップ4-2: マルチコンテナPodのログ

1. 複数コンテナを持つPodをデプロイ：

```yaml
# exercises/multi-container-pod.yaml を参照
```

2. Lensで Pod を選択
3. Logs タブでコンテナを切り替え
4. 各コンテナのログを確認

### ステップ4-3: メトリクスの確認

1. "Cluster" をクリック
2. クラスター全体のメトリクスを確認：
   - CPU使用率
   - メモリ使用率
   - Pod数

3. 個別のPodを選択
4. CPU・メモリのグラフを確認

### ステップ4-4: Metricsサーバーの有効化

```bash
# Minikubeの場合
minikube addons enable metrics-server

# Kindの場合
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

メトリクスが表示されるようになります。

**✅ チェックポイント**:
- Podのログを確認できる
- マルチコンテナのログを切り替えられる
- メトリクスを確認できる

---

## 演習5: ターミナルとコマンド実行

**目標**: Lens内でターミナルとkubectlを使う

### ステップ5-1: 統合ターミナルの使用

1. 下部の "Terminal" タブをクリック
2. kubectlコマンドを実行：

```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

### ステップ5-2: Pod内でShellを実行

1. 実行中のPodを選択
2. 右上の "..." メニューから "Pod Shell" を選択
3. Pod内でコマンドを実行：

```bash
# nginxコンテナの場合
ls -la /usr/share/nginx/html/
cat /etc/nginx/nginx.conf
```

### ステップ5-3: Port Forwarding

1. Serviceまたは Podを選択
2. "..." メニューから "Forward" を選択
3. ローカルポートと対象ポートを指定
4. ブラウザで `http://localhost:<port>` にアクセス

**✅ チェックポイント**:
- Lens内でkubectlコマンドを実行できる
- Pod内でShellを実行できる
- Port Forwardingができる

---

## 演習6: トラブルシューティング

**目標**: Lensを使って問題を特定・解決する

### ステップ6-1: 失敗しているPodの調査

1. 意図的に失敗するPodをデプロイ：

```bash
kubectl run failing-pod --image=nginx:wrong-tag
```

2. Lensで "Pods" を確認
3. 赤色で表示されているPodをクリック
4. "Events" タブで問題を確認：
   - ImagePullBackOff
   - ErrImagePull

### ステップ6-2: イベントの確認

1. 任意のリソースを選択
2. "Events" タブを表示
3. 時系列でイベントを確認

### ステップ6-3: リソース使用率の問題

1. リソース制限を超えるPodをデプロイ：

```yaml
# exercises/resource-exceeded-pod.yaml を参照
```

2. Lensでメトリクスを確認
3. OOMKilled等の問題を特定

### ステップ6-4: ネットワーク問題の調査

1. Serviceが正しく機能しているか確認
2. Endpointsを確認
3. Selectorが正しいか確認

**✅ チェックポイント**:
- 失敗しているリソースを特定できる
- イベントから問題の原因を特定できる
- リソース使用率の問題を発見できる

---

## 演習7: 拡張機能

**目標**: Lens拡張機能を活用する

### ステップ7-1: 拡張機能の概要

Lensは拡張機能で機能を追加できます：

- **@alebcay/openlens-node-pod-menu**: ノードメニューの拡張
- **Pod Security Insights**: セキュリティスキャン
- **Resource Map**: リソース間の依存関係可視化

### ステップ7-2: 拡張機能のインストール

1. "File" > "Extensions" (または Cmd/Ctrl + Shift + E)
2. "Browse" タブで拡張機能を検索
3. "Install" をクリック

### ステップ7-3: よく使う拡張機能

1. **@alebcay/openlens-node-pod-menu**
   - ノードのコンテキストメニューを拡張
   - ノード上のPod一覧を素早く表示

2. **@nevalla/kube-resource-map**
   - リソース間の関係を可視化
   - サービスメッシュの理解に有用

インストールして使ってみましょう。

**✅ チェックポイント**:
- 拡張機能をインストールできる
- 拡張機能を活用できる

---

## 理解度チェック

以下の質問にすべて答えられますか？

### 基礎
- [ ] Lensの主な機能を説明できる
- [ ] クラスターを追加できる
- [ ] ダッシュボードを理解している

### 実践
- [ ] リソースを視覚的に把握できる
- [ ] リソースの作成・編集・削除ができる
- [ ] ログとメトリクスを確認できる
- [ ] 統合ターミナルを使える

### 応用
- [ ] トラブルシューティングができる
- [ ] Port Forwardingができる
- [ ] 拡張機能を活用できる

## 📝 まとめ

この章では、Lensを徹底的に学習しました：

✅ Lensのインストールと初期設定
✅ マルチクラスター管理
✅ リソースの可視化と操作
✅ ログとメトリクスの確認
✅ 統合ターミナルの活用
✅ トラブルシューティング
✅ 拡張機能の活用

## 🎓 Lensの利点まとめ

**kubectlと比較した利点**:
- 視覚的な理解が容易
- 複数クラスターの一元管理
- リアルタイムメトリクス
- YAMLエディタの利便性
- ログの検索とフィルタリング

**実務での活用**:
- 開発環境でのデバッグ
- 本番環境の監視
- 新しいメンバーへのオンボーディング
- インシデント対応

## 🚀 次のステップ

すべての演習を完了し、理解度チェックに合格したら、次の章に進みましょう：

**次の章**: [07-service-mesh-linkerd](../07-service-mesh-linkerd/README.md) - Linkerdサービスメッシュ

**重要**: この章を完全に理解してから次に進んでください。次の章からはより高度なトピックに入ります。

## 🔗 参考リンク

- [Lens公式サイト](https://k8slens.dev/)
- [Lens GitHub](https://github.com/lensapp/lens)
- [Lens Extensions](https://github.com/lensapp/lens-extensions)
- [Lens Documentation](https://docs.k8slens.dev/)
