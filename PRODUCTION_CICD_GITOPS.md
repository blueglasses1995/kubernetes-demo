# Kubernetes本番環境 CI/CD & GitOps 完全ガイド

このドキュメントは、Kubernetes本番環境でのCI/CD及びGitOpsの実装を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [GitOps 原理](#1-gitops-原理)
2. [ArgoCD](#2-argocd)
3. [Flux CD](#3-flux-cd)
4. [Progressive Delivery](#4-progressive-delivery)
5. [Image Build パイプライン](#5-image-build-パイプライン)
6. [Testing in Kubernetes](#6-testing-in-kubernetes)

---

## 1. GitOps 原理

### 1.1 GitOps とは

#### 従来のデプロイ vs GitOps

```
┌─────────────────────────────────────────────────────────┐
│ 従来のPush型デプロイ                                    │
└─────────────────────────────────────────────────────────┘

Developer → CI Pipeline → kubectl apply → Kubernetes
             (Jenkins等)    (Push)

問題点:
- kubectl の認証情報をCI/CDツールに保存
- クラスター外部からの書き込みアクセス
- デプロイ状態とGitの乖離
- ロールバックが困難


┌─────────────────────────────────────────────────────────┐
│ GitOps（Pull型デプロイ）                                │
└─────────────────────────────────────────────────────────┘

Developer → Git Repository ← GitOps Operator (in cluster)
            (manifests)            ↓
                                Kubernetes

GitOps Operator が:
1. Gitリポジトリを監視
2. 変更を検知
3. クラスターに自動適用
4. 実際の状態とGitを継続的に同期

利点:
- クラスターへの外部アクセス不要
- Gitが唯一の信頼できる情報源（Single Source of Truth）
- 宣言的な定義
- 自動同期とドリフト検知
- Git履歴による監査とロールバック
```

#### GitOps の4原則

```
1. 宣言的（Declarative）
   システムの望ましい状態をYAML等で宣言的に記述

2. バージョン管理（Versioned）
   すべての設定をGitでバージョン管理

3. 自動プル（Pulled Automatically）
   GitOps Operatorがリポジトリから変更を自動プル

4. 継続的調整（Continuously Reconciled）
   実際の状態と宣言された状態の差分を継続的に調整
```

### 1.2 リポジトリ構成戦略

#### Mono-repo vs Multi-repo

```yaml
# パターン1: Mono-repo（単一リポジトリ）
k8s-manifests/
├── apps/
│   ├── frontend/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   ├── overlays/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── prod/
│   ├── backend/
│   └── database/
├── infrastructure/
│   ├── ingress-nginx/
│   ├── cert-manager/
│   └── monitoring/
└── clusters/
    ├── dev-cluster.yaml
    ├── staging-cluster.yaml
    └── prod-cluster.yaml

利点:
- 一元管理
- 変更の可視性が高い
- クロスアプリケーション変更が容易

欠点:
- アクセス制御の粒度が粗い
- リポジトリが巨大化

---
# パターン2: Multi-repo（複数リポジトリ）
# アプリケーションリポジトリ
frontend-app/
├── src/
├── Dockerfile
└── k8s/
    ├── deployment.yaml
    └── service.yaml

backend-app/
├── src/
├── Dockerfile
└── k8s/

# インフラストラクチャリポジトリ
k8s-infrastructure/
├── ingress-nginx/
├── cert-manager/
└── monitoring/

# 環境別マニフェストリポジトリ
k8s-manifests-prod/
k8s-manifests-staging/
k8s-manifests-dev/

利点:
- チームごとのアクセス制御が容易
- リポジトリサイズが小さい
- 独立したデプロイサイクル

欠点:
- 管理が複雑
- クロスリポジトリ変更が困難
```

#### App of Apps パターン

```yaml
# root-app.yaml - すべてのアプリケーションを管理するルートApp
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: apps
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# apps/frontend-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: apps/frontend/overlays/prod
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# apps/backend-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    path: apps/backend/overlays/prod
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 2. ArgoCD

### 2.1 ArgoCD アーキテクチャ

```
┌──────────────────────────────────────────────────────────┐
│ ArgoCD Architecture                                      │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ argocd-server (API & UI)                            │  │
│ │ ・Web UI                                            │  │
│ │ ・gRPC/REST API                                     │  │
│ │ ・認証/認可                                         │  │
│ └─────────────────────────────────────────────────────┘  │
│                          │                               │
│                          ▼                               │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ argocd-repo-server                                  │  │
│ │ ・Gitリポジトリとの通信                             │  │
│ │ ・マニフェストの生成（Helm, Kustomize）             │  │
│ │ ・マニフェストのキャッシュ                          │  │
│ └─────────────────────────────────────────────────────┘  │
│                          │                               │
│                          ▼                               │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ argocd-application-controller                       │  │
│ │ ・Application CRの監視                              │  │
│ │ ・Git vs クラスターの差分検知                       │  │
│ │ ・同期処理の実行                                    │  │
│ │ ・ヘルスチェック                                    │  │
│ └─────────────────────────────────────────────────────┘  │
│            │                                             │
│            ▼                                             │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ Kubernetes API Server                               │  │
│ │ リソースの作成/更新/削除                            │  │
│ └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘

同期プロセス:
1. Application Controller が定期的にGitをポーリング（デフォルト3分）
2. repo-server がマニフェストを生成
3. 生成されたマニフェストとクラスターの実際の状態を比較
4. 差分があれば sync を実行（自動or手動）
5. 各リソースのヘルスステータスを監視
```

### 2.2 ArgoCD インストール

```bash
# Namespace作成
kubectl create namespace argocd

# ArgoCD インストール（HA構成）
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# インストール確認
kubectl get pods -n argocd

# 出力:
# NAME                                  READY   STATUS    RESTARTS   AGE
# argocd-application-controller-0       1/1     Running   0          2m
# argocd-applicationset-controller-...  1/1     Running   0          2m
# argocd-dex-server-...                 1/1     Running   0          2m
# argocd-notifications-controller-...   1/1     Running   0          2m
# argocd-redis-...                      1/1     Running   0          2m
# argocd-repo-server-...                1/1     Running   0          2m
# argocd-server-...                     1/1     Running   0          2m

# 初期パスワード取得
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# ポートフォワードでUIにアクセス
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ブラウザで https://localhost:8080
# ユーザー名: admin
# パスワード: <上記で取得したパスワード>

# ArgoCD CLI インストール
brew install argocd  # macOS
# または
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# CLI ログイン
argocd login localhost:8080 --username admin --password <password> --insecure

# パスワード変更
argocd account update-password
```

### 2.3 ArgoCD Application の作成

#### CLI での作成

```bash
# 基本的なApplication作成
argocd app create myapp \
  --repo https://github.com/myorg/k8s-manifests \
  --path apps/myapp/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod

# Helm Chart の場合
argocd app create myapp \
  --repo https://github.com/myorg/helm-charts \
  --path charts/myapp \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --helm-set image.tag=v1.2.3 \
  --helm-set replicas=3

# 自動同期の有効化
argocd app set myapp --sync-policy automated

# Prune（不要リソースの削除）を有効化
argocd app set myapp --auto-prune

# Self Heal（手動変更の自動修正）を有効化
argocd app set myapp --self-heal

# Application の確認
argocd app list

# 詳細表示
argocd app get myapp

# 同期実行
argocd app sync myapp

# ロールバック
argocd app rollback myapp <revision>
```

#### YAML での作成

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  # Finalizer により、Applicationが削除されたときにリソースも削除
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  # Project（RBAC用）
  project: production

  # Gitリポジトリの設定
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: main
    path: apps/myapp/overlays/prod

    # Kustomize 設定
    kustomize:
      namePrefix: prod-
      nameSuffix: -v1
      images:
      - myregistry.io/myapp:v1.2.3

    # Helm 設定（Helmの場合）
    # helm:
    #   releaseName: myapp
    #   valueFiles:
    #   - values-prod.yaml
    #   parameters:
    #   - name: image.tag
    #     value: v1.2.3
    #   - name: replicas
    #     value: "3"

  # デプロイ先
  destination:
    server: https://kubernetes.default.svc
    namespace: prod

  # 同期ポリシー
  syncPolicy:
    # 自動同期
    automated:
      prune: true      # 不要なリソースを削除
      selfHeal: true   # ドリフト検知時に自動修正
      allowEmpty: false

    # Sync Options
    syncOptions:
    - CreateNamespace=true     # Namespaceが存在しない場合は作成
    - PruneLast=true           # 削除は最後に実行
    - RespectIgnoreDifferences=true

    # リトライ設定
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # 差分を無視する設定
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # HPAが管理するため無視

  # ヘルスチェックのカスタマイズ
  # info:
  # - name: url
  #   value: https://myapp.example.com
```

### 2.4 AppProject（RBAC）

```yaml
# appproject.yaml - 本番環境用のProject
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  # 説明
  description: Production applications

  # このProjectで許可するGitリポジトリ
  sourceRepos:
  - https://github.com/myorg/k8s-manifests
  - https://github.com/myorg/helm-charts

  # デプロイ可能なクラスターとNamespace
  destinations:
  - namespace: prod
    server: https://kubernetes.default.svc
  - namespace: monitoring
    server: https://kubernetes.default.svc

  # 許可するリソース種別
  # デフォルトはすべて拒否、明示的に許可
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRole
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRoleBinding

  # Namespace内で許可するリソース
  namespaceResourceWhitelist:
  - group: ''
    kind: Service
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Secret
  - group: 'apps'
    kind: Deployment
  - group: 'apps'
    kind: StatefulSet
  - group: 'networking.k8s.io'
    kind: Ingress

  # 拒否するリソース（ホワイトリストより優先）
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  - group: ''
    kind: LimitRange

  # Orphan resources（管理対象外のリソース）の扱い
  orphanedResources:
    warn: true  # 警告を表示

  # Role（誰がこのProjectにアクセスできるか）
  roles:
  # 開発者: 読み取りのみ
  - name: developer
    description: Developers (read-only)
    policies:
    - p, proj:production:developer, applications, get, production/*, allow
    - p, proj:production:developer, applications, sync, production/*, deny
    groups:
    - developers

  # SRE: フルアクセス
  - name: sre
    description: SRE team (full access)
    policies:
    - p, proj:production:sre, applications, *, production/*, allow
    groups:
    - sre-team

  # CI/CD: 同期のみ
  - name: cicd
    description: CI/CD automation
    policies:
    - p, proj:production:cicd, applications, sync, production/*, allow
    - p, proj:production:cicd, applications, get, production/*, allow
```

### 2.5 Sync Waves と Hooks

#### Sync Waves（同期順序制御）

```yaml
# 1. Namespace を最初に作成（wave 0）
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# 2. ConfigMap/Secret を作成（wave 1）
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
  annotations:
    argocd.argoproj.io/sync-wave: "1"
data:
  config.yaml: |
    server:
      port: 8080

---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: prod
  annotations:
    argocd.argoproj.io/sync-wave: "1"
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=

---
# 3. Service を作成（wave 2）
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  selector:
    app: myapp
  ports:
  - port: 8080

---
# 4. Deployment を作成（wave 3）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: prod
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:1.0
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret

---
# 5. Ingress を最後に作成（wave 4）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: prod
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 8080
```

#### Resource Hooks

```yaml
# PreSync Hook - 同期前に実行（DBマイグレーション等）
---
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: prod
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:1.2.3
        command: ["./migrate.sh"]
        env:
        - name: DB_HOST
          value: postgres.prod.svc
      restartPolicy: Never
  backoffLimit: 3

---
# PostSync Hook - 同期後に実行（スモークテスト等）
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  namespace: prod
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: test
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          sleep 10
          curl -f http://myapp.prod.svc:8080/health || exit 1
      restartPolicy: Never

---
# SyncFail Hook - 同期失敗時に実行（通知等）
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-failure
  namespace: prod
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: notify
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          curl -X POST https://hooks.slack.com/services/XXX/YYY/ZZZ \
            -H 'Content-Type: application/json' \
            -d '{"text":"ArgoCD sync failed for myapp"}'
      restartPolicy: Never

# Hook Delete Policy:
# - HookSucceeded: Hook成功時に削除
# - HookFailed: Hook失敗時に削除
# - BeforeHookCreation: 次のHook実行前に削除
```

### 2.6 Notifications

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack設定
  service.slack: |
    token: $slack-token

  # テンプレート定義
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} is now running new version.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          {{range $index, $c := .app.status.conditions}}
          ,{
            "title": "{{$c.type}}",
            "value": "{{$c.message}}",
            "short": true
          }
          {{end}}
          ]
        }]

  template.app-sync-failed: |
    message: |
      Failed to sync application {{.app.metadata.name}}.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#E96D76",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          }
          {{range $index, $c := .app.status.conditions}}
          ,{
            "title": "{{$c.type}}",
            "value": "{{$c.message}}",
            "short": true
          }
          {{end}}
          ]
        }]

  # トリガー定義
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'

  trigger.on-sync-failed: |
    - description: Application sync failed
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']

---
# Slack Token Secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  slack-token: xoxb-your-slack-bot-token

---
# Application に通知設定を追加
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: production-deploys
    notifications.argoproj.io/subscribe.on-sync-failed.slack: production-alerts
spec:
  # ... (省略)
```

---

## 3. Flux CD

### 3.1 Flux アーキテクチャ

```
┌──────────────────────────────────────────────────────────┐
│ Flux CD Architecture                                     │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ source-controller                                   │  │
│ │ ・Gitリポジトリの監視                               │  │
│ │ ・Helmリポジトリの監視                              │  │
│ │ ・OCI registryの監視                                │  │
│ │ ・ソースのダウンロードとキャッシュ                  │  │
│ └─────────────────────────────────────────────────────┘  │
│                          │                               │
│                          ▼                               │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ kustomize-controller                                │  │
│ │ ・Kustomizationの監視                               │  │
│ │ ・マニフェストの適用                                │  │
│ │ ・健全性チェック                                    │  │
│ │ ・Prune（削除）処理                                 │  │
│ └─────────────────────────────────────────────────────┘  │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ helm-controller                                     │  │
│ │ ・HelmReleaseの監視                                 │  │
│ │ ・Helm chart のインストール/アップグレード          │  │
│ │ ・ロールバック                                      │  │
│ └─────────────────────────────────────────────────────┘  │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ notification-controller                             │  │
│ │ ・イベントの受信（Webhook）                         │  │
│ │ ・通知の送信（Slack等）                             │  │
│ └─────────────────────────────────────────────────────┘  │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ image-reflector-controller                          │  │
│ │ ・コンテナレジストリの監視                          │  │
│ │ ・新しいイメージタグの検出                          │  │
│ └─────────────────────────────────────────────────────┘  │
│                          │                               │
│                          ▼                               │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ image-automation-controller                         │  │
│ │ ・イメージタグの自動更新                            │  │
│ │ ・Gitへのコミット & プッシュ                        │  │
│ └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Flux インストール

```bash
# Flux CLI のインストール
brew install fluxcd/tap/flux  # macOS
# または
curl -s https://fluxcd.io/install.sh | sudo bash

# 前提条件チェック
flux check --pre

# GitHubトークンの設定（repo権限が必要）
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

# Flux のブートストラップ（GitHubリポジトリ作成 & Fluxインストール）
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal

# インストール確認
flux check

# 出力:
# ► checking prerequisites
# ✔ Kubernetes 1.28.0 >=1.26.0-0
# ► checking controllers
# ✔ source-controller: deployment ready
# ✔ kustomize-controller: deployment ready
# ✔ helm-controller: deployment ready
# ✔ notification-controller: deployment ready
# ✔ image-reflector-controller: deployment ready
# ✔ image-automation-controller: deployment ready
# ✔ all checks passed

# Flux コンポーネント確認
kubectl get pods -n flux-system
```

### 3.3 GitRepository と Kustomization

```yaml
# gitrepository.yaml - Gitリポジトリの定義
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: k8s-manifests
  namespace: flux-system
spec:
  interval: 1m  # 1分ごとにポーリング
  url: https://github.com/myorg/k8s-manifests
  ref:
    branch: main

  # Git 認証（プライベートリポジトリの場合）
  secretRef:
    name: git-credentials

  # 特定のパスのみ監視
  include:
  - repository: https://github.com/myorg/k8s-manifests
    fromPath: apps/prod

---
# kustomization.yaml - マニフェストの適用
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m  # 5分ごとに調整
  path: ./apps/myapp/overlays/prod
  prune: true   # 不要なリソースを削除
  timeout: 2m

  # ソース指定
  sourceRef:
    kind: GitRepository
    name: k8s-manifests

  # ヘルスチェック
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    namespace: prod

  # 依存関係（他のKustomizationが完了後に実行）
  dependsOn:
  - name: infrastructure

  # リトライ設定
  retryInterval: 1m

  # Post-build変数置換
  postBuild:
    substitute:
      CLUSTER_NAME: "production"
      REGION: "ap-northeast-1"
    substituteFrom:
    - kind: ConfigMap
      name: cluster-config
```

### 3.4 Image Update Automation

```yaml
# 1. ImageRepository - コンテナレジストリの監視
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: myregistry.io/myapp
  interval: 1m

  # プライベートレジストリの認証
  secretRef:
    name: registry-credentials

---
# 2. ImagePolicy - どのイメージタグを使用するか
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp-prod
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp

  # セマンティックバージョニング
  policy:
    semver:
      range: 1.x.x  # 1.x.x のみ自動更新

  # または正規表現
  # policy:
  #   alphabetical:
  #     order: asc
  # policy:
  #   numerical:
  #     order: asc

---
# 3. ImageUpdateAutomation - Gitへの自動コミット
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: myapp-automation
  namespace: flux-system
spec:
  interval: 1m

  # Gitリポジトリ
  sourceRef:
    kind: GitRepository
    name: k8s-manifests

  # Gitへの書き込み設定
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: fluxbot
        email: flux@example.com
      messageTemplate: |
        Automated image update

        Automation name: {{ .AutomationObject }}

        Files:
        {{ range $filename, $_ := .Updated.Files -}}
        - {{ $filename }}
        {{ end -}}

        Objects:
        {{ range $resource, $_ := .Updated.Objects -}}
        - {{ $resource.Kind }} {{ $resource.Name }}
        {{ end -}}

        Images:
        {{ range .Updated.Images -}}
        - {{.}}
        {{ end -}}
    push:
      branch: main

  # 更新対象
  update:
    path: ./apps/myapp/overlays/prod
    strategy: Setters  # kustomize image transformer を使用

---
# 4. アプリケーションマニフェスト（イメージタグをマーカー）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myregistry.io/myapp:1.2.3 # {"$imagepolicy": "flux-system:myapp-prod"}
        #                                   ↑ このマーカーでFluxが自動更新
```

---

## 4. Progressive Delivery

### 4.1 Canary Deployment (Flagger)

#### Flagger のインストール

```bash
# Flagger インストール（Istio用）
kubectl apply -k github.com/fluxcd/flagger//kustomize/istio

# または Linkerd用
kubectl apply -k github.com/fluxcd/flagger//kustomize/linkerd

# Prometheus Operator用のServiceMonitor
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/kustomize/prometheus/flagger-servicemonitor.yaml
```

#### Canary リソース

```yaml
# canary.yaml - Canaryデプロイメント定義
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: prod
spec:
  # ターゲット
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  # Progressive Delivery設定
  progressDeadlineSeconds: 60  # タイムアウト

  # HPA設定（オプション）
  autoscalerRef:
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    name: myapp

  # Service設定
  service:
    port: 8080
    targetPort: 8080
    # Istio Gateway
    gateways:
    - istio-system/public-gateway
    hosts:
    - myapp.example.com

  # カナリア分析
  analysis:
    # チェック間隔
    interval: 1m

    # 成功と判断する閾値
    threshold: 5  # 5回連続成功

    # 最大トラフィック割合
    maxWeight: 50

    # トラフィック増加ステップ
    stepWeight: 10  # 10%ずつ増加

    # メトリクス
    metrics:
    # 成功率（Request Success Rate）
    - name: request-success-rate
      thresholdRange:
        min: 99  # 99%以上の成功率
      interval: 1m

    # レイテンシ（Request Duration）
    - name: request-duration
      thresholdRange:
        max: 500  # 500ms以下
      interval: 1m

    # カスタムメトリクス（Prometheus）
    - name: error-rate
      templateRef:
        name: error-rate
        namespace: flagger-system
      thresholdRange:
        max: 1  # 1%以下のエラー率
      interval: 1m

    # Webhook（カスタムチェック）
    webhooks:
    # Load test
    - name: load-test
      url: http://flagger-loadtester.prod/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary.prod:8080/"

    # Smoke test
    - name: smoke-test
      url: http://flagger-loadtester.prod/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "curl -f http://myapp-canary.prod:8080/health"

  # ロールバック設定
  skipAnalysis: false
```

#### Canary デプロイメントのフロー

```
Canaryデプロイメントの進行:

1. 初期状態（100% stable）
   stable: ████████████████████ 100%
   canary: 0%

2. Canary開始（10% canary）
   stable: ██████████████████ 90%
   canary: ██ 10%
   → メトリクスチェック（1分間）

3. メトリクスOK → 20% canary
   stable: ████████████████ 80%
   canary: ████ 20%
   → メトリクスチェック（1分間）

4. メトリクスOK → 30% canary
   stable: ██████████████ 70%
   canary: ██████ 30%

... (10%ずつ増加)

10. メトリクスOK → 100% canary
    stable: 0%
    canary: ████████████████████ 100%

11. Promotion（昇格）
    stable: ████████████████████ 100% (canaryバージョン)
    canary: 0%

失敗時:
X. メトリクス NG → ロールバック
   stable: ████████████████████ 100% (元のバージョン)
   canary: 0% (削除)
```

#### MetricTemplate（カスタムメトリクス）

```yaml
# metric-template.yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: error-rate
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring.svc:9090

  query: |
    100 - (
      sum(
        rate(
          http_requests_total{
            namespace="{{ namespace }}",
            deployment=~"{{ target }}",
            status!~"5.."
          }[{{ interval }}]
        )
      )
      /
      sum(
        rate(
          http_requests_total{
            namespace="{{ namespace }}",
            deployment=~"{{ target }}"
          }[{{ interval }}]
        )
      )
    ) * 100
```

### 4.2 Blue-Green Deployment

```yaml
# blue-green-canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  service:
    port: 8080

  # Blue-Green モード
  analysis:
    interval: 1m
    threshold: 10
    iterations: 10  # 10回チェック（10分間）

    # トラフィック割合は変更しない（0% or 100%）
    stepWeight: 100
    maxWeight: 100

    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m

    # Pre-rollout webhook（本番トラフィック投入前のテスト）
    webhooks:
    - name: integration-test
      url: http://flagger-loadtester.prod/
      timeout: 5m
      metadata:
        type: cmd
        cmd: "./integration-test.sh http://myapp-canary.prod:8080"

# Blue-Greenフロー:
# 1. Green（新バージョン）をデプロイ
# 2. Integration testを実行
# 3. テスト成功 → 全トラフィックをGreenに切り替え
# 4. 10分間監視
# 5. 問題なければBlue（旧バージョン）を削除
```

---

## 5. Image Build パイプライン

### 5.1 GitHub Actions でのイメージビルド

```yaml
# .github/workflows/build.yaml
name: Build and Push Docker Image

on:
  push:
    branches:
    - main
    tags:
    - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
    # 1. コードチェックアウト
    - name: Checkout
      uses: actions/checkout@v4

    # 2. Docker Buildx セットアップ
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    # 3. レジストリログイン
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    # 4. メタデータ生成（タグ、ラベル）
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=semver,pattern={{major}}
          type=sha,prefix={{branch}}-

    # 5. イメージビルド & プッシュ
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

        # ビルド引数
        build-args: |
          VERSION=${{ github.ref_name }}
          COMMIT_SHA=${{ github.sha }}
          BUILD_DATE=${{ github.event.head_commit.timestamp }}

    # 6. イメージスキャン（Trivy）
    - name: Run Trivy scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'

    # 7. スキャン結果をGitHub Securityにアップロード
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

    # 8. Slackに通知
    - name: Slack notification
      if: always()
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: |
          Image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          Build: ${{ job.status }}
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### 5.2 マルチステージビルドの最適化

```dockerfile
# Dockerfile - マルチステージビルド
# Stage 1: ビルド
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 依存関係のダウンロード（キャッシュ活用）
COPY go.mod go.sum ./
RUN go mod download

# ソースコードのコピーとビルド
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a \
    -ldflags="-s -w -X main.version=${VERSION} -X main.commit=${COMMIT_SHA}" \
    -o myapp .

# Stage 2: 最小ランタイム
FROM gcr.io/distroless/static-debian12:nonroot

# メタデータ
LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"
LABEL org.opencontainers.image.description="My Application"
LABEL org.opencontainers.image.licenses="MIT"

# ビルド成果物のみコピー
COPY --from=builder /app/myapp /myapp

# 非rootユーザーで実行
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/myapp"]
```

### 5.3 Buildpack を使ったビルド

```bash
# Cloud Native Buildpacks（ソースコードから自動でイメージ作成）

# pack CLI のインストール
brew install buildpacks/tap/pack

# イメージビルド（Dockerfileなし）
pack build myapp:latest \
  --builder paketobuildpacks/builder:base \
  --env BP_JVM_VERSION=17 \
  --env BP_MAVEN_BUILD_ARGUMENTS="-Dmaven.test.skip=true"

# または Kubernetes 上で Kpack を使用
# https://github.com/pivotal/kpack
```

---

## 6. Testing in Kubernetes

### 6.1 Integration Testing

```yaml
# test-job.yaml - Integration Test Job
apiVersion: batch/v1
kind: Job
metadata:
  name: integration-test
  namespace: test
spec:
  template:
    spec:
      containers:
      - name: test
        image: myapp-test:latest
        command:
        - /bin/sh
        - -c
        - |
          # サービスの起動を待つ
          until curl -f http://myapp.test.svc:8080/health; do
            echo "Waiting for myapp..."
            sleep 2
          done

          # Integration test実行
          pytest tests/integration/ -v --junit-xml=/results/results.xml

        volumeMounts:
        - name: test-results
          mountPath: /results

      restartPolicy: Never

      volumes:
      - name: test-results
        emptyDir: {}

  backoffLimit: 3
```

### 6.2 E2E Testing（Kubernetes上）

```yaml
# e2e-test.yaml - E2E Test with Ephemeral Environment
apiVersion: v1
kind: Namespace
metadata:
  name: e2e-test-${{ github.run_id }}

---
# アプリケーションのデプロイ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: e2e-test-${{ github.run_id }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:${{ github.sha }}
        env:
        - name: DATABASE_URL
          value: postgresql://postgres:5432/testdb

---
# データベース（テスト用）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: e2e-test-${{ github.run_id }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_DB
          value: testdb
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust

---
# E2E Test Job
apiVersion: batch/v1
kind: Job
metadata:
  name: e2e-test
  namespace: e2e-test-${{ github.run_id }}
spec:
  template:
    spec:
      containers:
      - name: test
        image: cypress/included:13.0.0
        command:
        - npx
        - cypress
        - run
        - --config
        - baseUrl=http://myapp.e2e-test-${{ github.run_id }}.svc:8080

      restartPolicy: Never
```

このドキュメントは、CI/CD & GitOpsの主要部分をカバーしています。次は「ネットワーク詳細」のドキュメントを作成します。

## 📚 参考リソース

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Flagger Documentation](https://docs.flagger.app/)
- [GitOps Principles](https://opengitops.dev/)
