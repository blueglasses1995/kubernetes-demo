# Kubernetes学習プロジェクト

## 📋 学習内容

このプロジェクトは、Kubernetesの基礎から実践的な使用方法までを段階的に学習するためのものです。

## 🏗️ プロジェクト構造

```
kubernetes-demo/
├── 01-basic-concepts/     # Kubernetes基本概念
├── 02-deployments/        # Deployments、ReplicaSets
├── 03-services/           # Services、Ingress
├── 04-config-secrets/     # ConfigMaps、Secrets
└── README.md             # このファイル
```

## 🚀 環境構築

### 前提条件
- Docker Desktop
- minikube
- kubectl

### セットアップ手順

1. **minikubeクラスターの開始**
   ```bash
   minikube start
   ```

2. **クラスターの状態確認**
   ```bash
   kubectl cluster-info
   kubectl get nodes
   ```

3. **Kubernetesダッシュボードの起動（オプション）**
   ```bash
   minikube dashboard
   ```

## 📚 学習の進め方

各ディレクトリの順番に従って学習を進めてください：

1. `01-basic-concepts` - Kubernetesの基本概念を学習
2. `02-deployments` - アプリケーションのデプロイメント
3. `03-services` - サービスとネットワーキング
4. `04-config-secrets` - 設定管理とシークレット

## 🔧 便利なコマンド

### 基本的なkubectlコマンド
```bash
# リソース一覧表示
kubectl get pods
kubectl get services
kubectl get deployments

# リソース詳細表示
kubectl describe pod <pod-name>
kubectl describe service <service-name>

# ログ確認
kubectl logs <pod-name>

# Pod内でコマンド実行
kubectl exec -it <pod-name> -- /bin/bash
```

### minikubeコマンド
```bash
# クラスター状態確認
minikube status

# クラスター停止
minikube stop

# クラスター削除
minikube delete

# Kubernetesサービスをブラウザで開く
minikube service <service-name>
```

## 🎯 学習目標

このプロジェクトを通じて以下を習得します：

- [ ] Kubernetesの基本概念（Pod、Service、Deploymentなど）
- [ ] YAMLマニフェストファイルの書き方
- [ ] kubectlコマンドの使い方
- [ ] アプリケーションのデプロイとスケーリング
- [ ] サービス間通信の設定
- [ ] 設定情報とシークレットの管理
- [ ] トラブルシューティングの基本

次のステップ: `01-basic-concepts/README.md` を確認してください。
