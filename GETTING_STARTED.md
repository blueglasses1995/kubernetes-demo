# 🎯 Kubernetes学習ガイド

## 環境構築完了！✅

おめでとうございます！Kubernetes学習環境の構築が完了しました。

### 構築された環境
- ✅ Docker Desktop
- ✅ minikube (Kubernetesクラスター)
- ✅ kubectl (Kubernetesコマンドラインツール)
- ✅ 学習用プロジェクト構造

## 🚀 学習を始めましょう

### ステップ1: 基本概念の理解
まず最初に基本的なコマンドを実行してみましょう：

```bash
# 現在のディレクトリに移動
cd /Users/toshikimatsukuma/Documents/study/kubernetes-demo

# クラスター状態確認
kubectl cluster-info
kubectl get nodes

# 最初のPodを作成
kubectl apply -f 01-basic-concepts/01-simple-pod.yaml

# Podの状態確認
kubectl get pods
kubectl describe pod my-first-pod

# Podのログを確認
kubectl logs my-first-pod
```

### ステップ2: 実際に試してみる

1. **最初のPodを作成**
   ```bash
   kubectl apply -f 01-basic-concepts/01-simple-pod.yaml
   kubectl get pods -w
   ```

2. **Pod内でコマンド実行**
   ```bash
   kubectl exec -it my-first-pod -- /bin/bash
   ```

3. **Podを削除**
   ```bash
   kubectl delete -f 01-basic-concepts/01-simple-pod.yaml
   ```

### ステップ3: Deploymentを試す

```bash
# Deploymentを作成
kubectl apply -f 02-deployments/01-nginx-deployment.yaml

# 作成されたリソースを確認
kubectl get deployments
kubectl get replicasets
kubectl get pods

# スケーリング
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods
```

### ステップ4: Serviceを試す

```bash
# Serviceを作成
kubectl apply -f 03-services/01-nginx-service.yaml

# Serviceを確認
kubectl get services

# minikubeでServiceにアクセス
minikube service nginx-nodeport
```

## 📚 次のステップ

各ディレクトリのREADME.mdファイルを順番に読んで、実際にコマンドを実行してみてください：

1. `01-basic-concepts/README.md` - 基本概念
2. `02-deployments/README.md` - Deployments
3. `03-services/README.md` - Services
4. `04-config-secrets/README.md` - ConfigMaps & Secrets

## 🔧 便利なコマンド

```bash
# 全リソース確認
kubectl get all

# 特定のnamespaceのリソース確認
kubectl get all -n kube-system

# YAML形式で出力
kubectl get pod my-first-pod -o yaml

# ポートフォワーディング
kubectl port-forward pod/my-first-pod 8080:80

# Kubernetesダッシュボード起動
minikube dashboard
```

## 🆘 トラブルシューティング

### クラスターが起動しない場合
```bash
# minikubeを削除して再作成
minikube delete
minikube start

# ログを確認
minikube logs
```

### Podが起動しない場合
```bash
# Pod詳細を確認
kubectl describe pod <pod-name>

# イベントを確認
kubectl get events --sort-by=.metadata.creationTimestamp

# ログを確認
kubectl logs <pod-name>
```

頑張って学習してください！🚀
