# Kind (Kubernetes in Docker) 環境セットアップガイド

## 📝 概要

Kind (Kubernetes in Docker) は、Dockerコンテナを使用してローカルでKubernetesクラスターを実行するツールです。
マルチノードクラスターの構築が容易で、高速かつ軽量なのが特徴です。

## 🎯 この環境で学べること

- マルチノードクラスターの構築と管理
- ノード間の通信とネットワーキング
- より本番環境に近い構成での検証
- クラスターのカスタマイズ（コントロールプレーン、ワーカーノード数）
- Ingress Controllerの使用
- CI/CDパイプラインでの使用方法

## 📦 インストール

### macOS
```bash
# Homebrewを使用
brew install kind

# kubectlのインストール（まだの場合）
brew install kubectl
```

### Linux
```bash
# AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Windows
```powershell
# Chocolateyを使用
choco install kind

# または直接ダウンロード
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\tools\kind.exe
```

## 🚀 クイックスタート

### 1. シンプルなクラスターの作成

```bash
# デフォルト設定でクラスターを作成（シングルノード）
kind create cluster

# クラスター名を指定
kind create cluster --name my-cluster

# クラスターの一覧
kind get clusters

# クラスターの削除
kind delete cluster --name my-cluster
```

### 2. マルチノードクラスターの作成

```bash
# 設定ファイルを使用してマルチノードクラスターを作成
kind create cluster --name multi-node --config=cluster-configs/multi-node-cluster.yaml

# クラスターの状態確認
kubectl cluster-info --context kind-multi-node
kubectl get nodes
```

### 3. クラスターへの接続確認

```bash
# コンテキストの確認
kubectl config get-contexts

# Kindクラスターに切り替え
kubectl config use-context kind-multi-node

# ノードの確認
kubectl get nodes -o wide

# システムPodの確認
kubectl get pods -A
```

## 🏗️ クラスター構成

このディレクトリには以下のクラスター設定ファイルが含まれています：

### 1. `cluster-configs/simple-cluster.yaml`
シンプルなシングルノードクラスター（学習用）

### 2. `cluster-configs/multi-node-cluster.yaml`
マルチノードクラスター（1コントロールプレーン + 3ワーカーノード）

### 3. `cluster-configs/ha-cluster.yaml`
HA構成クラスター（3コントロールプレーン + 3ワーカーノード）

### 4. `cluster-configs/ingress-cluster.yaml`
Ingress対応クラスター（ポートマッピング設定済み）

## 💻 実践的な演習

### 演習1: マルチノードクラスターでのPod配置

```bash
# マルチノードクラスターを作成
kind create cluster --name multi-node --config=cluster-configs/multi-node-cluster.yaml

# 演習用Deploymentを作成
kubectl apply -f exercises/01-node-affinity.yaml

# どのノードに配置されたか確認
kubectl get pods -o wide

# スケールアウトして複数ノードに分散させる
kubectl scale deployment node-demo --replicas=6
kubectl get pods -o wide
```

### 演習2: NodePortサービスの使用

```bash
# NodePortサービスをデプロイ
kubectl apply -f exercises/02-nodeport-service.yaml

# Serviceの確認
kubectl get services

# ノードのIPを取得
kubectl get nodes -o wide

# Dockerコンテナ経由でアクセス
docker exec -it multi-node-control-plane curl localhost:30080
```

### 演習3: Ingress Controllerのセットアップ

```bash
# Ingress対応クラスターを作成
kind create cluster --name ingress --config=cluster-configs/ingress-cluster.yaml

# Nginx Ingress Controllerをインストール
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Ingress Controllerの準備完了を待つ
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# サンプルアプリとIngressをデプロイ
kubectl apply -f exercises/03-ingress-demo.yaml

# localhostからアクセス可能
curl localhost/app1
curl localhost/app2
```

### 演習4: クラスター間の違いを体験

```bash
# 異なる構成のクラスターを作成
kind create cluster --name simple --config=cluster-configs/simple-cluster.yaml
kind create cluster --name ha --config=cluster-configs/ha-cluster.yaml

# コンテキストを切り替えながらノード構成を確認
kubectl config use-context kind-simple
kubectl get nodes

kubectl config use-context kind-ha
kubectl get nodes
```

## 🔧 便利なコマンド

### クラスター管理

```bash
# 全クラスター一覧
kind get clusters

# 特定のクラスターを削除
kind delete cluster --name <cluster-name>

# 全クラスターを削除
kind delete clusters --all

# クラスター情報をエクスポート
kind export kubeconfig --name <cluster-name>
```

### デバッグとトラブルシューティング

```bash
# クラスターのログを確認
kind export logs --name <cluster-name>

# ノードコンテナに直接アクセス
docker exec -it <cluster-name>-control-plane bash

# クラスター内のDockerイメージ一覧
docker exec -it <cluster-name>-control-plane crictl images

# ローカルのDockerイメージをクラスターにロード
kind load docker-image <image-name> --name <cluster-name>
```

### ローカルイメージの使用

```bash
# ローカルでイメージをビルド
docker build -t my-app:local .

# Kindクラスターにロード
kind load docker-image my-app:local --name <cluster-name>

# Deployment内で使用（imagePullPolicy: Never を設定）
kubectl apply -f my-deployment.yaml
```

## 🎓 ハンズオン演習フロー

以下の順番で進めることをおすすめします：

1. **シンプルクラスター** - 基本操作に慣れる
2. **マルチノードクラスター** - ノード間の違いを理解
3. **NodePortサービス** - サービス公開の方法を学ぶ
4. **Ingress** - より実践的なルーティングを実装
5. **HAクラスター** - 高可用性構成を体験

## 🔄 MinikubeとKindの使い分け

| 用途 | Minikube | Kind |
|------|----------|------|
| 学習開始時 | ⭐ おすすめ | - |
| ダッシュボード使用 | ⭐ あり | ❌ なし |
| マルチノード | △ 可能だが重い | ⭐ 高速・軽量 |
| クラスター作成速度 | 遅い（1-2分） | ⭐ 高速（10-30秒） |
| リソース消費 | 多い | ⭐ 少ない |
| CI/CD統合 | △ | ⭐ 最適 |
| 本番環境に近い構成 | △ | ⭐ |

## 🐛 トラブルシューティング

### クラスター作成に失敗する

```bash
# Dockerが起動しているか確認
docker ps

# 既存のクラスターを削除してから再作成
kind delete cluster --name <cluster-name>
kind create cluster --name <cluster-name>
```

### Podがイメージを取得できない

```bash
# ローカルイメージを使用する場合
kind load docker-image <image-name> --name <cluster-name>

# imagePullPolicyをNeverまたはIfNotPresentに設定
```

### ポートにアクセスできない

```bash
# クラスター作成時にポートマッピングを設定
# ingress-cluster.yamlを参照

# またはkubectl port-forwardを使用
kubectl port-forward service/<service-name> 8080:80
```

## 📚 次のステップ

- より複雑なマルチティアアプリケーションのデプロイ
- Helmを使用したアプリケーション管理
- StatefulSetとPersistentVolumeの使用
- NetworkPolicyによるセキュリティ設定

## 🔗 参考リンク

- [Kind公式ドキュメント](https://kind.sigs.k8s.io/)
- [Kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Kind Ingress Guide](https://kind.sigs.k8s.io/docs/user/ingress/)
- [Kubernetes公式ドキュメント](https://kubernetes.io/ja/docs/home/)
