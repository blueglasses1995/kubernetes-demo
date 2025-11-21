# Kubernetes本番環境 ストレージ完全ガイド

このドキュメントは、Kubernetes本番環境でのストレージの実装を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [CSI (Container Storage Interface)](#1-csi-container-storage-interface)
2. [StatefulSetとステートフルアプリケーション](#2-statefulsetとステートフルアプリケーション)
3. [Volume Types](#3-volume-types)
4. [ストレージパフォーマンス](#4-ストレージパフォーマンス)
5. [バックアップとスナップショット](#5-バックアップとスナップショット)

---

## 1. CSI (Container Storage Interface)

### 1.1 CSI アーキテクチャ

```
┌──────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                       │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ kubelet                                             │  │
│ │ ・VolumeAttachの監視                                │  │
│ │ ・CSI Node Plugin呼び出し                           │  │
│ └────────────────┬────────────────────────────────────┘  │
│                  │ gRPC                                  │
│                  ▼                                       │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ CSI Node Plugin (DaemonSet)                         │  │
│ │ ┌─────────────────────────────────────────────────┐ │  │
│ │ │ Node Service                                    │ │  │
│ │ │ ・NodePublishVolume (マウント)                  │ │  │
│ │ │ ・NodeUnpublishVolume (アンマウント)            │ │  │
│ │ │ ・NodeStageVolume (デバイスアタッチ)            │ │  │
│ │ │ ・NodeUnstageVolume (デバイスデタッチ)          │ │  │
│ │ └─────────────────────────────────────────────────┘ │  │
│ └─────────────────────────────────────────────────────┘  │
│                                                           │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ kube-controller-manager                             │  │
│ │ ・PVC/PV の管理                                     │  │
│ │ ・Attach/Detach Controller                          │  │
│ └────────────────┬────────────────────────────────────┘  │
│                  │                                       │
│                  ▼                                       │
│ ┌─────────────────────────────────────────────────────┐  │
│ │ CSI Controller Plugin (Deployment)                  │  │
│ │ ┌─────────────────────────────────────────────────┐ │  │
│ │ │ Controller Service                              │ │  │
│ │ │ ・CreateVolume (ボリューム作成)                 │ │  │
│ │ │ ・DeleteVolume (ボリューム削除)                 │ │  │
│ │ │ ・ControllerPublishVolume (Nodeにアタッチ)      │ │  │
│ │ │ ・ControllerUnpublishVolume (Nodeからデタッチ)  │ │  │
│ │ │ ・CreateSnapshot (スナップショット作成)         │ │  │
│ │ └─────────────────────────────────────────────────┘ │  │
│ │                                                      │  │
│ │ ┌─────────────────────────────────────────────────┐ │  │
│ │ │ Sidecar Containers                              │ │  │
│ │ │ ・external-provisioner                          │ │  │
│ │ │ ・external-attacher                             │ │  │
│ │ │ ・external-snapshotter                          │ │  │
│ │ │ ・external-resizer                              │ │  │
│ │ └─────────────────────────────────────────────────┘ │  │
│ └─────────────────────────────────────────────────────┘  │
│                  │ Cloud Provider API                    │
└──────────────────┼───────────────────────────────────────┘
                   ▼
┌──────────────────────────────────────────────────────────┐
│ Cloud Provider (AWS EBS / GCP PD / Azure Disk)           │
│ ・ボリュームの実際の作成/削除                            │
│ ・スナップショットの作成                                  │
└──────────────────────────────────────────────────────────┘
```

### 1.2 AWS EBS CSI Driver

#### インストール

```bash
# IAM Policy作成（EBSボリュームの作成/削除権限）
cat > ebs-csi-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSnapshot",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumesModifications"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ],
      "Condition": {
        "StringEquals": {
          "ec2:CreateAction": [
            "CreateVolume",
            "CreateSnapshot"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteTags"
      ],
      "Resource": [
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*:*:snapshot/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:RequestTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteVolume"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/kubernetes.io/created-for/pvc/name": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/CSIVolumeSnapshotName": "*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/ebs.csi.aws.com/cluster": "true"
        }
      }
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
  --policy-document file://ebs-csi-policy.json

# IRSA (IAM Roles for Service Accounts) 設定
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster production-eks \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Helm でインストール
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=ebs-csi-controller-sa
```

#### StorageClass 定義

```yaml
# gp3-storage-class.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
  # gp3固有のパラメータ
  iops: "3000"          # 3000-16000
  throughput: "125"     # 125-1000 MiB/s
volumeBindingMode: WaitForFirstConsumer  # Podがスケジュールされるまで作成を遅延
allowVolumeExpansion: true
reclaimPolicy: Delete

---
# io2（高IOPS）StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "10000"         # 最大64000
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain  # 削除時にボリュームを保持
```

### 1.3 Volume Lifecycle

```yaml
# PersistentVolumeClaim作成
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
  namespace: prod
spec:
  accessModes:
  - ReadWriteOnce  # RWO: 単一Nodeからのみアクセス
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi

---
# Podでの使用
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: prod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data

  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: myapp-data

# ライフサイクル:
# 1. PVC作成 → status: Pending
# 2. Pod作成 → CSI Controller が EBS ボリューム作成
# 3. ボリュームが作成 → PVC status: Bound
# 4. CSI Node が EBS を Node にアタッチ
# 5. CSI Node がファイルシステムをマウント
# 6. Pod起動
```

#### Volume 拡張

```bash
# PVCサイズの拡張（allowVolumeExpansion: true が必要）
kubectl patch pvc myapp-data -n prod -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# 拡張の進行状況確認
kubectl describe pvc myapp-data -n prod

# Events:
#   Type    Reason                      Age   From                         Message
#   ----    ------                      ----  ----                         -------
#   Normal  FileSystemResizeRequired    5s    volume_expand                Waiting for user to (re-)start a pod to finish file system resize of volume.
#   Normal  FileSystemResizeSuccessful  2s    kubelet                      MountVolume.NodeExpandVolume succeeded

# Podを再起動して反映
kubectl delete pod myapp -n prod
```

---

## 2. StatefulSetとステートフルアプリケーション

### 2.1 StatefulSet の動作原理

```yaml
# StatefulSet - 順序保証、安定したネットワークID
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: prod
spec:
  serviceName: mysql  # Headless Service名（必須）
  replicas: 3
  selector:
    matchLabels:
      app: mysql

  # Pod Management Policy
  podManagementPolicy: OrderedReady  # または Parallel

  # Update Strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # partition以上のordinalのPodのみ更新

  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql

        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password

        # データ永続化
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql

        # Liveness/Readiness Probe
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5

        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -e
            - "SELECT 1"
          initialDelaySeconds: 5
          periodSeconds: 2

  # VolumeClaimTemplate（StatefulSet専用）
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi

# StatefulSetの特徴:
# 1. Pod名: mysql-0, mysql-1, mysql-2 （順序保証）
# 2. PVC名: data-mysql-0, data-mysql-1, data-mysql-2 （自動作成）
# 3. DNS: mysql-0.mysql.prod.svc.cluster.local （安定）
# 4. 起動順序: 0 → 1 → 2 （順次）
# 5. 終了順序: 2 → 1 → 0 （逆順）
```

### 2.2 Headless Service

```yaml
# Headless Service（StatefulSetと組み合わせ）
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: prod
spec:
  clusterIP: None  # Headless
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql

# DNS レコード:
# mysql.prod.svc.cluster.local
#   → 10.244.1.10 (mysql-0.mysql.prod.svc.cluster.local)
#   → 10.244.1.11 (mysql-1.mysql.prod.svc.cluster.local)
#   → 10.244.1.12 (mysql-2.mysql.prod.svc.cluster.local)

# 各Podへの直接アクセス:
# mysql-0.mysql.prod.svc.cluster.local → 10.244.1.10
```

### 2.3 StatefulSet の更新戦略

```yaml
# RollingUpdate with Partition
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  replicas: 5
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 3  # ordinal >= 3 のPodのみ更新

# 更新手順（Canary式）:
# 1. partition: 4 → web-4 のみ更新
# 2. テスト・検証
# 3. partition: 3 → web-3, web-4 更新
# 4. partition: 2 → web-2, web-3, web-4 更新
# 5. partition: 0 → すべて更新
```

### 2.4 データベースクラスターの実装例（MySQL）

```yaml
# MySQL Primary-Replica 構成
---
# Primary用 StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-primary
  namespace: prod
spec:
  serviceName: mysql-primary
  replicas: 1
  selector:
    matchLabels:
      app: mysql
      role: primary
  template:
    metadata:
      labels:
        app: mysql
        role: primary
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: myapp
        - name: MYSQL_REPLICATION_MODE
          value: master
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: config
        configMap:
          name: mysql-primary-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: io2  # 高IOPS
      resources:
        requests:
          storage: 200Gi

---
# Replica用 StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-replica
  namespace: prod
spec:
  serviceName: mysql-replica
  replicas: 2
  selector:
    matchLabels:
      app: mysql
      role: replica
  template:
    metadata:
      labels:
        app: mysql
        role: replica
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_REPLICATION_MODE
          value: slave
        - name: MYSQL_MASTER_HOST
          value: mysql-primary-0.mysql-primary.prod.svc.cluster.local
        - name: MYSQL_MASTER_PORT
          value: "3306"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: config
        configMap:
          name: mysql-replica-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources:
        requests:
          storage: 200Gi

---
# Service（Primary - 書き込み用）
apiVersion: v1
kind: Service
metadata:
  name: mysql-primary
  namespace: prod
spec:
  clusterIP: None
  selector:
    app: mysql
    role: primary
  ports:
  - port: 3306

---
# Service（Replica - 読み取り用）
apiVersion: v1
kind: Service
metadata:
  name: mysql-replica
  namespace: prod
spec:
  selector:
    app: mysql
    role: replica
  ports:
  - port: 3306
```

---

## 3. Volume Types

### 3.1 emptyDir

```yaml
# emptyDir - Pod削除時にデータも削除
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: cache
      mountPath: /cache
    - name: tmp
      mountPath: /tmp

  volumes:
  # メモリバックの emptyDir（高速、揮発性）
  - name: cache
    emptyDir:
      medium: Memory  # tmpfs（RAM）を使用
      sizeLimit: 1Gi

  # ディスクバックの emptyDir
  - name: tmp
    emptyDir: {}

# 用途:
# - 一時ファイル
# - キャッシュ
# - コンテナ間でのファイル共有（同一Pod内）
```

### 3.2 ConfigMap と Secret

```yaml
# ConfigMap をVolumeとしてマウント
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
  settings.json: |
    {
      "debug": false
    }

---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true

  volumes:
  - name: config
    configMap:
      name: app-config
      items:  # 特定のキーのみマウント（オプション）
      - key: config.yaml
        path: config.yaml
        mode: 0644

# マウント結果:
# /etc/config/config.yaml
# /etc/config/settings.json

---
# Secret をVolumeとしてマウント
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
      defaultMode: 0400  # 読み取り専用
```

### 3.3 Projected Volumes

```yaml
# 複数のVolumeソースを1つのディレクトリに投影
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: all-in-one
      mountPath: /projected-volume
      readOnly: true

  volumes:
  - name: all-in-one
    projected:
      sources:
      # ConfigMap
      - configMap:
          name: app-config
          items:
          - key: config.yaml
            path: config/app.yaml

      # Secret
      - secret:
          name: app-secrets
          items:
          - key: password
            path: secrets/db-password

      # DownwardAPI（Pod情報）
      - downwardAPI:
          items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations

      # ServiceAccount Token
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200
          audience: api-server

# マウント結果:
# /projected-volume/
# ├── config/
# │   └── app.yaml
# ├── secrets/
# │   └── db-password
# ├── labels
# ├── annotations
# └── token
```

---

## 4. ストレージパフォーマンス

### 4.1 ストレージクラスの選択

#### AWS EBS タイプ比較

| タイプ | IOPS | スループット | レイテンシ | 価格 | 用途 |
|-------|------|------------|----------|------|------|
| **gp3** | 3,000-16,000 | 125-1,000 MB/s | 低 | 安 | 汎用（推奨） |
| **gp2** | 100-16,000 | 128-250 MB/s | 低 | 中 | レガシー |
| **io2** | 100-64,000 | 256-4,000 MB/s | 最低 | 高 | データベース |
| **st1** | 500 | 40-500 MB/s | 高 | 非常に安 | ログ、バックアップ |
| **sc1** | 250 | 12-250 MB/s | 高 | 最安 | アーカイブ |

```yaml
# パフォーマンス要件別のStorageClass

# 1. 汎用（推奨）
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: general-purpose
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# 2. 高IOPSデータベース用
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "20000"     # プロビジョンドIOPS
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# 3. 大容量ストリーミング用
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: throughput-optimized
provisioner: ebs.csi.aws.com
parameters:
  type: st1          # Throughput Optimized HDD
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### 4.2 I/O パフォーマンステスト

```yaml
# fio を使ったI/Oベンチマーク
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: benchmark-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi

---
apiVersion: batch/v1
kind: Job
metadata:
  name: fio-benchmark
spec:
  template:
    spec:
      containers:
      - name: fio
        image: ljishen/fio:latest
        command:
        - /bin/sh
        - -c
        - |
          # ランダムリード
          fio --name=randread \
              --ioengine=libaio \
              --iodepth=32 \
              --rw=randread \
              --bs=4k \
              --direct=1 \
              --size=10G \
              --numjobs=4 \
              --runtime=60 \
              --group_reporting \
              --filename=/data/test

          # ランダムライト
          fio --name=randwrite \
              --ioengine=libaio \
              --iodepth=32 \
              --rw=randwrite \
              --bs=4k \
              --direct=1 \
              --size=10G \
              --numjobs=4 \
              --runtime=60 \
              --group_reporting \
              --filename=/data/test

          # シーケンシャルリード
          fio --name=seqread \
              --ioengine=libaio \
              --iodepth=32 \
              --rw=read \
              --bs=1m \
              --direct=1 \
              --size=10G \
              --numjobs=1 \
              --runtime=60 \
              --group_reporting \
              --filename=/data/test

        volumeMounts:
        - name: data
          mountPath: /data

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: benchmark-pvc

      restartPolicy: Never
```

---

## 5. バックアップとスナップショット

### 5.1 VolumeSnapshot（CSI）

```yaml
# VolumeSnapshotClass
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete  # または Retain
parameters:
  tagSpecification_1: "Name=EBSSnapshot"
  tagSpecification_2: "Environment=production"

---
# VolumeSnapshot作成
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: myapp-snapshot
  namespace: prod
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: myapp-data

# スナップショット確認
# kubectl get volumesnapshot -n prod
# NAME              READYTOUSE   SOURCEPVC    RESTOREVOLUME   AGE
# myapp-snapshot    true         myapp-data   100Gi           5m

---
# スナップショットからのリストア
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data-restored
  namespace: prod
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: myapp-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### 5.2 スケジュールスナップショット

```yaml
# VolumeSnapshotを定期的に作成（CronJob）
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: snapshot-cronjob
  namespace: prod
spec:
  schedule: "0 2 * * *"  # 毎日午前2時
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          containers:
          - name: snapshot
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              SNAPSHOT_NAME="myapp-snapshot-${TIMESTAMP}"

              # スナップショット作成
              kubectl apply -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: ${SNAPSHOT_NAME}
                namespace: prod
              spec:
                volumeSnapshotClassName: ebs-snapshot-class
                source:
                  persistentVolumeClaimName: myapp-data
              EOF

              # 古いスナップショットの削除（30日以上）
              kubectl get volumesnapshot -n prod -o json | \
                jq -r '.items[] | select(.metadata.creationTimestamp < (now - 2592000 | todate)) | .metadata.name' | \
                xargs -r kubectl delete volumesnapshot -n prod

          restartPolicy: OnFailure

---
# ServiceAccount とRBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-creator
  namespace: prod

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-creator
  namespace: prod
rules:
- apiGroups: ["snapshot.storage.k8s.io"]
  resources: ["volumesnapshots"]
  verbs: ["get", "list", "create", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-creator
  namespace: prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: snapshot-creator
subjects:
- kind: ServiceAccount
  name: snapshot-creator
  namespace: prod
```

このドキュメントは、Kubernetesストレージの主要部分をカバーしています。

## 📚 参考リソース

- [CSI Specification](https://github.com/container-storage-interface/spec)
- [AWS EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- [StatefulSet Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
