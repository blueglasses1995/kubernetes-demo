# 01. Kubernetes基本概念

この章では、Kubernetesの基本的な概念とリソースについて学習します。

## 🎯 学習目標
- Kubernetesクラスターの構造を理解する
- 基本的なリソース（Pod、Service、Deployment）を理解する
- 最初のPodを作成・操作する

## 📖 基本概念

### Kubernetesクラスター
Kubernetesクラスターは以下のコンポーネントで構成されます：

- **マスターノード**: クラスター全体を管理
  - API Server: Kubernetes APIのエンドポイント
  - etcd: クラスターの状態を保存するデータストア
  - Scheduler: Podをノードに割り当て
  - Controller Manager: 各種コントローラーを管理

- **ワーカーノード**: アプリケーションを実行
  - kubelet: ノード上のPodを管理
  - kube-proxy: ネットワークプロキシ
  - Container Runtime: コンテナの実行環境

### 基本リソース

#### Pod
- Kubernetesの最小デプロイ単位
- 1つ以上のコンテナをグループ化
- 同じPod内のコンテナは同じIPアドレスとストレージを共有

#### Service
- Podへのネットワークアクセスを提供
- Podの動的な変更に対応してロードバランシング

#### Deployment
- Podの宣言的な更新を管理
- ReplicaSetを通じてPodの複製を管理
- ローリングアップデートとロールバック機能

## 🛠️ 実践演習

### 1. クラスター状態の確認

まず、minikubeクラスターが正常に動作していることを確認しましょう：

```bash
# クラスター情報を表示
kubectl cluster-info

# ノード一覧を表示
kubectl get nodes

# ノードの詳細情報を表示
kubectl describe node minikube
```

### 2. 最初のPodを作成

シンプルなnginx Podを作成してみましょう。
