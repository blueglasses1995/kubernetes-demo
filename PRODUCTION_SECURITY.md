# Kubernetes本番環境セキュリティ完全ガイド

このドキュメントは、Kubernetes本番環境で必須となるセキュリティ観点を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [RBAC（Role-Based Access Control）](#1-rbacrole-based-access-control)
2. [Pod Security](#2-pod-security)
3. [Secrets管理](#3-secrets管理)
4. [イメージセキュリティ](#4-イメージセキュリティ)
5. [Runtime Security](#5-runtime-security)

---

## 1. RBAC（Role-Based Access Control）

### 1.1 RBACの技術的原理

#### レイヤー構造

```
┌─────────────────────────────────────┐
│ Layer 7: kubectl/Client             │  ← ユーザーがリクエスト
├─────────────────────────────────────┤
│ Layer 6: API Server (AuthZ)         │  ← RBAC認可チェック
│   ├─ Authentication                 │
│   ├─ Authorization (RBAC)           │  ← ここでRBACが動作
│   └─ Admission Control              │
├─────────────────────────────────────┤
│ Layer 5: etcd                        │  ← RBAC設定を保存
│   /registry/rbac.authorization.k8s.io/
│   ├─ roles/
│   ├─ rolebindings/
│   ├─ clusterroles/
│   └─ clusterrolebindings/
└─────────────────────────────────────┘
```

#### API Serverでの認可処理フロー

```go
// kube-apiserver内部の認可処理（疑似コード）
func (a *Authorizer) Authorize(ctx context.Context, attrs authorizer.Attributes) (authorized authorizer.Decision, reason string, err error) {
    // 1. リクエスト情報の取得
    user := attrs.GetUser()
    verb := attrs.GetVerb()          // get, list, create, update, delete, watch
    resource := attrs.GetResource()  // pods, services, deployments
    namespace := attrs.GetNamespace()

    // 2. ユーザーに紐づくRoleBindingsを検索
    roleBindings := a.getRoleBindingsForUser(user, namespace)

    // 3. 各RoleBindingのRoleをチェック
    for _, binding := range roleBindings {
        role := a.getRole(binding.RoleRef)

        // 4. Roleのルールを評価
        for _, rule := range role.Rules {
            if a.ruleMatchesRequest(rule, verb, resource) {
                return authorizer.DecisionAllow, "", nil
            }
        }
    }

    // 5. どのルールにもマッチしない場合は拒否
    return authorizer.DecisionDeny, "no matching rule found", nil
}

// ルールマッチング関数
func (a *Authorizer) ruleMatchesRequest(rule rbacv1.PolicyRule, verb, resource string) bool {
    // Verbsチェック
    if !contains(rule.Verbs, verb) && !contains(rule.Verbs, "*") {
        return false
    }

    // Resourcesチェック
    if !contains(rule.Resources, resource) && !contains(rule.Resources, "*") {
        return false
    }

    return true
}
```

#### etcdでのRBAC情報保存

```bash
# etcd内部のRBAC情報（実際の保存形式）
etcdctl get /registry/rbac.authorization.k8s.io/roles/default/pod-reader --print-value-only

# 保存されているデータ（protobuf形式をデコード）
{
  "kind": "Role",
  "apiVersion": "rbac.authorization.k8s.io/v1",
  "metadata": {
    "name": "pod-reader",
    "namespace": "default",
    "uid": "12345678-1234-1234-1234-123456789012",
    "resourceVersion": "1000",
    "creationTimestamp": "2024-01-15T10:00:00Z"
  },
  "rules": [
    {
      "verbs": ["get", "list", "watch"],
      "apiGroups": [""],
      "resources": ["pods"]
    }
  ]
}
```

### 1.2 Role vs ClusterRole の使い分け

#### 技術的な違い

```
┌─────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                 │
├─────────────────────────────────────────────────────┤
│ ClusterRole (クラスタースコープ)                      │
│ ┌─────────────────────────────────────────────┐     │
│ │ ・全namespaceで有効                          │     │
│ │ ・Cluster-scopedリソースにアクセス可能        │     │
│ │   - nodes                                   │     │
│ │   - persistentvolumes                       │     │
│ │   - namespaces                              │     │
│ │ ・etcd保存先: /registry/rbac.../clusterroles/│     │
│ └─────────────────────────────────────────────┘     │
│                                                       │
│ ┌─────────────────┐  ┌─────────────────┐            │
│ │ Namespace: dev  │  │ Namespace: prod │            │
│ ├─────────────────┤  ├─────────────────┤            │
│ │ Role: pod-admin │  │ Role: pod-viewer│            │
│ │ (devだけ有効)   │  │ (prodだけ有効)  │            │
│ └─────────────────┘  └─────────────────┘            │
└─────────────────────────────────────────────────────┘
```

#### 実践例: 開発環境と本番環境の分離

```yaml
# 1. ClusterRole: 全環境共通の基本権限
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-base
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: []  # 全環境でexecは禁止

---
# 2. 開発環境: フル権限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-full-access
  namespace: dev
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["*"]  # 開発環境では全操作を許可

---
# 3. 本番環境: 読み取りのみ
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prod-readonly
  namespace: prod
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]  # 読み取りのみ
- apiGroups: [""]
  resources: ["pods/exec", "pods/attach"]
  verbs: []  # execとattachは明示的に禁止

---
# 4. RoleBinding: 開発者に開発環境のフル権限を付与
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-full-access
  apiGroup: rbac.authorization.k8s.io

---
# 5. RoleBinding: 開発者に本番環境の読み取り権限のみ付与
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-prod-readonly
  namespace: prod
subjects:
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: prod-readonly
  apiGroup: rbac.authorization.k8s.io
```

### 1.3 ServiceAccount の適切な管理

#### ServiceAccountの技術的仕組み

```
Pod起動時のServiceAccount自動マウントフロー:

1. kubelet が Pod を起動
   ↓
2. ServiceAccount Admission Controller が介入
   ↓
3. ServiceAccount の Token を自動生成
   ↓
4. Token を Secret として保存
   ↓
5. Secret を Pod の Volume としてマウント
   /var/run/secrets/kubernetes.io/serviceaccount/
   ├── token          ← JWT Token
   ├── ca.crt         ← クラスターCA証明書
   └── namespace      ← Pod が属する namespace
```

#### JWT Tokenの構造

```bash
# ServiceAccount TokenはJWT形式
# ヘッダー
{
  "alg": "RS256",
  "kid": "cluster-signing-key-id"
}

# ペイロード
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/service-account.name": "my-app",
  "kubernetes.io/serviceaccount/service-account.uid": "12345678-abcd-...",
  "sub": "system:serviceaccount:default:my-app",
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1735689600,  # 有効期限
  "iat": 1704153600,  # 発行時刻
  "nbf": 1704153600   # 有効開始時刻
}

# 署名
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  kube-controller-manager の秘密鍵
)
```

#### Least Privilege の実践

```yaml
# アンチパターン: すべてのPodにdefault ServiceAccountを使用
---
apiVersion: v1
kind: Pod
metadata:
  name: bad-example
spec:
  # serviceAccountName を指定しない → default を使用
  containers:
  - name: app
    image: myapp:1.0

---
# ベストプラクティス: 専用ServiceAccountを作成
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: prod
automountServiceAccountToken: false  # デフォルトで無効化

---
# 最小権限のRoleを定義
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: prod
rules:
# 必要最小限の権限のみ
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["myapp-config"]  # 特定のConfigMapのみ
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["myapp-db-credentials"]  # 特定のSecretのみ
  verbs: ["get"]

---
# RoleBindingで紐付け
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-binding
  namespace: prod
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: prod
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io

---
# Podで明示的にServiceAccountを指定
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: prod
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: true  # このPodでのみ有効化
  containers:
  - name: app
    image: myapp:1.0
```

### 1.4 監査ログ（Audit Logging）

#### 監査ログのレイヤー構造

```
┌─────────────────────────────────────┐
│ kubectl delete pod critical-pod     │  ← 操作実行
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│ kube-apiserver                      │
│ ┌─────────────────────────────────┐ │
│ │ 1. Authentication               │ │
│ │ 2. Authorization (RBAC)         │ │
│ │ 3. Audit Backend ← ★ここで記録  │ │
│ │ 4. Admission Control            │ │
│ │ 5. Validation                   │ │
│ │ 6. etcd書き込み                 │ │
│ └─────────────────────────────────┘ │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│ 監査ログファイル or Webhook          │
│ /var/log/kubernetes/audit.log       │
└─────────────────────────────────────┘
```

#### 監査ポリシーの設定

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 1. Secretsへのアクセスは完全にログ記録（RequestResponseレベル）
- level: RequestResponse
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  resources:
  - group: ""
    resources: ["secrets"]
  omitStages:
  - RequestReceived  # 受信段階は省略、応答後のみ記録

# 2. 本番環境の重要リソース変更を記録
- level: Request
  verbs: ["create", "update", "patch", "delete"]
  namespaces: ["prod", "staging"]
  resources:
  - group: ""
    resources: ["pods", "services", "persistentvolumeclaims"]
  - group: "apps"
    resources: ["deployments", "statefulsets", "daemonsets"]

# 3. RBAC設定変更は必ず記録
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]

# 4. exec/attach/portforwardは完全記録（セキュリティ重要）
- level: RequestResponse
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]

# 5. 認証失敗は記録
- level: Metadata
  omitStages:
  - RequestReceived
  users: ["system:anonymous"]

# 6. 読み取り専用操作は最小限
- level: None
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["events"]

# 7. ヘルスチェックは記録しない
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services"]

# 8. それ以外はMetadataレベル
- level: Metadata
  omitStages:
  - RequestReceived
```

#### API Serverの起動オプション

```bash
# /etc/kubernetes/manifests/kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    image: k8s.gcr.io/kube-apiserver:v1.28.0
    command:
    - kube-apiserver
    # 監査ログ設定
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30      # 30日間保持
    - --audit-log-maxbackup=10   # 10ファイルまで保存
    - --audit-log-maxsize=100    # 100MBでローテーション
    # Webhook送信（オプション）
    - --audit-webhook-config-file=/etc/kubernetes/audit-webhook.yaml
    - --audit-webhook-batch-max-size=100
    - --audit-webhook-batch-max-wait=5s
    volumeMounts:
    - name: audit-policy
      mountPath: /etc/kubernetes/audit-policy.yaml
      readOnly: true
    - name: audit-log
      mountPath: /var/log/kubernetes
  volumes:
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
  - name: audit-log
    hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
```

#### 監査ログの解析例

```bash
# 監査ログのJSON形式
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "RequestResponse",
  "auditID": "12345678-1234-1234-1234-123456789012",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/prod/pods/critical-app",
  "verb": "delete",
  "user": {
    "username": "john@example.com",
    "groups": ["developers", "system:authenticated"]
  },
  "sourceIPs": ["192.168.1.100"],
  "userAgent": "kubectl/v1.28.0",
  "objectRef": {
    "resource": "pods",
    "namespace": "prod",
    "name": "critical-app",
    "apiVersion": "v1"
  },
  "responseStatus": {
    "code": 200
  },
  "requestReceivedTimestamp": "2024-01-15T10:30:00.000000Z",
  "stageTimestamp": "2024-01-15T10:30:00.123456Z",
  "annotations": {
    "authorization.k8s.io/decision": "allow",
    "authorization.k8s.io/reason": "RBAC: allowed by RoleBinding \"prod-admin\""
  }
}
```

```bash
# 監査ログ解析コマンド例

# 1. 本番環境でのPod削除操作を検索
cat audit.log | jq 'select(.objectRef.namespace=="prod" and .verb=="delete" and .objectRef.resource=="pods")'

# 2. 特定ユーザーの操作を追跡
cat audit.log | jq 'select(.user.username=="john@example.com")'

# 3. 認証失敗を検出
cat audit.log | jq 'select(.responseStatus.code >= 401 and .responseStatus.code <= 403)'

# 4. Secret アクセスを監視
cat audit.log | jq 'select(.objectRef.resource=="secrets")'

# 5. exec/attach 操作を検出
cat audit.log | jq 'select(.objectRef.subresource=="exec" or .objectRef.subresource=="attach")'
```

### 1.5 RBAC の可視化とテスト

#### kubectl auth canでの権限確認

```bash
# 1. 自分の権限を確認
kubectl auth can-i create pods
kubectl auth can-i delete deployments --namespace=prod
kubectl auth can-i '*' '*'  # すべてのリソースへの全操作

# 2. 他のユーザー/ServiceAccountの権限を確認（管理者のみ）
kubectl auth can-i list pods --as=john@example.com
kubectl auth can-i delete pods --as=system:serviceaccount:default:myapp-sa --namespace=default

# 3. 特定のリソース名への権限確認
kubectl auth can-i get configmaps/myapp-config --namespace=prod

# 4. サブリソースへの権限確認
kubectl auth can-i create pods/exec --namespace=prod
```

#### rbac-lookup を使った逆引き

```bash
# rbac-lookupのインストール
kubectl krew install rbac-lookup

# 1. 特定ユーザーの全権限を表示
kubectl rbac-lookup john@example.com

# 出力例:
# SUBJECT                    SCOPE       ROLE
# john@example.com          prod        Role/pod-reader
# john@example.com          cluster     ClusterRole/view

# 2. 特定のRoleを持つユーザーを逆引き
kubectl rbac-lookup --kind role --name pod-reader

# 3. ServiceAccountの権限確認
kubectl rbac-lookup -o wide system:serviceaccount:default:myapp-sa
```

#### RBACツールの活用

```bash
# 1. kubectl-who-can: 特定操作が可能なユーザーを検索
kubectl krew install who-can
kubectl who-can delete pods --namespace=prod

# 出力例:
# ROLEBINDING                 NAMESPACE  SUBJECT                KIND            SA-NAMESPACE
# prod-admin                  prod       john@example.com      User
# prod-admin                  prod       admin-team            Group
# system:controller:...       prod       pod-garbage-collector ServiceAccount  kube-system

# 2. rback: RBAC 可視化ツール
kubectl krew install rback
kubectl rback

# 3. Graphviz形式で出力
kubectl rback --output-format dot | dot -Tpng > rbac-graph.png
```

---

## 2. Pod Security

### 2.1 Pod Security Standards (PSS)

#### 3つのセキュリティレベル

```
┌─────────────────────────────────────────────────────┐
│ Privileged (特権レベル)                              │
│ ・制限なし                                          │
│ ・ホストネットワーク、ホストPID使用可能              │
│ ・privileged containers 許可                        │
│ ・すべての capabilities 許可                        │
│ 用途: CNI、CSI、システムレベルのDaemonSet           │
└─────────────────────────────────────────────────────┘
           ▲ より厳格
           │
┌─────────────────────────────────────────────────────┐
│ Baseline (ベースライン)                             │
│ ・既知の特権昇格を防止                              │
│ ・ホストネームスペース使用禁止                      │
│ ・privileged containers 禁止                        │
│ ・危険な capabilities 禁止                          │
│ ・HostPath volume 制限                              │
│ 用途: 標準的なアプリケーション                      │
└─────────────────────────────────────────────────────┘
           ▲ より厳格
           │
┌─────────────────────────────────────────────────────┐
│ Restricted (制限レベル)                             │
│ ・現在のPod Hardening Best Practicesを強制          │
│ ・非rootユーザーで実行を強制                        │
│ ・Read-only root filesystem                         │
│ ・すべての capabilities をdrop                      │
│ ・seccomp, SELinux, AppArmor 強制                   │
│ 用途: セキュリティ要件が高いアプリケーション        │
└─────────────────────────────────────────────────────┘
```

#### Namespace レベルでのPSS適用

```yaml
# Namespace に PSS ラベルを適用
apiVersion: v1
kind: Namespace
metadata:
  name: prod-apps
  labels:
    # enforce: このレベルを満たさないPodは拒否される
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28

    # audit: ログに記録するが拒否はしない
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28

    # warn: ユーザーに警告を表示するが拒否はしない
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
```

#### Restricted レベルに準拠したPod定義

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: prod-apps
spec:
  # 1. 非rootユーザーで実行
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    # 2. seccomp プロファイル適用
    seccompProfile:
      type: RuntimeDefault
    # 3. SELinux 設定（オプション）
    seLinuxOptions:
      level: "s0:c123,c456"

  containers:
  - name: app
    image: myapp:1.0
    # 4. コンテナレベルのセキュリティ設定
    securityContext:
      # root への切り替えを禁止
      allowPrivilegeEscalation: false
      # すべての capabilities を削除
      capabilities:
        drop:
        - ALL
      # Read-only root filesystem
      readOnlyRootFilesystem: true

    # 5. 書き込みが必要な場所は emptyDir を使用
    volumeMounts:
    - name: cache
      mountPath: /app/cache
    - name: tmp
      mountPath: /tmp

  volumes:
  - name: cache
    emptyDir: {}
  - name: tmp
    emptyDir: {}
```

### 2.2 Pod Security Admission

#### Pod Security Admission Controller の動作原理

```
Pod作成リクエストフロー:

kubectl apply -f pod.yaml
         │
         ▼
┌─────────────────────┐
│ kube-apiserver      │
├─────────────────────┤
│ 1. Authentication   │
│ 2. Authorization    │
│ 3. Mutation         │  ← 他の Admission Webhooks
├─────────────────────┤
│ 4. Pod Security     │  ← ★ここでPSSをチェック
│    Admission        │
│  ┌───────────────┐  │
│  │ enforce check │  │ → 違反があれば reject
│  │ audit log     │  │ → 監査ログに記録
│  │ warn user     │  │ → ユーザーに警告表示
│  └───────────────┘  │
├─────────────────────┤
│ 5. Validation       │
│ 6. etcd 保存        │
└─────────────────────┘
```

#### PSS違反時のエラーメッセージ例

```bash
# Restricted レベル違反のPodを作成しようとした場合
kubectl apply -f privileged-pod.yaml

# エラー出力:
Error from server (Forbidden): error when creating "privileged-pod.yaml":
pods "my-pod" is forbidden: violates PodSecurity "restricted:v1.28":
  allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false),
  unrestricted capabilities (container "app" must set securityContext.capabilities.drop=["ALL"]),
  runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true),
  seccompProfile (pod or container "app" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

### 2.3 SecurityContext の詳細設定

#### Linux Capabilities の管理

```yaml
# Linux Capabilities の実践例
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-example
spec:
  containers:
  # ケース1: ネットワークバインドが必要なアプリ（ポート80, 443）
  - name: web-server
    image: nginx:alpine
    securityContext:
      runAsUser: 1000  # 非rootユーザー
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # 1024以下のポートにバインド可能

  # ケース2: ping コマンドが必要なアプリ
  - name: network-monitor
    image: busybox
    securityContext:
      runAsUser: 1000
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - NET_RAW  # raw socket（ping）に必要

  # ケース3: 時刻同期が必要なアプリ
  - name: time-sync
    image: chrony
    securityContext:
      runAsUser: 1000
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
        add:
        - SYS_TIME  # システム時刻の変更に必要
```

#### 主要な Linux Capabilities 一覧

```
# 危険度: 高（通常は許可すべきでない）
CAP_SYS_ADMIN    - システム管理操作全般（マウント、名前空間操作など）
CAP_SYS_MODULE   - カーネルモジュールのロード
CAP_SYS_PTRACE   - プロセスのトレース（デバッグ）
CAP_DAC_OVERRIDE - ファイルパーミッションの無視
CAP_SYS_BOOT     - システム再起動

# 危険度: 中（特定用途で必要）
CAP_NET_ADMIN    - ネットワーク管理操作
CAP_NET_RAW      - raw socket の使用（ping等）
CAP_SYS_TIME     - システム時刻の変更
CAP_CHOWN        - ファイル所有者の変更
CAP_SETUID/SETGID - ユーザー/グループIDの変更

# 危険度: 低（一般的なアプリで使用）
CAP_NET_BIND_SERVICE - 1024以下のポートへのバインド
CAP_KILL         - シグナル送信
```

#### AppArmor プロファイルの適用

```yaml
# AppArmor プロファイルを使用したPod
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-example
  annotations:
    # コンテナ名ごとに AppArmor プロファイルを指定
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
  - name: app
    image: myapp:1.0
```

```bash
# AppArmor プロファイル定義（ホストOS上）
# /etc/apparmor.d/k8s-deny-write

#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  # アプリケーションの実行を許可
  /usr/bin/myapp ix,

  # 読み取りのみ許可
  /etc/** r,
  /usr/** r,
  /app/** r,

  # 特定ディレクトリへの書き込みのみ許可
  /tmp/** rw,
  /var/cache/myapp/** rw,

  # それ以外への書き込みを拒否
  /** w,  # deny（デフォルト拒否）

  # ネットワークアクセス制限
  network inet tcp,
  network inet udp,
  deny network raw,  # raw socket 拒否

  # Capabilities 制限
  capability net_bind_service,
  deny capability sys_admin,
  deny capability sys_module,
}
```

```bash
# AppArmorプロファイルの読み込み（各ノードで実行）
sudo apparmor_parser -r -W /etc/apparmor.d/k8s-deny-write

# プロファイルの確認
sudo aa-status | grep k8s-deny-write
```

---

## 3. Secrets管理

### 3.1 Kubernetes Secrets の制限事項

#### etcd内部でのSecrets保存形式

```
デフォルト設定（暗号化なし）:

┌─────────────────────────────────────┐
│ etcd                                │
│ /registry/secrets/default/db-pass   │
├─────────────────────────────────────┤
│ {                                   │
│   "kind": "Secret",                 │
│   "data": {                         │
│     "password": "cGFzc3dvcmQxMjM=" │  ← base64（暗号化ではない！）
│   }                                 │
│ }                                   │
└─────────────────────────────────────┘
      │
      │ base64 -d
      ▼
   "password123"  ← 平文で復号可能
```

```bash
# etcd から直接 Secret を読み取る例（危険性の実証）

# 1. etcd Pod に接続
kubectl -n kube-system exec -it etcd-master -- sh

# 2. Secret を取得
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/db-password

# 3. 出力（protobuf形式だが、base64部分は読める）
# ...
# data:
#   password: cGFzc3dvcmQxMjM=
# ...

# 4. base64 デコード
echo "cGFzc3dvcmQxMjM=" | base64 -d
# 出力: password123
```

#### Secretsの主な制限事項

```yaml
# 1. サイズ制限: 1MBまで
apiVersion: v1
kind: Secret
metadata:
  name: large-secret
data:
  # 1MB以上のデータは保存できない
  large-file: <base64 encoded data, max 1MB>

# 2. etcd容量への影響
# ・すべてのSecretがetcdに保存される
# ・大量のSecretはetcdを圧迫
# ・etcdのデフォルトクォータ: 2GB

# 3. アクセス制御の粒度
# ・Namespace内のすべてのSecretへのアクセスか、
#   特定のSecretへのアクセスのみ指定可能
# ・Secret内の特定キーへのアクセス制御は不可

# 4. バージョニング非対応
# ・Secretの変更履歴は保存されない
# ・ロールバック機能なし

# 5. ローテーション機能なし
# ・自動的な定期ローテーションの仕組みがない
# ・手動での更新が必要
```

### 3.2 Encryption at Rest（保存時の暗号化）

#### EncryptionConfiguration の設定

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps  # ConfigMapも暗号化可能
  providers:
  # 1. aescbc: AES-CBC 暗号化（推奨）
  - aescbc:
      keys:
      - name: key1
        secret: <base64 encoded 32-byte key>  # 32バイトの鍵をbase64エンコード

  # 2. kms: 外部 KMS を使用（最も安全）
  - kms:
      name: aws-kms-plugin
      endpoint: unix:///var/run/kmsplugin/socket.sock
      cachesize: 1000
      timeout: 3s

  # 3. identity: 暗号化なし（後方互換性のため）
  - identity: {}

# 注意: providersの順序が重要
# - 書き込み時: 最初のprovider（aescbc）を使用
# - 読み取り時: すべてのproviderを順番に試行
```

```bash
# 暗号化キーの生成
head -c 32 /dev/urandom | base64

# 出力例:
# a3ViZXJuZXRlcy1lbmNyeXB0aW9uLXNlY3JldC1rZXkxMjM0NTY3ODkw

# API Server の起動オプションに追加
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    volumeMounts:
    - name: encryption-config
      mountPath: /etc/kubernetes/encryption-config.yaml
      readOnly: true
  volumes:
  - name: encryption-config
    hostPath:
      path: /etc/kubernetes/encryption-config.yaml
      type: File
```

#### 既存Secretsの暗号化

```bash
# Encryption at Rest 有効化後、既存のSecretを再暗号化

# すべてのSecretを読み取り→書き込みして暗号化
kubectl get secrets --all-namespaces -o json | kubectl replace -f -

# 進捗確認
kubectl get secrets --all-namespaces | wc -l

# 暗号化確認: etcd から直接読み取り
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/db-password

# 暗号化されていれば、以下のようなバイナリデータが表示される:
# k8s:enc:aescbc:v1:key1:<encrypted binary data>
```

### 3.3 External Secrets Operator

#### アーキテクチャ

```
┌──────────────────────────────────────────────────┐
│ Kubernetes Cluster                               │
│                                                  │
│ ┌────────────────────────────────────────────┐   │
│ │ ExternalSecret (CRD)                       │   │
│ │ ┌────────────────────────────────────────┐ │   │
│ │ │ apiVersion: external-secrets.io/v1beta1│ │   │
│ │ │ kind: ExternalSecret                   │ │   │
│ │ │ spec:                                  │ │   │
│ │ │   secretStoreRef:                      │ │   │
│ │ │     name: aws-secrets-manager          │ │   │
│ │ │   target:                              │ │   │
│ │ │     name: db-credentials  ← 生成先      │ │   │
│ │ │   data:                                │ │   │
│ │ │   - secretKey: password                │ │   │
│ │ │     remoteRef:                         │ │   │
│ │ │       key: prod/db/password            │ │   │
│ │ └────────────────────────────────────────┘ │   │
│ └────────────────────────────────────────────┘   │
│                     │                            │
│                     ▼                            │
│ ┌────────────────────────────────────────────┐   │
│ │ External Secrets Controller                │   │
│ │ ・ExternalSecretを監視                      │   │
│ │ ・定期的に外部を同期（デフォルト: 1時間）    │   │
│ │ ・Secretを自動作成/更新                     │   │
│ └────────────────────────────────────────────┘   │
│                     │                            │
└─────────────────────┼────────────────────────────┘
                      │ API呼び出し
                      ▼
┌──────────────────────────────────────────────────┐
│ 外部Secretsストア                                │
│ ┌──────────────┐  ┌──────────────┐              │
│ │AWS Secrets   │  │HashiCorp     │              │
│ │Manager       │  │Vault         │              │
│ └──────────────┘  └──────────────┘              │
│ ┌──────────────┐  ┌──────────────┐              │
│ │Google Secret │  │Azure Key     │              │
│ │Manager       │  │Vault         │              │
│ └──────────────┘  └──────────────┘              │
└──────────────────────────────────────────────────┘
```

#### External Secrets Operator のインストール

```bash
# Helm でインストール
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace \
  --set installCRDs=true

# CRDの確認
kubectl get crd | grep external-secrets
# 出力:
# externalsecrets.external-secrets.io
# secretstores.external-secrets.io
# clustersecretstores.external-secrets.io
```

#### AWS Secrets Manager との連携

```yaml
# 1. SecretStore の作成
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: prod
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-1
      auth:
        # Option A: IRSA (IAM Roles for Service Accounts) - 推奨
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

        # Option B: AccessKeyを使用（非推奨）
        # secretRef:
        #   accessKeyIDSecretRef:
        #     name: aws-creds
        #     key: access-key-id
        #   secretAccessKeySecretRef:
        #     name: aws-creds
        #     key: secret-access-key

---
# 2. ServiceAccount with IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ExternalSecretsRole

---
# 3. ExternalSecret の作成
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: prod
spec:
  refreshInterval: 1h  # 1時間ごとに同期
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore

  target:
    name: db-credentials  # 作成されるKubernetes Secret名
    creationPolicy: Owner  # ExternalSecretが削除されたらSecretも削除

  data:
  # 単純な key-value マッピング
  - secretKey: DB_PASSWORD
    remoteRef:
      key: prod/mysql/password  # AWS Secrets Manager のキー

  - secretKey: DB_USERNAME
    remoteRef:
      key: prod/mysql/username

  # JSONからの抽出
  - secretKey: DB_HOST
    remoteRef:
      key: prod/mysql/config  # JSONが保存されているキー
      property: host          # JSON内のプロパティ

  # AWS Secrets Manager のJSON例:
  # {
  #   "host": "mysql.example.com",
  #   "port": 3306
  # }

---
# 4. 生成されたSecretを使用
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: prod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials  # ExternalSecretが自動生成
          key: DB_PASSWORD
```

#### HashiCorp Vault との連携

```yaml
# 1. Vault SecretStore
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: prod
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"  # KV-v2 secrets engine のパス
      version: "v2"
      auth:
        # Kubernetes認証を使用
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets-role"
          serviceAccountRef:
            name: external-secrets-sa

---
# 2. ExternalSecret でVaultからSecretを取得
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
  namespace: prod
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore

  target:
    name: app-credentials

  data:
  - secretKey: api-key
    remoteRef:
      key: prod/api-keys  # Vault path: secret/data/prod/api-keys
      property: key       # JSON field
```

```bash
# Vault側の設定（Vault管理者が実施）

# 1. Kubernetes認証の有効化
vault auth enable kubernetes

# 2. Kubernetes認証の設定
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# 3. Policyの作成
vault policy write external-secrets-policy - <<EOF
path "secret/data/prod/*" {
  capabilities = ["read"]
}
EOF

# 4. Roleの作成
vault write auth/kubernetes/role/external-secrets-role \
  bound_service_account_names=external-secrets-sa \
  bound_service_account_namespaces=prod \
  policies=external-secrets-policy \
  ttl=24h
```

### 3.4 Secrets のローテーション

#### External Secrets Operator によるローテーション

```yaml
# 自動ローテーション設定
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotated-secret
  namespace: prod
spec:
  refreshInterval: 10m  # 10分ごとに確認

  secretStoreRef:
    name: aws-secrets-manager

  target:
    name: app-secret
    template:
      # Secretが更新されたときのアノテーション追加
      metadata:
        annotations:
          reloader.stakater.com/match: "true"  # Reloaderと連携

  data:
  - secretKey: token
    remoteRef:
      key: prod/api-token
```

```yaml
# Reloader によるPod自動再起動
# https://github.com/stakater/Reloader

# 1. Reloader のインストール
---
# helm install reloader stakater/reloader

# 2. Deployment に Reloader アノテーションを追加
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: prod
spec:
  template:
    metadata:
      annotations:
        # このSecretが更新されたらPodを再起動
        secret.reloader.stakater.com/reload: "app-secret"
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: token

# 動作フロー:
# 1. AWS Secrets Manager でトークンを更新
# 2. External Secrets Operator が10分以内に検知
# 3. Kubernetes Secret (app-secret) を更新
# 4. Reloader が Secret 変更を検知
# 5. Deployment のPodをローリング再起動
# 6. 新しいトークンでアプリが起動
```

---

## 4. イメージセキュリティ

### 4.1 イメージスキャニング（Trivy）

#### Trivy の技術的仕組み

```
Trivy スキャンフロー:

1. イメージ取得
   docker pull myapp:1.0
   ↓
2. レイヤー展開
   ├─ layer 1: base OS
   ├─ layer 2: dependencies
   └─ layer 3: application
   ↓
3. パッケージリスト抽出
   ├─ OS packages (apt/yum/apk)
   ├─ Language packages (npm/pip/gem)
   └─ Application binaries
   ↓
4. 脆弱性データベースと照合
   Trivy DB (CVE, GitHub Advisory等)
   ↓
5. レポート生成
   CVE-2024-1234: HIGH
   CVE-2024-5678: CRITICAL
```

```bash
# Trivy のインストール
# macOS
brew install aquasecurity/trivy/trivy

# Linux
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# イメージスキャン
trivy image myapp:1.0

# 出力例:
# myapp:1.0 (alpine 3.18.0)
# ===========================
# Total: 5 (UNKNOWN: 0, LOW: 1, MEDIUM: 2, HIGH: 1, CRITICAL: 1)
#
# ┌───────────────┬────────────────┬──────────┬───────────────────┬───────────────┬──────────────────────┐
# │   Library     │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │       Title          │
# ├───────────────┼────────────────┼──────────┼───────────────────┼───────────────┼──────────────────────┤
# │ openssl       │ CVE-2024-1234  │ CRITICAL │ 3.0.8             │ 3.0.9         │ Remote code execution│
# │ curl          │ CVE-2024-5678  │ HIGH     │ 7.88.0            │ 7.88.1        │ Buffer overflow      │
# └───────────────┴────────────────┴──────────┴───────────────────┴───────────────┴──────────────────────┘

# 重大度でフィルタ
trivy image --severity CRITICAL,HIGH myapp:1.0

# JSON 出力
trivy image -f json -o results.json myapp:1.0

# CI/CD での使用: 脆弱性があれば失敗
trivy image --exit-code 1 --severity CRITICAL myapp:1.0
```

#### CI/CD パイプラインへの統合

```yaml
# GitHub Actions での Trivy スキャン
name: Container Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Build image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'  # 脆弱性があればfail

    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Push image (only if scan passed)
      if: success()
      run: |
        docker tag myapp:${{ github.sha }} myregistry.io/myapp:${{ github.sha }}
        docker push myregistry.io/myapp:${{ github.sha }}
```

### 4.2 Admission Controller によるイメージポリシー強制

```yaml
# OPA Gatekeeper によるイメージポリシー

# 1. ConstraintTemplate: 許可されたレジストリのみ使用
---
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedrepos

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
        not any(satisfied)
        msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.initContainers[_]
        satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
        not any(satisfied)
        msg := sprintf("initContainer <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
      }

---
# 2. Constraint: ポリシーの適用
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: prod-repo-restriction
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - prod
    - staging
  parameters:
    repos:
    - "myregistry.io/"
    - "gcr.io/my-project/"
    # Docker Hubやその他のパブリックレジストリは拒否

# テスト
---
# これは拒否される
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: prod
spec:
  containers:
  - name: app
    image: nginx:latest  # パブリックレジストリ

# エラー:
# Error from server ([prod-repo-restriction] container <app> has an invalid image repo <nginx:latest>, allowed repos are ["myregistry.io/", "gcr.io/my-project/"])

---
# これは許可される
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: prod
spec:
  containers:
  - name: app
    image: myregistry.io/nginx:1.25.0  # 許可されたレジストリ
```

---

## 5. Runtime Security

### 5.1 Falco による runtime detection

#### Falco のアーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│ Kubernetes Node                                     │
│                                                      │
│ ┌────────────────────────────────────────────────┐  │
│ │ Pod: myapp                                     │  │
│ │ └─ Container                                   │  │
│ │    ├─ Process: /bin/sh                         │  │
│ │    └─ Syscall: execve("/bin/sh")  ← 検知       │  │
│ └────────────────────────────────────────────────┘  │
│                      │                               │
│                      │ syscalls                      │
│                      ▼                               │
│ ┌────────────────────────────────────────────────┐  │
│ │ Linux Kernel                                   │  │
│ │ ┌────────────────────────────────────────────┐ │  │
│ │ │ Falco Kernel Driver (kernel module)        │ │  │
│ │ │ or eBPF probe                              │ │  │
│ │ │ ・すべてのsyscallをキャプチャ              │ │  │
│ │ └────────────────────────────────────────────┘ │  │
│ └────────────────────────────────────────────────┘  │
│                      │                               │
│                      │ syscall events                │
│                      ▼                               │
│ ┌────────────────────────────────────────────────┐  │
│ │ Falco Userspace (DaemonSet)                    │  │
│ │ ┌────────────────────────────────────────────┐ │  │
│ │ │ Rule Engine                                │ │  │
│ │ │ ・ルールに基づいてイベントを評価            │ │  │
│ │ │ ・マッチしたらアラート生成                  │ │  │
│ │ └────────────────────────────────────────────┘ │  │
│ └────────────────────────────────────────────────┘  │
│                      │                               │
└──────────────────────┼───────────────────────────────┘
                       │ alerts
                       ▼
┌──────────────────────────────────────────────────────┐
│ Alert Destinations                                   │
│ ├─ Stdout / File                                     │
│ ├─ Webhook (Slack, PagerDuty)                        │
│ ├─ SIEM (Splunk, Elasticsearch)                      │
│ └─ Falco Sidekick (統合alert routing)                │
└──────────────────────────────────────────────────────┘
```

#### Falco のインストール

```bash
# Helm でインストール
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \  # eBPF probe を使用（kernel module不要）
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

# インストール確認
kubectl get pods -n falco

# 出力:
# NAME                                   READY   STATUS    RESTARTS   AGE
# falco-5xj8k                            1/1     Running   0          2m
# falco-g4m2p                            1/1     Running   0          2m
# falco-falcosidekick-...                1/1     Running   0          2m
# falco-falcosidekick-ui-...             1/1     Running   0          2m
```

#### Falco ルールの実例

```yaml
# /etc/falco/falco_rules.local.yaml

# ルール1: シェル起動の検知
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process and
    container and
    shell_procs and
    proc.tty != 0 and
    container_entrypoint and
    not user_expected_terminal_shell_in_container_conditions
  output: >
    Shell spawned in container
    (user=%user.name user_loginuid=%user.loginuid
    container_id=%container.id container_name=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline
    terminal=%proc.tty)
  priority: WARNING
  tags: [container, shell, mitre_execution]

# ルール2: 機密ファイルへのアクセス検知
- rule: Read sensitive file in container
  desc: Detect reads of sensitive files in containers
  condition: >
    open_read and
    container and
    sensitive_files and
    not user_known_read_sensitive_files_activities
  output: >
    Sensitive file opened for reading
    (user=%user.name user_loginuid=%user.loginuid
    file=%fd.name container_id=%container.id
    container_name=%container.name
    command=%proc.cmdline)
  priority: WARNING

# sensitive_files マクロの定義
- macro: sensitive_files
  condition: >
    fd.name in (/etc/shadow, /etc/sudoers, /root/.ssh/id_rsa,
                /root/.ssh/id_dsa, /root/.ssh/id_ecdsa,
                /root/.ssh/id_ed25519,
                /etc/kubernetes/admin.conf)

# ルール3: 予期しないネットワーク接続
- rule: Unexpected outbound connection
  desc: Detect unexpected outbound network connections from containers
  condition: >
    outbound and
    container and
    not user_known_outbound_connection_activities and
    fd.sip != "127.0.0.1" and
    fd.sip != "0.0.0.0"
  output: >
    Unexpected outbound connection
    (user=%user.name user_loginuid=%user.loginuid
    connection=%fd.name container_id=%container.id
    container_name=%container.name
    dest_ip=%fd.rip dest_port=%fd.rport
    proto=%fd.l4proto exe=%proc.name)
  priority: NOTICE

# ルール4: パッケージ管理ツールの実行検知（コンテナ内でのソフトウェアインストール）
- rule: Package Management Launched in Container
  desc: Detect package management tools running inside containers
  condition: >
    spawned_process and
    container and
    user.name != "_apt" and
    package_mgmt_procs and
    not package_mgmt_ancestor_procs and
    not user_known_package_manager_in_container
  output: >
    Package management tool launched in container
    (user=%user.name user_loginuid=%user.loginuid
    command=%proc.cmdline container_id=%container.id
    container_name=%container.name image=%container.image.repository)
  priority: WARNING
  tags: [container, process, mitre_persistence]

- macro: package_mgmt_procs
  condition: proc.name in (apt, apt-get, yum, rpm, dpkg, apk, pip, npm, gem)
```

```bash
# Falco アラートのリアルタイム監視
kubectl logs -f -n falco -l app.kubernetes.io/name=falco

# アラート例:
# 10:30:15.123456789: Warning Shell spawned in container
# (user=root user_loginuid=-1 container_id=abc123
# container_name=myapp-pod shell=bash parent=runc
# cmdline=bash terminal=34816)

# 10:31:22.987654321: Warning Sensitive file opened for reading
# (user=root file=/etc/shadow container_id=abc123
# container_name=myapp-pod command=cat /etc/shadow)
```

#### Falco Sidekick でアラートをSlackに送信

```yaml
# Falco Sidekick 設定
# values.yaml
falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
      minimumpriority: "warning"  # WARNING以上のみ送信
      messageformat: "Alert: *{{.Rule}}* triggered on {{.Hostname}}\nPriority: {{.Priority}}\nOutput: {{.Output}}"

    # 他の通知先も設定可能
    pagerduty:
      routingkey: "YOUR_ROUTING_KEY"
      minimumpriority: "critical"

    elasticsearch:
      hostport: "elasticsearch.logging.svc:9200"
      index: "falco"
      type: "event"
```

---

このドキュメントはセキュリティの基礎部分をカバーしています。次のステップとして「可観測性（Observability）」のドキュメントを作成します。

## 📚 参考リソース

- [Kubernetes RBAC Documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [External Secrets Operator](https://external-secrets.io/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Falco Documentation](https://falco.org/docs/)
- [OPA Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
