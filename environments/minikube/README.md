# Minikube環境セットアップガイド

## 📝 概要

Minikubeは、ローカル環境でKubernetesクラスターを実行するためのツールです。
学習用途に最適で、Kubernetesダッシュボードなど豊富な機能を提供します。

## 🎯 この環境で学べること

- Kubernetesの基本概念
- kubectl コマンドの使い方
- Pod、Service、Deploymentなどの基本リソース
- Kubernetes ダッシュボードの使用
- ローカル開発環境でのKubernetes操作

## 📦 インストール

### macOS
```bash
# Homebrewを使用
brew install minikube

# kubectlのインストール（まだの場合）
brew install kubectl
```

### Linux
```bash
# バイナリを直接ダウンロード
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# kubectlのインストール
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Windows
```powershell
# Chocolateyを使用
choco install minikube
choco install kubernetes-cli
```

## 🚀 クイックスタート

### 1. Minikubeクラスターの起動

```bash
# クラスターを起動（初回は時間がかかります）
minikube start

# ドライバーを指定する場合
minikube start --driver=docker

# リソースを指定する場合
minikube start --cpus=4 --memory=8192 --disk-size=20g
```

### 2. クラスターの状態確認

```bash
# Minikubeの状態確認
minikube status

# クラスター情報
kubectl cluster-info

# ノードの確認
kubectl get nodes

# 実行中のPodを確認
kubectl get pods -A
```

### 3. Kubernetesダッシュボードを開く

```bash
# ダッシュボードを起動（ブラウザが自動で開きます）
minikube dashboard

# バックグラウンドで起動する場合
minikube dashboard &
```

## 💻 基本的な演習

### 演習1: 最初のPodを作成

```bash
# サンプルPodを作成
kubectl apply -f exercises/01-hello-pod.yaml

# Podの状態を確認
kubectl get pods

# Podの詳細を確認
kubectl describe pod hello-minikube

# Podのログを確認
kubectl logs hello-minikube

# Podを削除
kubectl delete -f exercises/01-hello-pod.yaml
```

### 演習2: Deploymentとスケーリング

```bash
# Deploymentを作成
kubectl apply -f exercises/02-nginx-deployment.yaml

# Deploymentの状態確認
kubectl get deployments
kubectl get pods

# スケールアウト
kubectl scale deployment nginx-deployment --replicas=5

# スケールイン
kubectl scale deployment nginx-deployment --replicas=2

# クリーンアップ
kubectl delete -f exercises/02-nginx-deployment.yaml
```

### 演習3: Serviceとアクセス

```bash
# DeploymentとServiceを作成
kubectl apply -f exercises/03-web-app.yaml

# Serviceの確認
kubectl get services

# Minikubeのサービス機能でブラウザから確認
minikube service web-app-service

# サービスのURLを取得
minikube service web-app-service --url

# クリーンアップ
kubectl delete -f exercises/03-web-app.yaml
```

## 🔧 便利なコマンド

### クラスター管理

```bash
# クラスターを停止
minikube stop

# クラスターを削除
minikube delete

# クラスター情報
minikube status
minikube version

# アドオン一覧
minikube addons list

# アドオンを有効化（例：metrics-server）
minikube addons enable metrics-server

# SSH接続
minikube ssh
```

### デバッグとトラブルシューティング

```bash
# ログ確認
minikube logs

# IPアドレス取得
minikube ip

# Dockerデーモンに接続（ローカルのDockerコマンドでMinikube内のイメージを操作）
eval $(minikube docker-env)

# 元に戻す
eval $(minikube docker-env -u)
```

### リソース監視

```bash
# リソース使用状況（metrics-serverが必要）
kubectl top nodes
kubectl top pods

# すべてのリソースを確認
kubectl get all -A
```

## 🎓 ハンズオン演習フロー

以下の順番で進めることをおすすめします：

1. **`exercises/01-hello-pod.yaml`** - 単純なPodの作成
2. **`exercises/02-nginx-deployment.yaml`** - Deploymentとスケーリング
3. **`exercises/03-web-app.yaml`** - ServiceとLoadBalancer
4. **`exercises/04-configmap-demo.yaml`** - ConfigMapの使用
5. **`exercises/05-multi-tier-app.yaml`** - 複数コンポーネントのアプリケーション

## 🐛 トラブルシューティング

### クラスターが起動しない

```bash
# ログを確認
minikube logs

# 完全に削除して再作成
minikube delete --all --purge
minikube start
```

### Podが起動しない

```bash
# Pod詳細を確認
kubectl describe pod <pod-name>

# イベントを確認
kubectl get events --sort-by=.metadata.creationTimestamp

# ログを確認
kubectl logs <pod-name>
```

### Serviceにアクセスできない

```bash
# Serviceのエンドポイントを確認
kubectl get endpoints

# ポートフォワーディングで直接アクセス
kubectl port-forward service/<service-name> 8080:80
```

## 📚 次のステップ

Minikubeに慣れたら、より実践的な環境として**Kind**環境にも挑戦してみましょう！

- [Kind環境セットアップガイド](../kind/README.md)
- マルチノードクラスター
- より本番環境に近い構成

## 🔗 参考リンク

- [Minikube公式ドキュメント](https://minikube.sigs.k8s.io/docs/)
- [Kubernetes公式ドキュメント](https://kubernetes.io/ja/docs/home/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
