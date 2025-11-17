# Kubernetes学習プロジェクト

## 📋 学習内容

このプロジェクトは、Kubernetesの基礎から最先端の技術までを段階的に学習するためのものです。
**MinikubeとKind**の2つの環境を使い分けることで、効率的に学習を進められます。

## 🗺️ 完全マスターカリキュラム

**📘 [CURRICULUM.md](./CURRICULUM.md) - 完全マスターカリキュラム（全147時間）を確認する**

基礎から最先端まで、一歩ずつ学習する体系的なカリキュラムを用意しています：

- **Phase 1**: 基礎編（1-2ヶ月）- Kubernetes基本からHPA、PersistentVolumeまで
- **Phase 2**: エコシステムツール編（2-3ヶ月）- Helm、Lens
- **Phase 3**: サービスメッシュ編（2-3ヶ月）- Linkerd、Istio Ambient
- **Phase 4**: 高度なオーケストレーション編（2-3ヶ月）- Karpenter、Crossplane
- **Phase 5**: プラットフォーム抽象化編（1-2ヶ月）- Render、Fly.io

## 🏗️ プロジェクト構造

```
kubernetes-demo/
├── CURRICULUM.md          # 📘 完全マスターカリキュラム（必読）
├── environments/          # 環境別セットアップガイド
│   ├── minikube/         # Minikube環境（基本学習用）
│   └── kind/             # Kind環境（実践練習用）
│
├── 01-basic-concepts/     # Kubernetes基本概念
├── 02-deployments/        # Deployments、ReplicaSets
├── 03-services/           # Services、Ingress
├── 04-advanced-topics/    # 高度なトピック
│
├── 05-helm/               # Helmパッケージマネージャー
├── 06-lens/               # Lens Kubernetes IDE
├── 07-service-mesh-linkerd/  # Linkerdサービスメッシュ
├── 08-service-mesh-istio/    # Istio Ambientサービスメッシュ
├── 09-advanced-orchestration/  # Karpenter & Crossplane
├── 10-platform-abstraction/    # Render & Fly.io
│
└── README.md             # このファイル
```

## 🚀 環境構築

### 🎯 環境の選択

このプロジェクトでは2つの学習環境を用意しています：

#### 1️⃣ Minikube（基本学習用）- 初心者におすすめ
- ✅ Kubernetesダッシュボード付き
- ✅ 視覚的に理解しやすい
- ✅ 豊富なアドオン機能
- 📖 [Minikube環境セットアップ](./environments/minikube/README.md)

```bash
# クイックスタート
minikube start
minikube dashboard
```

#### 2️⃣ Kind（実践練習用）- 経験者・実践向け
- ⚡ 高速起動（10-30秒）
- 🏗️ マルチノードクラスター対応
- 🚀 本番環境に近い構成
- 📖 [Kind環境セットアップ](./environments/kind/README.md)

```bash
# クイックスタート
kind create cluster --name practice --config=environments/kind/cluster-configs/multi-node-cluster.yaml
```

### 📚 環境の使い分けガイド

詳しい使い分け方法は [environments/README.md](./environments/README.md) をご覧ください。

| 用途 | Minikube | Kind |
|------|----------|------|
| 学習開始時 | ⭐ おすすめ | - |
| マルチノード | △ | ⭐ 最適 |
| 起動速度 | 遅い | ⭐ 高速 |
| ダッシュボード | ✅ | ❌ |

## 📚 学習の進め方

### フェーズ1: 基礎固め（Minikube環境）

まずはMinikubeで基本を学習：

1. **環境構築** - [Minikubeセットアップ](./environments/minikube/README.md)
2. **基本概念** - `01-basic-concepts/` でPod、Service等を学習
3. **デプロイメント** - `02-deployments/` でDeploymentを理解
4. **ネットワーキング** - `03-services/` でServiceとIngressを学習

### フェーズ2: 実践力アップ（Kind環境）

Kindでより実践的な環境を経験：

1. **環境構築** - [Kindセットアップ](./environments/kind/README.md)
2. **マルチノード** - 複数ノードでのPod配置を学習
3. **Ingress** - Ingress Controllerのセットアップ
4. **高度なトピック** - `04-advanced-topics/` で実践的な機能を習得

### フェーズ3: 高度な機能の習得

より深く学ぶために：

1. **Rolling Update** - 無停止でのアプリケーション更新
2. **Autoscaling** - HPA（水平Pod自動スケーリング）
3. **リソース管理** - ResourceQuotaとLimitRange
4. **セキュリティ** - NetworkPolicyによるアクセス制御
5. **永続化** - PersistentVolumeとStatefulSet

詳しくは [04-advanced-topics/README.md](./04-advanced-topics/README.md) をご覧ください。

## 🔧 便利なコマンド

### 基本的なkubectlコマンド
```bash
# リソース一覧表示
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get all  # すべてのリソース

# リソース詳細表示
kubectl describe pod <pod-name>
kubectl describe service <service-name>

# ログ確認
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # リアルタイム表示

# Pod内でコマンド実行
kubectl exec -it <pod-name> -- /bin/bash

# コンテキスト切り替え
kubectl config get-contexts
kubectl config use-context minikube
kubectl config use-context kind-practice
```

### Minikubeコマンド
```bash
# クラスター起動・停止
minikube start
minikube stop
minikube delete

# ダッシュボード
minikube dashboard

# サービスにアクセス
minikube service <service-name>

# アドオン管理
minikube addons list
minikube addons enable metrics-server
```

### Kindコマンド
```bash
# クラスター作成
kind create cluster --name <cluster-name>
kind create cluster --config=<config-file>

# クラスター一覧・削除
kind get clusters
kind delete cluster --name <cluster-name>

# ローカルイメージをロード
kind load docker-image <image-name> --name <cluster-name>

# ログ確認
kind export logs --name <cluster-name>
```

## 🎯 学習目標

このプロジェクトを通じて以下を習得します：

### 基礎レベル
- [ ] Kubernetesの基本概念（Pod、Service、Deploymentなど）
- [ ] YAMLマニフェストファイルの書き方
- [ ] kubectlコマンドの使い方
- [ ] MinikubeとKindの使い分け

### 中級レベル
- [ ] アプリケーションのデプロイとスケーリング
- [ ] サービス間通信の設定
- [ ] 設定情報とシークレットの管理
- [ ] マルチノードクラスターでの運用

### 上級レベル
- [ ] Rolling UpdateとRollback
- [ ] HorizontalPodAutoscaler（自動スケーリング）
- [ ] ResourceQuotaとLimitRange
- [ ] NetworkPolicyによるセキュリティ管理
- [ ] PersistentVolumeとStatefulSet
- [ ] トラブルシューティングの実践

## 🚦 はじめ方

1. まず [environments/README.md](./environments/README.md) で環境を選択
2. 選んだ環境のセットアップガイドに従ってクラスターを起動
3. `01-basic-concepts/` から順番に学習を開始

## 🔗 参考リンク

- [Kubernetes公式ドキュメント](https://kubernetes.io/ja/docs/home/)
- [Minikube公式ドキュメント](https://minikube.sigs.k8s.io/docs/)
- [Kind公式ドキュメント](https://kind.sigs.k8s.io/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

Happy Learning! 🚀
