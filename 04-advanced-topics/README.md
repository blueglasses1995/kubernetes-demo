# 高度なKubernetesトピック

## 📚 概要

このディレクトリには、Kubernetesの高度な機能と実践的なシナリオが含まれています。
基本的な概念を理解した後に挑戦してください。

## 🎯 学習内容

1. **Rolling UpdateとRollback** - アプリケーションの無停止更新
2. **HorizontalPodAutoscaler (HPA)** - 自動スケーリング
3. **ResourceQuotaとLimitRange** - リソース管理
4. **NetworkPolicy** - ネットワークセキュリティ
5. **PersistentVolumeとStatefulSet** - ステートフルアプリケーション

## 📁 演習ファイル

### 01-rolling-update.yaml
アプリケーションのローリングアップデートとロールバックを学習

```bash
# デプロイ
kubectl apply -f 01-rolling-update.yaml

# 更新状態の確認
kubectl rollout status deployment/rolling-demo

# 履歴確認
kubectl rollout history deployment/rolling-demo

# ロールバック
kubectl rollout undo deployment/rolling-demo
```

### 02-hpa-demo.yaml
負荷に応じた自動スケーリングを体験

```bash
# metrics-serverが必要（Minikubeの場合）
minikube addons enable metrics-server

# デプロイ
kubectl apply -f 02-hpa-demo.yaml

# 負荷をかける
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

# HPAの状態を監視
kubectl get hpa -w
```

### 03-resource-quota.yaml
Namespaceごとのリソース制限を設定

```bash
# 専用Namespaceを作成
kubectl create namespace limited-resources

# ResourceQuotaを適用
kubectl apply -f 03-resource-quota.yaml

# Quotaの確認
kubectl describe resourcequota -n limited-resources
```

### 04-network-policy.yaml
Pod間の通信を制御

```bash
# NetworkPolicyをデプロイ
kubectl apply -f 04-network-policy.yaml

# 通信テスト
kubectl exec -it frontend -- curl backend-service
kubectl exec -it unauthorized -- curl backend-service  # これは失敗する
```

### 05-persistent-volume.yaml
永続的なストレージの使用

```bash
# PersistentVolumeとPVCを作成
kubectl apply -f 05-persistent-volume.yaml

# データの永続性をテスト
kubectl exec -it mysql-pod -- mysql -u root -ppassword -e "CREATE DATABASE testdb;"
kubectl delete pod mysql-pod
kubectl apply -f 05-persistent-volume.yaml
kubectl exec -it mysql-pod -- mysql -u root -ppassword -e "SHOW DATABASES;"
```

## 🎓 推奨学習順序

1. **基礎の復習** (1-3章を完了していること)
2. **Rolling Update** → デプロイメント戦略の理解
3. **HPA** → スケーリングの自動化
4. **ResourceQuota** → リソース管理の重要性
5. **NetworkPolicy** → セキュリティの実装
6. **PersistentVolume** → ステートフルアプリケーション

## 💡 実践シナリオ

### シナリオ1: 本番環境へのデプロイ
1. ローリングアップデートでアプリケーションを更新
2. 問題が発生したらロールバック
3. HPAで負荷に応じて自動スケール

### シナリオ2: マルチテナント環境
1. Namespaceで環境を分離
2. ResourceQuotaで各環境のリソースを制限
3. NetworkPolicyでセキュリティを確保

### シナリオ3: データベースのデプロイ
1. PersistentVolumeでデータを永続化
2. StatefulSetで安定したネットワークIDを保証
3. Secretでパスワードを管理

## 🔧 環境別の注意点

### Minikube
```bash
# メトリクスサーバーを有効化
minikube addons enable metrics-server

# ストレージクラスの確認
kubectl get storageclass
```

### Kind
```bash
# メトリクスサーバーをインストール
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# metrics-serverの設定を調整（Kind用）
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

## 🐛 トラブルシューティング

### HPAがメトリクスを取得できない
```bash
# metrics-serverが動作しているか確認
kubectl get deployment metrics-server -n kube-system

# Podのメトリクスが取得できるか確認
kubectl top pods
```

### PersistentVolumeが作成されない
```bash
# StorageClassを確認
kubectl get storageclass

# PVとPVCの状態を確認
kubectl get pv,pvc
kubectl describe pvc <pvc-name>
```

### NetworkPolicyが動作しない
```bash
# CNIプラグインがNetworkPolicyをサポートしているか確認
# Minikubeのデフォルトは対応していないため、Calicoなどを使用

# Minikubeの場合
minikube start --cni=calico
```

## 📊 チェックリスト

学習が完了したら以下をチェック：

- [ ] ローリングアップデートを実行し、ロールバックできる
- [ ] HPAを設定し、負荷に応じてPodが増減することを確認
- [ ] ResourceQuotaを設定し、リソース制限が機能することを確認
- [ ] NetworkPolicyを設定し、Pod間通信を制御できる
- [ ] PersistentVolumeを使用し、データが永続化されることを確認

## 🔗 参考リンク

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
