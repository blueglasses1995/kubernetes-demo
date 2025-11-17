# 09 - Advanced Orchestration: Karpenter & Crossplane

## 📚 この章で学ぶこと

この章では、Kubernetesのインフラストラクチャ自動化ツールを**徹底的に**学習します。

**重要**: Istioの章（08-service-mesh-istio）を完全に理解してからこの章に進んでください。

## 🎯 学習目標

### Karpenter
- [ ] Karpenterの概念を理解する
- [ ] ノード自動スケーリングを実装できる
- [ ] コスト最適化ができる
- [ ] プロビジョナーを設定できる

### Crossplane
- [ ] Crossplaneの概念を理解する
- [ ] クラウドリソースをKubernetesで管理できる
- [ ] Compositionを作成できる
- [ ] Infrastructure as Codeを実践できる

## 📖 目次

### Part 1: Karpenter
1. [Karpenterとは](#karpenterとは)
2. [演習1: Karpenterのセットアップ](#演習1-karpenterのセットアップ)
3. [演習2: ノード自動プロビジョニング](#演習2-ノード自動プロビジョニング)
4. [演習3: コスト最適化](#演習3-コスト最適化)
5. [演習4: 高度な設定](#演習4-高度な設定)

### Part 2: Crossplane
6. [Crossplaneとは](#crossplaneとは)
7. [演習5: Crossplaneのセットアップ](#演習5-crossplaneのセットアップ)
8. [演習6: クラウドリソース管理](#演習6-クラウドリソース管理)
9. [演習7: Composition作成](#演習7-composition作成)
10. [理解度チェック](#理解度チェック)

---

# Part 1: Karpenter

## Karpenterとは

### 従来のクラスターオートスケーラーの課題

Kubernetes Cluster Autoscaler（CA）の問題点：
- ノードグループごとの設定が必要
- スケールアップが遅い
- リソースの無駄が多い
- 複雑な設定

### Karpenterの利点

**Karpenterは次世代のノードオートスケーラー**：

1. **高速**: Podを直接監視し、即座にノードをプロビジョニング
2. **柔軟**: ワークロードに最適なインスタンスタイプを自動選択
3. **コスト効率**: Spot/オンデマンドの混在、最適なサイジング
4. **シンプル**: ノードグループ不要

### アーキテクチャ

```
Unschedulable Pods
    ↓
Karpenter Controller
    ↓
クラウドプロバイダーAPI（AWS/Azure/GCP）
    ↓
新しいノードがプロビジョニング
```

---

## 演習1: Karpenterのセットアップ

**目標**: AWS EKSクラスターにKarpenterをインストールする

**注意**: Karpenterは実際のクラウド環境（AWS推奨）が必要です。

### ステップ1-1: 前提条件

```bash
# AWS CLI設定済み
aws configure

# eksctl インストール済み
eksctl version

# Helm インストール済み
helm version
```

### ステップ1-2: EKSクラスターの作成

```bash
# Karpenter用のEKSクラスターを作成
export CLUSTER_NAME=karpenter-demo
export AWS_REGION=ap-northeast-1

eksctl create cluster -f karpenter/cluster.yaml
```

`karpenter/cluster.yaml`:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: karpenter-demo
  region: ap-northeast-1
  version: "1.28"

managedNodeGroups:
  - name: karpenter-nodes
    instanceType: t3.medium
    minSize: 2
    maxSize: 3
    desiredCapacity: 2

iam:
  withOIDC: true
```

### ステップ1-3: IAMロールとポリシー

```bash
# Karpenter用のIAMロールを作成
eksctl create iamserviceaccount \
  --cluster="${CLUSTER_NAME}" \
  --name=karpenter \
  --namespace=karpenter \
  --role-name="${CLUSTER_NAME}-karpenter" \
  --attach-policy-arn="arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy" \
  --attach-policy-arn="arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy" \
  --attach-policy-arn="arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly" \
  --attach-policy-arn="arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore" \
  --approve
```

### ステップ1-4: Karpenterのインストール

```bash
# HelmリポジトリにKarpenter追加
helm repo add karpenter https://charts.karpenter.sh
helm repo update

# Karpenterをインストール
helm upgrade --install karpenter karpenter/karpenter \
  --namespace karpenter --create-namespace \
  --set serviceAccount.create=false \
  --set serviceAccount.name=karpenter \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --wait
```

### ステップ1-5: インストール確認

```bash
kubectl get pods -n karpenter
kubectl logs -f -n karpenter deployment/karpenter
```

**✅ チェックポイント**:
- EKSクラスターを作成できる
- Karpenterをインストールできる

---

## 演習2: ノード自動プロビジョニング

**目標**: Karpenterでノードを自動的にプロビジョニングする

### ステップ2-1: Provisionerの作成

```yaml
# manifests/provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64"]
    - key: karpenter.k8s.aws/instance-category
      operator: In
      values: ["c", "m", "r"]
    - key: karpenter.k8s.aws/instance-generation
      operator: Gt
      values: ["2"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000
```

### ステップ2-2: AWSNodeTemplateの作成

```yaml
# manifests/aws-node-template.yaml
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}
```

適用：
```bash
kubectl apply -f manifests/provisioner.yaml
kubectl apply -f manifests/aws-node-template.yaml
```

### ステップ2-3: テストワークロードのデプロイ

```bash
# スケジュールできないPodを作成
kubectl apply -f manifests/test-deployment.yaml
```

`manifests/test-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      containers:
      - name: inflate
        image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
        resources:
          requests:
            cpu: 1
            memory: 1.5Gi
```

### ステップ2-4: スケールアップのテスト

```bash
# レプリカを増やす
kubectl scale deployment inflate --replicas=10

# Karpenterがノードを自動プロビジョニング
kubectl get nodes -w

# Karpenterのログを確認
kubectl logs -f -n karpenter deployment/karpenter
```

新しいノードが自動的に作成されます！

### ステップ2-5: スケールダウンのテスト

```bash
# レプリカを減らす
kubectl scale deployment inflate --replicas=0

# 30秒後（ttlSecondsAfterEmpty）にノードが削除される
kubectl get nodes -w
```

**✅ チェックポイント**:
- Provisionerを作成できる
- ワークロードに応じてノードが自動作成される
- 不要なノードが自動削除される

---

## 演習3: コスト最適化

**目標**: Karpenterでコストを最適化する

### ステップ3-1: Spotインスタンスの活用

```yaml
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]  # Spotのみ使用
```

### ステップ3-2: インスタンスタイプの制限

```yaml
spec:
  requirements:
    # 小〜中型インスタンスのみ
    - key: karpenter.k8s.aws/instance-size
      operator: In
      values: ["small", "medium", "large"]
```

### ステップ3-3: コンソリデーション

不要なノードを積極的に削除：

```yaml
spec:
  consolidation:
    enabled: true
```

### ステップ3-4: コスト監視

```bash
# ノードのコストを確認（AWS Cost Explorer使用）
# KarpenterはタグでノードをマークするのでCost Explorerで分析可能
```

**✅ チェックポイント**:
- Spotインスタンスを活用できる
- インスタンスタイプを最適化できる
- コンソリデーションを設定できる

---

# Part 2: Crossplane

## Crossplaneとは

### Infrastructure as Codeの進化

従来のIaC（Terraform、CloudFormation）の課題：
- Kubernetesとは別の管理が必要
- 状態管理が複雑
- GitOps統合が困難

### Crossplaneの利点

**Kubernetesでクラウドリソースをネイティブ管理**：

1. **統一されたAPI**: Kubernetesリソースとしてインフラを管理
2. **GitOps対応**: ArgoCD、Fluxと統合
3. **マルチクラウド**: AWS、Azure、GCP、100+ プロバイダー
4. **抽象化**: Compositionで独自APIを作成

---

## 演習5: Crossplaneのセットアップ

**目標**: CrossplaneをインストールしてAWSプロバイダーを設定する

### ステップ5-1: Crossplaneのインストール

```bash
# Helmリポジトリに追加
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Crossplaneをインストール
helm install crossplane \
  --namespace crossplane-system \
  --create-namespace \
  crossplane-stable/crossplane
```

### ステップ5-2: AWSプロバイダーのインストール

```yaml
# crossplane/aws-provider.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v0.47.0
```

```bash
kubectl apply -f crossplane/aws-provider.yaml

# プロバイダーが ready になるまで待つ
kubectl get providers
```

### ステップ5-3: AWS認証情報の設定

```bash
# AWS認証情報をSecretとして作成
kubectl create secret generic aws-creds \
  -n crossplane-system \
  --from-file=creds=./aws-credentials.txt
```

`aws-credentials.txt`:
```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
```

### ステップ5-4: ProviderConfigの作成

```yaml
# crossplane/provider-config.yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

```bash
kubectl apply -f crossplane/provider-config.yaml
```

**✅ チェックポイント**:
- Crossplaneをインストールできる
- AWSプロバイダーを設定できる

---

## 演習6: クラウドリソース管理

**目標**: CrossplaneでS3バケットを作成・管理する

### ステップ6-1: S3バケットの作成

```yaml
# manifests/s3-bucket.yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-crossplane-bucket
spec:
  forProvider:
    region: ap-northeast-1
    tags:
      Environment: dev
      ManagedBy: crossplane
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f manifests/s3-bucket.yaml

# リソースが作成されるのを確認
kubectl get buckets
kubectl describe bucket my-crossplane-bucket
```

### ステップ6-2: バケットポリシーの追加

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketPublicAccessBlock
metadata:
  name: my-bucket-pab
spec:
  forProvider:
    bucket: my-crossplane-bucket
    blockPublicAcls: true
    blockPublicPolicy: true
    ignorePublicAcls: true
    restrictPublicBuckets: true
  providerConfigRef:
    name: default
```

### ステップ6-3: リソースの削除

```bash
# Kubernetesリソースを削除すると、AWSのS3バケットも削除される
kubectl delete bucket my-crossplane-bucket
```

**✅ チェックポイント**:
- Crossplaneでクラウドリソースを作成できる
- リソースのライフサイクルを管理できる

---

## 演習7: Composition作成

**目標**: Compositionで抽象化されたAPIを作成する

### ステップ7-1: CompositeResourceDefinitionの作成

独自の `Database` APIを定義：

```yaml
# crossplane/xrd.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.example.com
spec:
  group: example.com
  names:
    kind: XDatabase
    plural: xdatabases
  claimNames:
    kind: Database
    plural: databases
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
                enum: [small, medium, large]
            required:
            - size
```

### ステップ7-2: Compositionの作成

```yaml
# crossplane/composition.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: database-dynamodb
spec:
  compositeTypeRef:
    apiVersion: example.com/v1alpha1
    kind: XDatabase
  resources:
  - name: dynamodb-table
    base:
      apiVersion: dynamodb.aws.upbound.io/v1beta1
      kind: Table
      spec:
        forProvider:
          region: ap-northeast-1
          attribute:
          - name: id
            type: S
          hashKey: id
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.size
      toFieldPath: spec.forProvider.billingMode
      transforms:
      - type: map
        map:
          small: PAY_PER_REQUEST
          medium: PROVISIONED
          large: PROVISIONED
```

### ステップ7-3: Databaseの作成

```yaml
# manifests/my-database.yaml
apiVersion: example.com/v1alpha1
kind: Database
metadata:
  name: my-app-db
spec:
  size: small
```

```bash
kubectl apply -f manifests/my-database.yaml

# DynamoDBテーブルが自動作成される
kubectl get databases
kubectl get tables
```

**✅ チェックポイント**:
- CompositeResourceDefinitionを作成できる
- Compositionで複数リソースを組み合わせられる
- 抽象化されたAPIを使える

---

## 理解度チェック

### Karpenter
- [ ] Karpenterの利点を説明できる
- [ ] Provisionerを作成できる
- [ ] コスト最適化ができる

### Crossplane
- [ ] Crossplaneの概念を理解している
- [ ] クラウドリソースをKubernetesで管理できる
- [ ] Compositionを作成できる

## 📝 まとめ

✅ Karpenterによるノード自動プロビジョニング
✅ コスト最適化戦略
✅ Crossplaneによるインフラ管理
✅ Compositionによる抽象化

## 🚀 次のステップ

**次の章**: [10-platform-abstraction](../10-platform-abstraction/README.md) - RenderとFly.io

**重要**: この章を完全に理解してから次に進んでください。

## 🔗 参考リンク

- [Karpenter](https://karpenter.sh/)
- [Crossplane](https://www.crossplane.io/)
- [Crossplane AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-aws)
