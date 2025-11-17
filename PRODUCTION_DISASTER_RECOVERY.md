# Kubernetes本番環境 災害復旧（DR）完全ガイド

このドキュメントは、Kubernetes本番環境で必須となる災害復旧（Disaster Recovery）の実装を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [etcd バックアップ](#1-etcd-バックアップ)
2. [Veleroによるアプリケーションバックアップ](#2-veleroによるアプリケーションバックアップ)
3. [RTO/RPO の設計](#3-rtorpo-の設計)
4. [Multi-cluster DR](#4-multi-cluster-dr)
5. [DR訓練の実施](#5-dr訓練の実施)

---

## 1. etcd バックアップ

### 1.1 etcd の技術的重要性

#### etcd が保存する重要データ

```
etcd は Kubernetes のすべての状態を保存:

/registry/
├── pods/                           # すべてのPod定義
├── services/                       # すべてのService
├── deployments/                    # Deployment、ReplicaSet
├── configmaps/                     # ConfigMap
├── secrets/                        # Secrets（暗号化されている場合も）
├── rbac.authorization.k8s.io/      # RBAC設定
├── namespaces/                     # Namespace定義
├── persistentvolumes/              # PV/PVC
├── customresourcedefinitions/      # CRD定義
└── <custom resources>/             # すべてのカスタムリソース

⚠️ etcdが失われると、クラスター全体が復旧不可能 ⚠️
```

### 1.2 etcdctl を使った手動バックアップ

#### バックアップコマンド

```bash
# etcd Pod内で実行（静的Pod: /etc/kubernetes/manifests/etcd.yaml）
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# 出力例:
# {"level":"info","ts":1705300800.123,"caller":"snapshot/v3_snapshot.go:68","msg":"created temporary db file","path":"/backup/etcd-snapshot-20240115-103000.db.part"}
# {"level":"info","ts":1705300800.456,"msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
# {"level":"info","ts":1705300801.789,"msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"64 MB","took":"now"}
# Snapshot saved at /backup/etcd-snapshot-20240115-103000.db

# スナップショットの検証
ETCDCTL_API=3 etcdctl \
  --write-out=table \
  snapshot status /backup/etcd-snapshot-20240115-103000.db

# 出力:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 12345678 |   123456 |      15234 |     64 MB  |
# +----------+----------+------------+------------+
```

### 1.3 自動バックアップの実装（CronJob）

#### etcd バックアップ CronJob

```yaml
---
# 1. バックアップ用 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: etcd-backup
  namespace: kube-system

---
# 2. RBAC設定（etcd Podへのexec権限）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: etcd-backup
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["get", "list", "create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: etcd-backup
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: etcd-backup
subjects:
- kind: ServiceAccount
  name: etcd-backup
  namespace: kube-system

---
# 3. バックアップスクリプト ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: etcd-backup-script
  namespace: kube-system
data:
  backup.sh: |
    #!/bin/bash
    set -euo pipefail

    # 変数設定
    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    BACKUP_FILE="etcd-snapshot-${TIMESTAMP}.db"
    S3_BUCKET="s3://my-k8s-backups/etcd"
    RETENTION_DAYS=30

    echo "Starting etcd backup at ${TIMESTAMP}"

    # etcd Podの検出
    ETCD_POD=$(kubectl get pods -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')
    echo "etcd Pod: ${ETCD_POD}"

    # etcd スナップショット作成
    kubectl exec -n kube-system ${ETCD_POD} -- sh -c \
      "ETCDCTL_API=3 etcdctl \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        snapshot save /tmp/${BACKUP_FILE}"

    # スナップショットのコピー
    kubectl cp -n kube-system ${ETCD_POD}:/tmp/${BACKUP_FILE} /tmp/${BACKUP_FILE}

    # スナップショットの検証
    ETCDCTL_API=3 etcdctl snapshot status /tmp/${BACKUP_FILE} --write-out=json > /tmp/status.json
    REVISION=$(cat /tmp/status.json | jq -r '.revision')
    TOTAL_KEYS=$(cat /tmp/status.json | jq -r '.totalKey')

    echo "Snapshot created: revision=${REVISION}, keys=${TOTAL_KEYS}"

    # S3にアップロード
    aws s3 cp /tmp/${BACKUP_FILE} ${S3_BUCKET}/${BACKUP_FILE} \
      --metadata "revision=${REVISION},total-keys=${TOTAL_KEYS},timestamp=${TIMESTAMP}"

    echo "Uploaded to ${S3_BUCKET}/${BACKUP_FILE}"

    # 古いバックアップの削除
    echo "Cleaning up old backups (retention: ${RETENTION_DAYS} days)"
    aws s3 ls ${S3_BUCKET}/ | while read -r line; do
      createDate=$(echo $line | awk '{print $1" "$2}')
      createDate=$(date -d "$createDate" +%s)
      olderThan=$(date -d "-${RETENTION_DAYS} days" +%s)
      if [[ $createDate -lt $olderThan ]]; then
        fileName=$(echo $line | awk '{print $4}')
        if [[ $fileName != "" ]]; then
          aws s3 rm ${S3_BUCKET}/${fileName}
          echo "Deleted: ${fileName}"
        fi
      fi
    done

    # ローカルファイルの削除
    rm -f /tmp/${BACKUP_FILE} /tmp/status.json

    echo "Backup completed successfully"

---
# 4. CronJob定義
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  # 毎日午前3時に実行
  schedule: "0 3 * * *"

  # タイムゾーン設定（Kubernetes 1.25+）
  timeZone: "Asia/Tokyo"

  # 成功したJobを3個保持
  successfulJobsHistoryLimit: 3

  # 失敗したJobを1個保持
  failedJobsHistoryLimit: 1

  # 並行実行を禁止
  concurrencyPolicy: Forbid

  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: etcd-backup
        spec:
          serviceAccountName: etcd-backup
          restartPolicy: OnFailure

          containers:
          - name: backup
            image: amazon/aws-cli:2.13.0
            command:
            - /bin/bash
            - /scripts/backup.sh

            env:
            # AWS認証（IRSAを使用）
            - name: AWS_REGION
              value: ap-northeast-1
            - name: AWS_ROLE_ARN
              value: arn:aws:iam::123456789012:role/EtcdBackupRole
            - name: AWS_WEB_IDENTITY_TOKEN_FILE
              value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token

            volumeMounts:
            - name: scripts
              mountPath: /scripts
            - name: aws-token
              mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
              readOnly: true

            resources:
              requests:
                cpu: 100m
                memory: 128Mi
              limits:
                cpu: 500m
                memory: 512Mi

          volumes:
          - name: scripts
            configMap:
              name: etcd-backup-script
              defaultMode: 0755
          - name: aws-token
            projected:
              sources:
              - serviceAccountToken:
                  path: token
                  expirationSeconds: 86400
                  audience: sts.amazonaws.com
```

### 1.4 etcd リストア（復旧）手順

#### リストア実行スクリプト

```bash
#!/bin/bash
# etcd-restore.sh - etcd を完全に復旧するスクリプト

set -euo pipefail

BACKUP_FILE="$1"
ETCD_DATA_DIR="/var/lib/etcd"

echo "=== etcd Restore Procedure ==="
echo "Backup file: ${BACKUP_FILE}"
echo "⚠️  WARNING: This will COMPLETELY REPLACE the current etcd data ⚠️"
read -p "Are you sure you want to continue? (type 'yes' to continue): " confirm

if [ "$confirm" != "yes" ]; then
  echo "Restore cancelled"
  exit 1
fi

# 1. すべてのkube-apiserver を停止
echo "Step 1: Stopping kube-apiserver..."
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 10

# kube-apiserverが停止したことを確認
while pgrep -x "kube-apiserver" > /dev/null; do
  echo "Waiting for kube-apiserver to stop..."
  sleep 2
done
echo "kube-apiserver stopped"

# 2. etcd を停止
echo "Step 2: Stopping etcd..."
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/
sleep 10

# etcdが停止したことを確認
while pgrep -x "etcd" > /dev/null; do
  echo "Waiting for etcd to stop..."
  sleep 2
done
echo "etcd stopped"

# 3. 既存のetcdデータをバックアップ（念のため）
echo "Step 3: Backing up current etcd data..."
sudo mv ${ETCD_DATA_DIR} ${ETCD_DATA_DIR}.backup.$(date +%Y%m%d-%H%M%S)

# 4. スナップショットからリストア
echo "Step 4: Restoring from snapshot..."
sudo ETCDCTL_API=3 etcdctl snapshot restore ${BACKUP_FILE} \
  --data-dir=${ETCD_DATA_DIR} \
  --name=master-1 \
  --initial-cluster=master-1=https://192.168.1.10:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# 権限設定
sudo chown -R etcd:etcd ${ETCD_DATA_DIR}

echo "Snapshot restored to ${ETCD_DATA_DIR}"

# 5. etcd を起動
echo "Step 5: Starting etcd..."
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sleep 10

# etcdが起動したことを確認
until kubectl --kubeconfig=/etc/kubernetes/admin.conf get cs 2>/dev/null | grep -q etcd; do
  echo "Waiting for etcd to start..."
  sleep 2
done
echo "etcd started"

# 6. kube-apiserver を起動
echo "Step 6: Starting kube-apiserver..."
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
sleep 10

# kube-apiserverが起動したことを確認
until kubectl get nodes 2>/dev/null; do
  echo "Waiting for kube-apiserver to start..."
  sleep 2
done
echo "kube-apiserver started"

# 7. クラスターの状態確認
echo "Step 7: Verifying cluster status..."
kubectl get nodes
kubectl get pods --all-namespaces

echo ""
echo "=== Restore completed successfully ==="
echo "Please verify that all resources are restored correctly."
```

---

## 2. Veleroによるアプリケーションバックアップ

### 2.1 Velero のアーキテクチャ

```
┌────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                         │
│                                                             │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ Velero Server (Deployment)                          │    │
│ │ ┌─────────────────────────────────────────────────┐ │    │
│ │ │ Controllers:                                    │ │    │
│ │ │ ・Backup Controller                             │ │    │
│ │ │ ・Restore Controller                            │ │    │
│ │ │ ・Schedule Controller                           │ │    │
│ │ │ ・Download Request Controller                   │ │    │
│ │ └─────────────────────────────────────────────────┘ │    │
│ └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│                          │ kubectl API                      │
│                          ▼                                  │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ Kubernetes API Server                               │    │
│ │ ・Namespaces                                        │    │
│ │ ・Deployments, StatefulSets, etc.                   │    │
│ │ ・ConfigMaps, Secrets                               │    │
│ │ ・PersistentVolumeClaims                            │    │
│ └─────────────────────────────────────────────────────┘    │
│                          │                                  │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ Object Storage (S3 / GCS / Azure Blob)                       │
│ ┌──────────────────────────────────────────────────────┐     │
│ │ backups/                                             │     │
│ │ ├─ backup-20240115-030000/                           │     │
│ │ │  ├─ backup.tar.gz        ← リソース定義           │     │
│ │ │  ├─ backup-logs.gz       ← ログ                   │     │
│ │ │  ├─ backup-resources.json                          │     │
│ │ │  └─ backup-volumesnapshots.json                    │     │
│ │ └─ backup-20240116-030000/                           │     │
│ └──────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘

                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ Volume Snapshots (CSI Snapshots)                             │
│ ├─ EBS Snapshots (AWS)                                       │
│ ├─ Persistent Disk Snapshots (GCP)                           │
│ └─ Disk Snapshots (Azure)                                    │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Velero のインストール

```bash
# 1. Velero CLI のインストール
# macOS
brew install velero

# Linux
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# 2. AWS用の設定（S3バックアップ）
cat > credentials-velero <<EOF
[default]
aws_access_key_id = <ACCESS_KEY_ID>
aws_secret_access_key = <SECRET_ACCESS_KEY>
EOF

# 3. Velero のインストール（AWS）
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=ap-northeast-1 \
  --snapshot-location-config region=ap-northeast-1 \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=true \
  --use-node-agent \
  --uploader-type=restic

# インストール確認
kubectl get pods -n velero

# 出力:
# NAME                      READY   STATUS    RESTARTS   AGE
# velero-7d9f8b4c5d-xyz     1/1     Running   0          2m
# node-agent-abc123         1/1     Running   0          2m
# node-agent-def456         1/1     Running   0          2m
```

### 2.3 バックアップの作成

#### 基本的なバックアップ

```bash
# 1. 特定Namespaceのバックアップ
velero backup create prod-backup \
  --include-namespaces prod \
  --wait

# 2. 特定ラベルを持つリソースのバックアップ
velero backup create app-backup \
  --selector app=myapp

# 3. すべてのNamespaceのバックアップ（kube-system除く）
velero backup create cluster-backup \
  --exclude-namespaces kube-system,kube-public,kube-node-lease,velero

# 4. PersistentVolumeを含むバックアップ
velero backup create fullbackup \
  --include-namespaces prod \
  --snapshot-volumes=true \
  --default-volumes-to-fs-backup  # Resticを使ったファイルシステムバックアップ

# バックアップステータスの確認
velero backup describe prod-backup --details

# 出力例:
# Name:         prod-backup
# Namespace:    velero
# Labels:       velero.io/storage-location=default
# Annotations:  <none>
#
# Phase:  Completed
#
# Namespaces:
#   Included:  prod
#   Excluded:  <none>
#
# Resources:
#   Included:        *
#   Excluded:        <none>
#   Cluster-scoped:  auto
#
# Backup Format Version:  1.1.0
#
# Started:    2024-01-15 03:00:00 +0900 JST
# Completed:  2024-01-15 03:02:30 +0900 JST
#
# Expiration:  2024-02-14 03:00:00 +0900 JST
#
# Total items to be backed up:  150
# Items backed up:              150
#
# Resource List:
#   apps/v1/Deployment:
#     - prod/myapp
#     - prod/nginx
#   v1/Service:
#     - prod/myapp
#     - prod/nginx
#   v1/ConfigMap:
#     - prod/myapp-config
#   v1/Secret:
#     - prod/myapp-secrets
#   v1/PersistentVolumeClaim:
#     - prod/myapp-data
```

#### スケジュールバックアップ

```yaml
# Schedule CRD でのバックアップスケジュール設定
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  # cron形式のスケジュール
  schedule: "0 3 * * *"  # 毎日午前3時

  # バックアップテンプレート
  template:
    # バックアップ対象
    includedNamespaces:
    - prod
    - staging

    # 除外するリソース
    excludedResources:
    - events
    - events.events.k8s.io

    # ボリュームスナップショット
    snapshotVolumes: true
    defaultVolumesToFsBackup: true

    # TTL（保持期間）
    ttl: 720h  # 30日間

    # ラベル
    labelSelector:
      matchExpressions:
      - key: backup
        operator: NotIn
        values:
        - "false"

    # フック（バックアップ前後の処理）
    hooks:
      resources:
      - name: mysql-backup-hook
        includedNamespaces:
        - prod
        labelSelector:
          matchLabels:
            app: mysql
        pre:
        - exec:
            container: mysql
            command:
            - /bin/bash
            - -c
            - "mysqldump -u root -p$MYSQL_ROOT_PASSWORD --all-databases > /backup/dump.sql"
            onError: Fail
            timeout: 10m
        post:
        - exec:
            container: mysql
            command:
            - /bin/bash
            - -c
            - "rm -f /backup/dump.sql"
```

```bash
# CLIでのスケジュール作成
velero schedule create daily-backup \
  --schedule="0 3 * * *" \
  --include-namespaces prod,staging \
  --ttl 720h

# スケジュールの確認
velero schedule get

# 出力:
# NAME           STATUS    CREATED                         SCHEDULE    BACKUP TTL   LAST BACKUP   SELECTOR
# daily-backup   Enabled   2024-01-15 10:00:00 +0900 JST   0 3 * * *   720h0m0s     2h ago        <none>

# 手動でのスケジュール実行
velero backup create --from-schedule daily-backup
```

### 2.4 リストア（復旧）

```bash
# 1. 最新バックアップからのリストア
velero restore create --from-backup prod-backup

# 2. 特定Namespaceのみリストア
velero restore create prod-restore \
  --from-backup cluster-backup \
  --include-namespaces prod

# 3. 特定リソースタイプのみリストア
velero restore create config-restore \
  --from-backup prod-backup \
  --include-resources configmaps,secrets

# 4. 既存リソースを置き換えないリストア
velero restore create careful-restore \
  --from-backup prod-backup \
  --existing-resource-policy=none  # 既存リソースはスキップ

# 5. Namespace を変更してリストア（本番→検証環境）
velero restore create test-restore \
  --from-backup prod-backup \
  --namespace-mappings prod:test

# リストアステータスの確認
velero restore describe prod-restore

# 出力:
# Name:         prod-restore
# Namespace:    velero
# Labels:       <none>
# Annotations:  <none>
#
# Phase:                       Completed
# Backup:                      prod-backup
# Namespaces:
#   Included:  prod
#   Excluded:  <none>
#
# Resources:
#   Included:        *
#   Excluded:        nodes, events, events.events.k8s.io
#   Cluster-scoped:  auto
#
# Namespace mappings:  <none>
#
# Restore PVs:  auto
#
# Started:    2024-01-15 15:00:00 +0900 JST
# Completed:  2024-01-15 15:05:30 +0900 JST
#
# Warnings:  0
# Errors:    0
```

---

## 3. RTO/RPO の設計

### 3.1 RTO と RPO の定義

```
Timeline of a Disaster:

正常運用          障害発生                      復旧完了
   │                │                              │
   ▼                ▼                              ▼
───┴────────────────┴──────────────────────────────┴──────
                    ◄────────── RTO ────────────►

                    ◄── RPO ──►
                    │          │
                 最後の      障害
                バックアップ  発生

RPO (Recovery Point Objective):
  ・許容可能なデータ損失の時間
  ・「どこまでのデータを失っても許容できるか」
  ・例: RPO = 1時間 → 最大1時間分のデータ損失を許容

RTO (Recovery Time Objective):
  ・目標復旧時間
  ・「どれくらいの時間でサービスを復旧させるか」
  ・例: RTO = 4時間 → 4時間以内にサービス復旧
```

### 3.2 サービスレベルごとのRTO/RPO設計

#### Tier 1: ミッションクリティカル

```yaml
# Tier 1: 決済システム、認証システム等
# RTO: 1時間以内
# RPO: 5分以内

---
# 1. etcd バックアップ: 10分ごと
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup-tier1
spec:
  schedule: "*/10 * * * *"  # 10分ごと
  # ... (前述のetcdバックアップ設定)

---
# 2. Velero バックアップ: 1時間ごと
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier1-hourly
spec:
  schedule: "0 * * * *"  # 毎時0分
  template:
    includedNamespaces:
    - payment
    - auth
    labelSelector:
      matchLabels:
        tier: "1"
    snapshotVolumes: true
    ttl: 168h  # 7日間保持

---
# 3. Multi-region レプリケーション
# - クラスターを複数リージョンに配置
# - データベースはマルチリージョンレプリケーション
# - リアルタイム同期
```

#### Tier 2: 重要システム

```yaml
# Tier 2: APIサーバー、管理画面等
# RTO: 4時間以内
# RPO: 1時間以内

---
# 1. etcd バックアップ: 1時間ごと
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup-tier2
spec:
  schedule: "0 * * * *"

---
# 2. Velero バックアップ: 6時間ごと
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier2-6hourly
spec:
  schedule: "0 */6 * * *"  # 6時間ごと
  template:
    includedNamespaces:
    - api
    - admin
    labelSelector:
      matchLabels:
        tier: "2"
    ttl: 720h  # 30日間保持
```

#### Tier 3: 通常システム

```yaml
# Tier 3: 内部ツール、開発環境等
# RTO: 24時間以内
# RPO: 24時間以内

---
# 1. etcd バックアップ: 1日1回
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup-tier3
spec:
  schedule: "0 3 * * *"  # 毎日午前3時

---
# 2. Velero バックアップ: 1日1回
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier3-daily
spec:
  schedule: "0 3 * * *"
  template:
    includedNamespaces:
    - tools
    - dev
    labelSelector:
      matchLabels:
        tier: "3"
    ttl: 2160h  # 90日間保持
```

### 3.3 バックアップ保持ポリシー

```
バックアップ世代管理:

┌─────────────────────────────────────────────────────────┐
│ Grandfather-Father-Son (GFS) 方式                       │
└─────────────────────────────────────────────────────────┘

Daily (Son):
├─ backup-20240115-030000 (今日)
├─ backup-20240114-030000 (1日前)
├─ backup-20240113-030000 (2日前)
├─ backup-20240112-030000 (3日前)
├─ backup-20240111-030000 (4日前)
├─ backup-20240110-030000 (5日前)
└─ backup-20240109-030000 (6日前)
保持: 7日分

Weekly (Father):
├─ backup-20240107-weekly (先週)
├─ backup-20231231-weekly (2週前)
├─ backup-20231224-weekly (3週前)
└─ backup-20231217-weekly (4週前)
保持: 4週分

Monthly (Grandfather):
├─ backup-202401-monthly (今月)
├─ backup-202312-monthly (先月)
├─ backup-202311-monthly (2ヶ月前)
└─ ...
保持: 12ヶ月分
```

```bash
# GFS方式のバックアップスケジュール

# 1. 日次バックアップ（毎日午前3時、7日保持）
velero schedule create daily-backup \
  --schedule="0 3 * * *" \
  --ttl 168h

# 2. 週次バックアップ（毎週日曜午前4時、4週保持）
velero schedule create weekly-backup \
  --schedule="0 4 * * 0" \
  --ttl 672h

# 3. 月次バックアップ（毎月1日午前5時、12ヶ月保持）
velero schedule create monthly-backup \
  --schedule="0 5 1 * *" \
  --ttl 8760h
```

---

## 4. Multi-cluster DR

### 4.1 アクティブ-パッシブ構成

```
┌──────────────────────────────────────────────────────┐
│ Primary Cluster (ap-northeast-1)                     │
│                                                       │
│ ┌──────────────┐      ┌──────────────┐              │
│ │ Production   │      │ Velero       │              │
│ │ Workloads    │      │ (Backup)     │              │
│ └──────────────┘      └──────┬───────┘              │
│                              │                       │
└──────────────────────────────┼───────────────────────┘
                               │
                               │ S3 Replication
                               ▼
┌──────────────────────────────────────────────────────┐
│ S3 Bucket (Multi-region)                             │
│ ┌──────────────────────────────────────────────────┐ │
│ │ Primary: ap-northeast-1                          │ │
│ │ Replica: us-west-2                               │ │
│ └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
                               │
                               │ Restore (DR発生時)
                               ▼
┌──────────────────────────────────────────────────────┐
│ DR Cluster (us-west-2) - 通常は停止                  │
│                                                       │
│ ┌──────────────┐      ┌──────────────┐              │
│ │ Velero       │ ───► │ Production   │              │
│ │ (Restore)    │      │ Workloads    │              │
│ └──────────────┘      └──────────────┘              │
└──────────────────────────────────────────────────────┘
```

#### DR Cluster へのフェイルオーバー手順

```bash
#!/bin/bash
# dr-failover.sh - DR クラスターへのフェイルオーバー

set -euo pipefail

echo "=== Disaster Recovery Failover Procedure ==="
echo "Primary Cluster: production-eks (ap-northeast-1)"
echo "DR Cluster: dr-eks (us-west-2)"
echo ""
read -p "Confirm failover to DR cluster? (type 'FAILOVER' to continue): " confirm

if [ "$confirm" != "FAILOVER" ]; then
  echo "Failover cancelled"
  exit 1
fi

# 1. DR クラスターに切り替え
echo "Step 1: Switching to DR cluster..."
kubectl config use-context dr-eks
kubectl cluster-info

# 2. Velero が利用可能か確認
echo "Step 2: Verifying Velero..."
kubectl get pods -n velero

# 3. 最新のバックアップを確認
echo "Step 3: Finding latest backup..."
LATEST_BACKUP=$(velero backup get -o json | jq -r '.items | sort_by(.status.completionTimestamp) | last | .metadata.name')
echo "Latest backup: ${LATEST_BACKUP}"

velero backup describe ${LATEST_BACKUP}

# 4. リストア実行
echo "Step 4: Restoring from backup..."
velero restore create dr-restore-$(date +%Y%m%d-%H%M%S) \
  --from-backup ${LATEST_BACKUP} \
  --wait

# 5. Podの起動確認
echo "Step 5: Waiting for pods to be ready..."
kubectl wait --for=condition=Ready pods --all -n prod --timeout=600s

# 6. サービスのヘルスチェック
echo "Step 6: Health check..."
kubectl get pods --all-namespaces
kubectl get svc --all-namespaces

# 7. DNSを切り替え（Route53例）
echo "Step 7: Updating DNS..."
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "dr-cluster-alb.us-west-2.elb.amazonaws.com"}]
      }
    }]
  }'

echo ""
echo "=== Failover completed ==="
echo "Services are now running on DR cluster"
echo "Don't forget to:"
echo "1. Notify stakeholders"
echo "2. Monitor the DR cluster closely"
echo "3. Plan for failback when primary is restored"
```

### 4.2 アクティブ-アクティブ構成

```
┌──────────────────────────────────────────────────────┐
│ Cluster 1 (ap-northeast-1)                           │
│ ┌──────────────┐                                     │
│ │ Workloads    │ ◄─┐                                 │
│ │ (Active)     │   │ Multi-cluster Service Mesh     │
│ └──────────────┘   │ (Istio Multi-primary)          │
└────────────────────┼──────────────────────────────────┘
                     │
                     │
┌────────────────────┼──────────────────────────────────┐
│ Cluster 2 (us-west-2)                                 │
│ ┌──────────────┐   │                                 │
│ │ Workloads    │ ◄─┘                                 │
│ │ (Active)     │                                     │
│ └──────────────┘                                     │
└──────────────────────────────────────────────────────┘

Global Load Balancer (AWS Global Accelerator / Cloudflare)
              │
              ├──► Cluster 1 (70% traffic)
              └──► Cluster 2 (30% traffic)
```

---

## 5. DR訓練の実施

### 5.1 DR訓練チェックリスト

```markdown
# DR訓練実施計画書

## 訓練概要
- **目的**: プライマリクラスター障害時のDRクラスターへのフェイルオーバー手順の検証
- **日時**: 2024年2月15日 22:00-24:00
- **対象システム**: 本番環境（production-eks）
- **訓練方式**: シミュレーション（実際のフェイルオーバーは実施しない）

## 事前準備（1週間前）
- [ ] DR訓練参加者の確定
- [ ] DR手順書の最新化
- [ ] バックアップの最新性確認
- [ ] DRクラスターの起動確認
- [ ] 監視ツールの準備
- [ ] ステークホルダーへの通知

## 訓練当日（開始前）
- [ ] 全参加者の接続確認
- [ ] 最新バックアップの確認
- [ ] DRクラスターのリソース確認
- [ ] 通信環境の確認

## 訓練手順
### フェーズ1: 障害検知（22:00-22:10）
- [ ] プライマリクラスター障害のシミュレーション
- [ ] アラート発火の確認
- [ ] オンコール担当者への通知
- [ ] インシデント宣言

### フェーズ2: 初動対応（22:10-22:20）
- [ ] インシデント対応チームの招集
- [ ] 状況の把握と影響範囲の特定
- [ ] DR発動の意思決定
- [ ] ステークホルダーへの報告

### フェーズ3: DR実行（22:20-23:00）
- [ ] DRクラスターへのコンテキスト切り替え
- [ ] 最新バックアップの特定
- [ ] リストアの実行
- [ ] Pod起動状態の確認
- [ ] サービスヘルスチェック
- [ ] DNS切り替え（シミュレーション）

### フェーズ4: 検証（23:00-23:30）
- [ ] 各サービスの動作確認
- [ ] データ整合性の確認
- [ ] パフォーマンステスト
- [ ] 外部連携の確認

### フェーズ5: 振り返り（23:30-24:00）
- [ ] 実施内容のレビュー
- [ ] 問題点の洗い出し
- [ ] 改善事項の記録
- [ ] 次回訓練日程の確認

## 成功基準
- [ ] RTO: 40分以内にリストア完了（目標: 60分）
- [ ] RPO: データ損失5分以内（目標: 15分）
- [ ] すべてのCriticalサービスが正常稼働
- [ ] 手順書どおりに実施できた

## 訓練後対応
- [ ] 訓練レポートの作成
- [ ] DR手順書の更新
- [ ] 改善事項の対応
- [ ] ステークホルダーへの報告
```

### 5.2 訓練自動化スクリプト

```bash
#!/bin/bash
# dr-drill.sh - DR訓練の自動実行スクリプト

set -euo pipefail

DRILL_DATE=$(date +%Y%m%d-%H%M%S)
REPORT_FILE="dr-drill-report-${DRILL_DATE}.md"

echo "=== DR Drill Started at ${DRILL_DATE} ===" | tee ${REPORT_FILE}

# タイマー開始
START_TIME=$(date +%s)

# Phase 1: 障害検知
echo "" | tee -a ${REPORT_FILE}
echo "## Phase 1: Failure Detection" | tee -a ${REPORT_FILE}
PHASE1_START=$(date +%s)

echo "Simulating primary cluster failure..." | tee -a ${REPORT_FILE}
# （実際の訓練では、プライマリクラスターのネットワークを一時的に遮断する等）

PHASE1_END=$(date +%s)
PHASE1_DURATION=$((PHASE1_END - PHASE1_START))
echo "Phase 1 completed in ${PHASE1_DURATION} seconds" | tee -a ${REPORT_FILE}

# Phase 2: DR実行
echo "" | tee -a ${REPORT_FILE}
echo "## Phase 2: DR Execution" | tee -a ${REPORT_FILE}
PHASE2_START=$(date +%s)

# DRクラスターに切り替え
echo "Switching to DR cluster..." | tee -a ${REPORT_FILE}
kubectl config use-context dr-eks

# 最新バックアップの取得
LATEST_BACKUP=$(velero backup get -o json | jq -r '.items | sort_by(.status.completionTimestamp) | last | .metadata.name')
echo "Latest backup: ${LATEST_BACKUP}" | tee -a ${REPORT_FILE}

# バックアップの詳細を記録
velero backup describe ${LATEST_BACKUP} >> ${REPORT_FILE}

# リストア実行
RESTORE_NAME="dr-drill-${DRILL_DATE}"
echo "Starting restore: ${RESTORE_NAME}..." | tee -a ${REPORT_FILE}
velero restore create ${RESTORE_NAME} \
  --from-backup ${LATEST_BACKUP} \
  --wait

PHASE2_END=$(date +%s)
PHASE2_DURATION=$((PHASE2_END - PHASE2_START))
echo "Phase 2 completed in ${PHASE2_DURATION} seconds" | tee -a ${REPORT_FILE}

# Phase 3: 検証
echo "" | tee -a ${REPORT_FILE}
echo "## Phase 3: Verification" | tee -a ${REPORT_FILE}
PHASE3_START=$(date +%s)

# Pod状態の確認
echo "Checking pod status..." | tee -a ${REPORT_FILE}
kubectl get pods --all-namespaces -o wide >> ${REPORT_FILE}

# サービスヘルスチェック
echo "Running health checks..." | tee -a ${REPORT_FILE}
HEALTH_CHECK_FAILED=0

# 各サービスのヘルスチェック
for service in api frontend backend; do
  echo "Checking ${service}..." | tee -a ${REPORT_FILE}
  if kubectl exec -n prod deploy/${service} -- curl -f http://localhost:8080/health; then
    echo "✓ ${service} is healthy" | tee -a ${REPORT_FILE}
  else
    echo "✗ ${service} health check failed" | tee -a ${REPORT_FILE}
    HEALTH_CHECK_FAILED=1
  fi
done

PHASE3_END=$(date +%s)
PHASE3_DURATION=$((PHASE3_END - PHASE3_START))
echo "Phase 3 completed in ${PHASE3_DURATION} seconds" | tee -a ${REPORT_FILE}

# 総合評価
END_TIME=$(date +%s)
TOTAL_DURATION=$((END_TIME - START_TIME))
RTO_TARGET=3600  # 60分

echo "" | tee -a ${REPORT_FILE}
echo "## Summary" | tee -a ${REPORT_FILE}
echo "- Total duration: ${TOTAL_DURATION} seconds ($((TOTAL_DURATION / 60)) minutes)" | tee -a ${REPORT_FILE}
echo "- RTO target: ${RTO_TARGET} seconds ($((RTO_TARGET / 60)) minutes)" | tee -a ${REPORT_FILE}

if [ ${TOTAL_DURATION} -le ${RTO_TARGET} ] && [ ${HEALTH_CHECK_FAILED} -eq 0 ]; then
  echo "- Result: ✓ SUCCESS" | tee -a ${REPORT_FILE}
  EXIT_CODE=0
else
  echo "- Result: ✗ FAILED" | tee -a ${REPORT_FILE}
  EXIT_CODE=1
fi

echo "" | tee -a ${REPORT_FILE}
echo "Report saved to: ${REPORT_FILE}"

exit ${EXIT_CODE}
```

---

このドキュメントは災害復旧の主要部分をカバーしています。次は「クラスター運用」と「トラブルシューティング」のドキュメントを作成します。

## 📚 参考リソース

- [etcd Disaster Recovery](https://etcd.io/docs/latest/op-guide/recovery/)
- [Velero Documentation](https://velero.io/docs/)
- [AWS Backup for EKS](https://docs.aws.amazon.com/eks/latest/userguide/backup-restore.html)
- [Kubernetes Disaster Recovery Best Practices](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)
