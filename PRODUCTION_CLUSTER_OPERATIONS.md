# Kubernetes本番環境 クラスター運用完全ガイド

このドキュメントは、Kubernetes本番環境で必須となるクラスター運用を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [クラスターアップグレード](#1-クラスターアップグレード)
2. [ノードメンテナンス](#2-ノードメンテナンス)
3. [証明書管理](#3-証明書管理)
4. [容量計画](#4-容量計画)

---

## 1. クラスターアップグレード

### 1.1 Kubernetes Version Upgrade 戦略

#### バージョンスキューポリシー

```
Kubernetesコンポーネントのバージョン互換性:

kube-apiserver: v1.28.x  ← 基準バージョン
       │
       ├─ kube-controller-manager: v1.28.x または v1.27.x
       ├─ kube-scheduler:          v1.28.x または v1.27.x
       ├─ kubelet:                 v1.28.x, v1.27.x, または v1.26.x
       ├─ kube-proxy:              v1.28.x, v1.27.x, または v1.26.x
       └─ kubectl:                 v1.29.x, v1.28.x, または v1.27.x

ルール:
1. kube-apiserver が最新である必要がある
2. controller-manager, scheduler は apiserver の ±1 バージョン
3. kubelet, kube-proxy は apiserver の -2 までサポート
4. kubectl は apiserver の ±1 バージョン

⚠️ バージョンスキップは非推奨 ⚠️
例: v1.26 → v1.28 は NG
     v1.26 → v1.27 → v1.28 と段階的にアップグレード
```

### 1.2 アップグレード手順（kubeadm）

#### コントロールプレーンのアップグレード

```bash
#!/bin/bash
# upgrade-control-plane.sh - コントロールプレーンノードのアップグレード

set -euo pipefail

CURRENT_VERSION="1.27.8"
TARGET_VERSION="1.28.4"

echo "=== Kubernetes Control Plane Upgrade ==="
echo "Current version: ${CURRENT_VERSION}"
echo "Target version:  ${TARGET_VERSION}"
echo ""

# 1. 最新のkubeadmをインストール
echo "Step 1: Upgrading kubeadm..."
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=${TARGET_VERSION}-00
apt-mark hold kubeadm

# kubeadm バージョン確認
kubeadm version

# 2. アップグレードプランの確認
echo "Step 2: Checking upgrade plan..."
kubeadm upgrade plan

# 出力例:
# [upgrade/config] Making sure the configuration is correct:
# [upgrade/config] Reading configuration from the cluster...
# [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
# [preflight] Running pre-flight checks.
# [upgrade] Running cluster health checks
# [upgrade] Fetching available versions to upgrade to
# [upgrade/versions] Cluster version: v1.27.8
# [upgrade/versions] kubeadm version: v1.28.4
# [upgrade/versions] Target version: v1.28.4
#
# Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
# COMPONENT   CURRENT       TARGET
# kubelet     3 x v1.27.8   v1.28.4
#
# Upgrade to the latest version in the v1.28 series:
#
# COMPONENT                 CURRENT   TARGET
# kube-apiserver            v1.27.8   v1.28.4
# kube-controller-manager   v1.27.8   v1.28.4
# kube-scheduler            v1.27.8   v1.28.4
# kube-proxy                v1.27.8   v1.28.4
# CoreDNS                   v1.10.1   v1.10.1
# etcd                      3.5.9-0   3.5.9-0

echo ""
read -p "Proceed with upgrade? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    echo "Upgrade cancelled"
    exit 1
fi

# 3. 最初のコントロールプレーンノードをアップグレード
echo "Step 3: Applying upgrade..."
kubeadm upgrade apply v${TARGET_VERSION} -y

# 4. CNIプラグインのアップグレード（必要に応じて）
echo "Step 4: Upgrading CNI..."
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 5. ノードのdrain（Podの退避）
echo "Step 5: Draining node..."
kubectl drain $(hostname) --ignore-daemonsets --delete-emptydir-data

# 6. kubelet と kubectl のアップグレード
echo "Step 6: Upgrading kubelet and kubectl..."
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=${TARGET_VERSION}-00 kubectl=${TARGET_VERSION}-00
apt-mark hold kubelet kubectl

# 7. kubelet の再起動
echo "Step 7: Restarting kubelet..."
systemctl daemon-reload
systemctl restart kubelet

# 8. ノードのuncordon（Podのスケジュール再開）
echo "Step 8: Uncordoning node..."
kubectl uncordon $(hostname)

# 9. 確認
echo "Step 9: Verification..."
kubectl get nodes
kubectl version

echo ""
echo "=== Control plane upgrade completed ==="
echo "Next steps:"
echo "1. Verify cluster functionality"
echo "2. Upgrade other control plane nodes (if HA)"
echo "3. Upgrade worker nodes"
```

#### ワーカーノードのアップグレード

```bash
#!/bin/bash
# upgrade-worker.sh - ワーカーノードのアップグレード

set -euo pipefail

TARGET_VERSION="1.28.4"
NODE_NAME=$(hostname)

echo "=== Worker Node Upgrade: ${NODE_NAME} ==="

# 1. ノードのdrain（マスターノードから実行）
echo "Step 1: Draining node ${NODE_NAME}..."
# この手順はマスターノードから実行:
# kubectl drain ${NODE_NAME} --ignore-daemonsets --delete-emptydir-data --force --grace-period=300

# PodDisruptionBudget を考慮したdrain
# kubectl drain ${NODE_NAME} \
#   --ignore-daemonsets \
#   --delete-emptydir-data \
#   --force \
#   --grace-period=300 \
#   --timeout=600s \
#   --pod-selector='app!=critical-service'

# 2. kubeadm のアップグレード
echo "Step 2: Upgrading kubeadm..."
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=${TARGET_VERSION}-00
apt-mark hold kubeadm

# 3. ノードのアップグレード
echo "Step 3: Upgrading node configuration..."
kubeadm upgrade node

# 4. kubelet と kubectl のアップグレード
echo "Step 4: Upgrading kubelet and kubectl..."
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=${TARGET_VERSION}-00 kubectl=${TARGET_VERSION}-00
apt-mark hold kubelet kubectl

# 5. kubelet の再起動
echo "Step 5: Restarting kubelet..."
systemctl daemon-reload
systemctl restart kubelet

# 6. ノードのuncordon（マスターノードから実行）
echo "Step 6: Uncordoning node ${NODE_NAME}..."
# この手順はマスターノードから実行:
# kubectl uncordon ${NODE_NAME}

echo ""
echo "=== Worker node upgrade completed ==="
```

### 1.3 マネージドKubernetesのアップグレード（EKS例）

```bash
#!/bin/bash
# eks-upgrade.sh - EKSクラスターのアップグレード

set -euo pipefail

CLUSTER_NAME="production-eks"
CURRENT_VERSION="1.27"
TARGET_VERSION="1.28"
REGION="ap-northeast-1"

echo "=== EKS Cluster Upgrade ==="
echo "Cluster: ${CLUSTER_NAME}"
echo "Current version: ${CURRENT_VERSION}"
echo "Target version: ${TARGET_VERSION}"

# 1. 事前確認
echo "Step 1: Pre-upgrade checks..."

# アドオンのバージョン確認
aws eks describe-addon-versions \
    --kubernetes-version ${TARGET_VERSION} \
    --region ${REGION}

# 非推奨APIの確認
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# 2. コントロールプレーンのアップグレード
echo "Step 2: Upgrading EKS control plane..."
aws eks update-cluster-version \
    --name ${CLUSTER_NAME} \
    --kubernetes-version ${TARGET_VERSION} \
    --region ${REGION}

# アップグレードの進行状況を監視
while true; do
    STATUS=$(aws eks describe-update \
        --name ${CLUSTER_NAME} \
        --update-id $(aws eks list-updates --name ${CLUSTER_NAME} --region ${REGION} --query 'updateIds[0]' --output text) \
        --region ${REGION} \
        --query 'update.status' \
        --output text)

    echo "Update status: ${STATUS}"

    if [ "${STATUS}" = "Successful" ]; then
        echo "Control plane upgrade completed"
        break
    elif [ "${STATUS}" = "Failed" ]; then
        echo "Control plane upgrade failed"
        exit 1
    fi

    sleep 30
done

# 3. アドオンのアップグレード
echo "Step 3: Upgrading EKS addons..."

# kube-proxy
aws eks update-addon \
    --cluster-name ${CLUSTER_NAME} \
    --addon-name kube-proxy \
    --addon-version $(aws eks describe-addon-versions --kubernetes-version ${TARGET_VERSION} --addon-name kube-proxy --query 'addons[0].addonVersions[0].addonVersion' --output text) \
    --region ${REGION}

# CoreDNS
aws eks update-addon \
    --cluster-name ${CLUSTER_NAME} \
    --addon-name coredns \
    --addon-version $(aws eks describe-addon-versions --kubernetes-version ${TARGET_VERSION} --addon-name coredns --query 'addons[0].addonVersions[0].addonVersion' --output text) \
    --region ${REGION}

# VPC CNI
aws eks update-addon \
    --cluster-name ${CLUSTER_NAME} \
    --addon-name vpc-cni \
    --addon-version $(aws eks describe-addon-versions --kubernetes-version ${TARGET_VERSION} --addon-name vpc-cni --query 'addons[0].addonVersions[0].addonVersion' --output text) \
    --region ${REGION}

# 4. ノードグループのアップグレード
echo "Step 4: Upgrading node groups..."

# 各ノードグループを順次アップグレード
for NODE_GROUP in $(aws eks list-nodegroups --cluster-name ${CLUSTER_NAME} --region ${REGION} --query 'nodegroups[]' --output text); do
    echo "Upgrading node group: ${NODE_GROUP}"

    # ノードグループのアップグレード
    aws eks update-nodegroup-version \
        --cluster-name ${CLUSTER_NAME} \
        --nodegroup-name ${NODE_GROUP} \
        --region ${REGION}

    # 完了を待つ
    aws eks wait nodegroup-active \
        --cluster-name ${CLUSTER_NAME} \
        --nodegroup-name ${NODE_GROUP} \
        --region ${REGION}

    echo "Node group ${NODE_GROUP} upgraded"
done

# 5. 確認
echo "Step 5: Verification..."
kubectl version
kubectl get nodes

echo ""
echo "=== EKS cluster upgrade completed ==="
```

---

## 2. ノードメンテナンス

### 2.1 Drain と Cordon

#### Drain（Podの安全な退避）

```bash
# 基本的な drain
kubectl drain node-1 --ignore-daemonsets

# より詳細な設定
kubectl drain node-1 \
  --ignore-daemonsets \              # DaemonSet Podは無視
  --delete-emptydir-data \           # emptyDir を使用しているPodも削除
  --force \                          # ReplicaSetに管理されていないPodも強制削除
  --grace-period=300 \               # 終了猶予期間（秒）
  --timeout=600s \                   # drain操作のタイムアウト
  --pod-selector='tier!=critical'    # 特定のPodのみdrain

# Drain が失敗する一般的なケース:
# 1. PodDisruptionBudget (PDB) により、最小レプリカ数を下回る
# 2. local storage (emptyDir) を使用しているPod
# 3. Standalone Pod（ReplicaSetに管理されていない）

# PDB を確認
kubectl get pdb --all-namespaces

# 例:
# NAMESPACE   NAME           MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# prod        myapp-pdb      2               N/A               0                     30d
```

#### Cordon（新規Podのスケジュール停止）

```bash
# ノードへの新規Podのスケジュールを停止（既存Podは残る）
kubectl cordon node-1

# 複数ノードを一括cordon
kubectl cordon node-1 node-2 node-3

# 確認
kubectl get nodes
# NAME     STATUS                     ROLES    AGE   VERSION
# node-1   Ready,SchedulingDisabled   <none>   30d   v1.28.0

# Uncordon（スケジュール再開）
kubectl uncordon node-1
```

### 2.2 OS Patching

#### ローリングアップデート方式

```bash
#!/bin/bash
# rolling-os-patch.sh - ワーカーノードのOSパッチをローリング適用

set -euo pipefail

NODES=$(kubectl get nodes -l node-role.kubernetes.io/worker=true -o name | sed 's/node\///')

for NODE in ${NODES}; do
    echo "=== Patching node: ${NODE} ==="

    # 1. Cordon
    echo "Cordoning ${NODE}..."
    kubectl cordon ${NODE}

    # 2. Drain
    echo "Draining ${NODE}..."
    kubectl drain ${NODE} \
        --ignore-daemonsets \
        --delete-emptydir-data \
        --force \
        --grace-period=300 \
        --timeout=600s

    # 3. OSパッチの適用（SSH経由）
    echo "Applying OS patches on ${NODE}..."
    ssh ${NODE} sudo apt-get update
    ssh ${NODE} sudo apt-get upgrade -y

    # カーネルアップデートがある場合は再起動
    if ssh ${NODE} '[ -f /var/run/reboot-required ]'; then
        echo "Reboot required for ${NODE}"
        ssh ${NODE} sudo reboot

        # ノードがオフラインになるまで待機
        sleep 30

        # ノードがオンラインに戻るまで待機
        until ssh ${NODE} uptime 2>/dev/null; do
            echo "Waiting for ${NODE} to come back online..."
            sleep 10
        done

        # kubeletが起動するまで待機
        until kubectl get node ${NODE} | grep -q Ready; do
            echo "Waiting for kubelet on ${NODE}..."
            sleep 10
        done
    fi

    # 4. Uncordon
    echo "Uncordoning ${NODE}..."
    kubectl uncordon ${NODE}

    # 5. Podが正常にスケジュールされるまで待機
    sleep 60

    # 6. クラスターの健全性確認
    echo "Checking cluster health..."
    kubectl get nodes
    kubectl get pods --all-namespaces | grep -v Running | grep -v Completed || echo "All pods are running"

    echo "Node ${NODE} patching completed"
    echo ""
done

echo "=== All nodes patched successfully ==="
```

### 2.3 Node Problem Detector

#### Node Problem Detector のデプロイ

```yaml
# Node Problem Detector DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-problem-detector
  namespace: kube-system
  labels:
    app: node-problem-detector
spec:
  selector:
    matchLabels:
      app: node-problem-detector
  template:
    metadata:
      labels:
        app: node-problem-detector
    spec:
      # すべてのノードに配置
      tolerations:
      - effect: NoSchedule
        operator: Exists

      hostNetwork: true

      containers:
      - name: node-problem-detector
        image: registry.k8s.io/node-problem-detector/node-problem-detector:v0.8.13
        command:
        - /node-problem-detector
        - --logtostderr
        - --config.system-log-monitor=/config/kernel-monitor.json
        - --config.system-log-monitor=/config/docker-monitor.json
        - --config.custom-plugin-monitor=/config/custom-plugin-monitor.json

        securityContext:
          privileged: true

        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName

        volumeMounts:
        # ログファイルへのアクセス
        - name: log
          mountPath: /var/log
          readOnly: true
        - name: kmsg
          mountPath: /dev/kmsg
          readOnly: true
        # ホストの時刻を使用
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
        # 設定ファイル
        - name: config
          mountPath: /config
          readOnly: true

      volumes:
      - name: log
        hostPath:
          path: /var/log/
      - name: kmsg
        hostPath:
          path: /dev/kmsg
      - name: localtime
        hostPath:
          path: /etc/localtime
      - name: config
        configMap:
          name: node-problem-detector-config

---
# 設定ファイル
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-problem-detector-config
  namespace: kube-system
data:
  kernel-monitor.json: |
    {
      "plugin": "kmsg",
      "logPath": "/dev/kmsg",
      "lookback": "5m",
      "bufferSize": 10,
      "source": "kernel-monitor",
      "conditions": [
        {
          "type": "KernelDeadlock",
          "reason": "KernelHasNoDeadlock",
          "message": "kernel has no deadlock"
        },
        {
          "type": "ReadonlyFilesystem",
          "reason": "FilesystemIsReadOnly",
          "message": "Filesystem is read-only"
        }
      ],
      "rules": [
        {
          "type": "temporary",
          "reason": "OOMKilling",
          "pattern": "Kill process \\d+ (.+) score \\d+ or sacrifice child\\nKilled process \\d+ (.+) total-vm:\\d+kB, anon-rss:\\d+kB, file-rss:\\d+kB.*"
        },
        {
          "type": "permanent",
          "condition": "KernelDeadlock",
          "reason": "TaskHung",
          "pattern": "task \\S+:\\w+ blocked for more than \\w+ seconds\\."
        },
        {
          "type": "permanent",
          "condition": "ReadonlyFilesystem",
          "reason": "FilesystemReadonly",
          "pattern": "Remounting filesystem read-only"
        }
      ]
    }

  docker-monitor.json: |
    {
      "plugin": "journald",
      "pluginConfig": {
        "source": "dockerd"
      },
      "logPath": "/var/log/journal",
      "lookback": "5m",
      "bufferSize": 10,
      "source": "docker-monitor",
      "conditions": [],
      "rules": [
        {
          "type": "temporary",
          "reason": "CorruptDockerImage",
          "pattern": "Error trying v2 registry: failed to register layer: rename /var/lib/docker/image/(.+) /var/lib/docker/image/(.+): directory not empty.*"
        }
      ]
    }

  custom-plugin-monitor.json: |
    {
      "plugin": "custom",
      "pluginConfig": {
        "invoke_interval": "30s",
        "timeout": "5s",
        "max_output_length": 80,
        "concurrency": 3
      },
      "source": "custom-plugin-monitor",
      "conditions": [
        {
          "type": "FrequentUnregisterNetDevice",
          "reason": "NoFrequentUnregisterNetDevice",
          "message": "node is functioning properly"
        }
      ],
      "rules": [
        {
          "type": "permanent",
          "condition": "FrequentUnregisterNetDevice",
          "reason": "UnregisterNetDevice",
          "path": "/config/plugin/check_unregister_net_device.sh",
          "timeout": "3s"
        }
      ]
    }
```

#### Node Conditionの確認

```bash
# ノードの状態確認
kubectl describe node node-1

# 出力例（Conditions部分）:
# Conditions:
#   Type                          Status  LastHeartbeatTime                 LastTransitionTime                Reason                          Message
#   ----                          ------  -----------------                 ------------------                ------                          -------
#   FrequentUnregisterNetDevice   False   Mon, 15 Jan 2024 10:30:00 +0900   Mon, 15 Jan 2024 08:00:00 +0900   NoFrequentUnregisterNetDevice   node is functioning properly
#   KernelDeadlock                False   Mon, 15 Jan 2024 10:30:00 +0900   Mon, 15 Jan 2024 08:00:00 +0900   KernelHasNoDeadlock             kernel has no deadlock
#   ReadonlyFilesystem            False   Mon, 15 Jan 2024 10:30:00 +0900   Mon, 15 Jan 2024 08:00:00 +0900   FilesystemIsNotReadOnly         Filesystem is not read-only
#   MemoryPressure                False   Mon, 15 Jan 2024 10:30:15 +0900   Mon, 15 Jan 2024 08:00:00 +0900   KubeletHasSufficientMemory      kubelet has sufficient memory available
#   DiskPressure                  False   Mon, 15 Jan 2024 10:30:15 +0900   Mon, 15 Jan 2024 08:00:00 +0900   KubeletHasNoDiskPressure        kubelet has no disk pressure
#   PIDPressure                   False   Mon, 15 Jan 2024 10:30:15 +0900   Mon, 15 Jan 2024 08:00:00 +0900   KubeletHasSufficientPID         kubelet has sufficient PID available
#   Ready                         True    Mon, 15 Jan 2024 10:30:15 +0900   Mon, 15 Jan 2024 08:00:00 +0900   KubeletReady                    kubelet is posting ready status

# 問題のあるノードのみフィルタ
kubectl get nodes -o json | jq '.items[] | select(.status.conditions[] | select(.type=="Ready" and .status!="True")) | .metadata.name'
```

---

## 3. 証明書管理

### 3.1 Kubernetes 証明書の構造

#### 証明書の種類と用途

```
Kubernetes証明書の階層構造:

/etc/kubernetes/pki/
├── ca.crt                              # クラスターCA（root CA）
├── ca.key                              # クラスターCA秘密鍵
├── apiserver.crt                       # kube-apiserver サーバー証明書
├── apiserver.key                       # kube-apiserver 秘密鍵
├── apiserver-kubelet-client.crt        # apiserver → kubelet クライアント証明書
├── apiserver-kubelet-client.key        # apiserver → kubelet クライアント秘密鍵
├── front-proxy-ca.crt                  # フロントプロキシCA
├── front-proxy-ca.key                  # フロントプロキシCA秘密鍵
├── front-proxy-client.crt              # フロントプロキシクライアント証明書
├── front-proxy-client.key              # フロントプロキシクライアント秘密鍵
├── sa.pub                              # ServiceAccount公開鍵
├── sa.key                              # ServiceAccount秘密鍵
│
└── etcd/
    ├── ca.crt                          # etcd CA
    ├── ca.key                          # etcd CA秘密鍵
    ├── server.crt                      # etcd サーバー証明書
    ├── server.key                      # etcd サーバー秘密鍵
    ├── peer.crt                        # etcd ピア通信証明書
    ├── peer.key                        # etcd ピア通信秘密鍵
    ├── healthcheck-client.crt          # etcd ヘルスチェッククライアント証明書
    └── healthcheck-client.key          # etcd ヘルスチェッククライアント秘密鍵

証明書の用途:
┌──────────────────────────────────────────────────────────┐
│ ca.crt/ca.key                                            │
│ ・クラスター全体のルートCA                                │
│ ・すべてのコンポーネント証明書を署名                      │
│ ・有効期限: 10年（デフォルト）                            │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ apiserver.crt/apiserver.key                              │
│ ・kube-apiserver のTLSサーバー証明書                     │
│ ・クライアント（kubectl, kubelet等）が検証               │
│ ・SAN: kubernetes, kubernetes.default, <各種IP/DNS>     │
│ ・有効期限: 1年（デフォルト）⚠️                          │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│ apiserver-kubelet-client.crt/apiserver-kubelet-client.key│
│ ・kube-apiserver が kubelet にアクセスする際の           │
│   クライアント証明書                                     │
│ ・有効期限: 1年（デフォルト）⚠️                          │
└──────────────────────────────────────────────────────────┘
```

### 3.2 証明書の有効期限確認

```bash
# すべての証明書の有効期限を確認
kubeadm certs check-expiration

# 出力例:
# [check-expiration] Reading configuration from the cluster...
# [check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
#
# CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
# admin.conf                 Jan 15, 2025 08:00 UTC   364d            ca                      no
# apiserver                  Jan 15, 2025 08:00 UTC   364d            ca                      no
# apiserver-etcd-client      Jan 15, 2025 08:00 UTC   364d            etcd-ca                 no
# apiserver-kubelet-client   Jan 15, 2025 08:00 UTC   364d            ca                      no
# controller-manager.conf    Jan 15, 2025 08:00 UTC   364d            ca                      no
# etcd-healthcheck-client    Jan 15, 2025 08:00 UTC   364d            etcd-ca                 no
# etcd-peer                  Jan 15, 2025 08:00 UTC   364d            etcd-ca                 no
# etcd-server                Jan 15, 2025 08:00 UTC   364d            etcd-ca                 no
# front-proxy-client         Jan 15, 2025 08:00 UTC   364d            front-proxy-ca          no
# scheduler.conf             Jan 15, 2025 08:00 UTC   364d            ca                      no
#
# CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
# ca                      Jan 12, 2034 08:00 UTC   9y              no
# etcd-ca                 Jan 12, 2034 08:00 UTC   9y              no
# front-proxy-ca          Jan 12, 2034 08:00 UTC   9y              no

# 個別の証明書確認
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text

# 有効期限のみ表示
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
# notBefore=Jan 15 08:00:00 2024 GMT
# notAfter=Jan 15 08:00:00 2025 GMT
```

### 3.3 証明書の手動更新

```bash
#!/bin/bash
# renew-certificates.sh - Kubernetes証明書の更新

set -euo pipefail

echo "=== Kubernetes Certificate Renewal ==="

# 1. 証明書の有効期限確認
echo "Step 1: Checking certificate expiration..."
kubeadm certs check-expiration

# 2. すべての証明書を更新
echo "Step 2: Renewing all certificates..."
kubeadm certs renew all

# 出力例:
# [renew] Reading configuration from the cluster...
# [renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
#
# certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
# certificate for serving the Kubernetes API renewed
# certificate the apiserver uses to access etcd renewed
# certificate for the API server to connect to kubelet renewed
# certificate embedded in the kubeconfig file for the controller manager to use renewed
# certificate for liveness probes to healthcheck etcd renewed
# certificate for etcd nodes to communicate with each other renewed
# certificate for serving etcd renewed
# certificate for the front proxy client renewed
# certificate embedded in the kubeconfig file for the scheduler manager to use renewed

# 3. kubeconfigファイルの更新（必要に応じて）
echo "Step 3: Updating kubeconfig files..."
kubeadm init phase kubeconfig all

# 4. kubeconfigをホームディレクトリにコピー
cp /etc/kubernetes/admin.conf ~/.kube/config

# 5. コントロールプレーンコンポーネントの再起動
echo "Step 4: Restarting control plane components..."

# 静的Podの再起動（manifestを一時的に移動）
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

sleep 10

mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

# Podが起動するまで待機
echo "Waiting for control plane to restart..."
sleep 30

# 6. 確認
echo "Step 5: Verification..."
kubeadm certs check-expiration
kubectl get nodes
kubectl get pods -n kube-system

echo ""
echo "=== Certificate renewal completed ==="
```

### 3.4 証明書の自動更新（cert-manager）

```yaml
# cert-manager でのKubernetes証明書管理
# ※通常はクラスター証明書には使用しませんが、Ingressや内部サービスの証明書には利用

---
# cert-manager のインストール
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

---
# ClusterIssuer: Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    # HTTP-01 challenge
    - http01:
        ingress:
          class: nginx
    # DNS-01 challenge（ワイルドカード証明書用）
    - dns01:
        route53:
          region: ap-northeast-1
          hostedZoneID: Z1234567890ABC
          role: arn:aws:iam::123456789012:role/cert-manager

---
# Certificate: Ingress用の証明書
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: prod
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: example.com
  dnsNames:
  - example.com
  - www.example.com
  - api.example.com

  # 自動更新設定
  renewBefore: 720h  # 30日前に更新開始

  # 証明書のライフサイクル
  duration: 2160h    # 90日（Let's Encryptの最大期間）

---
# Ingress with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: prod
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-com-tls  # cert-managerが自動作成
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

## 4. 容量計画

### 4.1 クラスターサイジング

#### リソース要件の算出

```
容量計画の基本式:

必要ノード数 = (総Pod数 × 平均Pod要求リソース) / (ノード容量 × リソース使用率)

例:
- 総Pod数: 500
- 平均CPU要求: 0.5 core/pod
- 平均メモリ要求: 512 MB/pod
- ノード容量: 8 cores, 32 GB RAM
- 目標リソース使用率: 70%

CPU計算:
必要CPU = 500 pods × 0.5 cores = 250 cores
ノードあたりCPU = 8 cores × 0.7 = 5.6 cores
必要ノード数（CPU） = 250 / 5.6 = 45 ノード

メモリ計算:
必要メモリ = 500 pods × 512 MB = 256 GB
ノードあたりメモリ = 32 GB × 0.7 = 22.4 GB
必要ノード数（メモリ） = 256 / 22.4 = 12 ノード

→ CPU制約のため、45ノード必要

バッファとHA考慮:
- N+1冗長性: +1ノード
- ローリングアップデート: +20% (9ノード)
- バーストトラフィック: +30% (14ノード)

最終的な推奨ノード数: 45 + 1 + 9 + 14 = 69 ノード
```

#### 実際の使用状況の測定

```bash
# クラスター全体のリソース使用状況
kubectl top nodes

# 出力例:
# NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1    1200m        15%    8Gi             26%
# node-2    2100m        26%    12Gi            39%
# node-3    890m         11%    6Gi             19%

# Pod単位のリソース使用状況
kubectl top pods --all-namespaces

# Namespace単位のリソース集計
kubectl top pods -n prod | awk '{sum+=$2} END {print "Total CPU: " sum "m"}'
kubectl top pods -n prod | awk '{sum+=$3} END {print "Total Memory: " sum "Mi"}'

# リソース要求/制限の集計
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  .spec.containers[] |
  select(.resources.requests != null) |
  {
    namespace: .metadata.namespace,
    pod: .metadata.name,
    container: .name,
    cpu_request: .resources.requests.cpu,
    memory_request: .resources.requests.memory,
    cpu_limit: .resources.limits.cpu,
    memory_limit: .resources.limits.memory
  }
' | jq -s 'group_by(.namespace) | map({namespace: .[0].namespace, pods: length})'
```

### 4.2 Growth Planning（成長予測）

```python
# growth_forecast.py - リソース使用量の予測

import pandas as pd
import numpy as np
from sklearn.linear_model import LinearRegression
import matplotlib.pyplot as plt

# Prometheusから過去90日間のメトリクスを取得（疑似データ）
dates = pd.date_range('2023-10-15', '2024-01-15', freq='D')
# 実際はPrometheusから以下のクエリで取得:
# sum(rate(container_cpu_usage_seconds_total[1h]))
# sum(container_memory_working_set_bytes)

cpu_usage = np.linspace(100, 250, len(dates)) + np.random.normal(0, 10, len(dates))
memory_usage = np.linspace(500, 1200, len(dates)) + np.random.normal(0, 50, len(dates))

df = pd.DataFrame({
    'date': dates,
    'cpu_cores': cpu_usage,
    'memory_gb': memory_usage
})

# 線形回帰モデル
X = np.arange(len(df)).reshape(-1, 1)
y_cpu = df['cpu_cores'].values
y_memory = df['memory_gb'].values

model_cpu = LinearRegression().fit(X, y_cpu)
model_memory = LinearRegression().fit(X, y_memory)

# 今後90日間の予測
future_days = 90
future_X = np.arange(len(df), len(df) + future_days).reshape(-1, 1)

predicted_cpu = model_cpu.predict(future_X)
predicted_memory = model_memory.predict(future_X)

# 結果出力
print("=== Growth Forecast (90 days) ===")
print(f"Current CPU usage: {y_cpu[-1]:.1f} cores")
print(f"Predicted CPU usage: {predicted_cpu[-1]:.1f} cores")
print(f"Growth rate: {((predicted_cpu[-1] / y_cpu[-1]) - 1) * 100:.1f}%")
print()
print(f"Current Memory usage: {y_memory[-1]:.1f} GB")
print(f"Predicted Memory usage: {predicted_memory[-1]:.1f} GB")
print(f"Growth rate: {((predicted_memory[-1] / y_memory[-1]) - 1) * 100:.1f}%")

# 容量不足の警告
cluster_cpu_capacity = 500  # cores
cluster_memory_capacity = 2000  # GB

if predicted_cpu[-1] > cluster_cpu_capacity * 0.8:
    print(f"\n⚠️  CPU capacity warning: Predicted usage will exceed 80% in {future_days} days")
    days_until_full = int((cluster_cpu_capacity * 0.8 - y_cpu[-1]) / (model_cpu.coef_[0]))
    print(f"    Estimated days until 80% capacity: {days_until_full}")

if predicted_memory[-1] > cluster_memory_capacity * 0.8:
    print(f"\n⚠️  Memory capacity warning: Predicted usage will exceed 80% in {future_days} days")
    days_until_full = int((cluster_memory_capacity * 0.8 - y_memory[-1]) / (model_memory.coef_[0]))
    print(f"    Estimated days until 80% capacity: {days_until_full}")
```

### 4.3 Bin Packing 効率

```bash
# Bin packing 効率の測定
# ・ノードのリソースがどれだけ効率的に使用されているか

# 各ノードのリソース割当効率を計算
kubectl get nodes -o json | jq -r '
  .items[] |
  {
    name: .metadata.name,
    allocatable_cpu: .status.allocatable.cpu,
    allocatable_memory: .status.allocatable.memory
  }
' > /tmp/nodes.json

kubectl get pods --all-namespaces -o json | jq -r '
  .items[] |
  select(.status.phase == "Running") |
  {
    node: .spec.nodeName,
    cpu_request: (.spec.containers[].resources.requests.cpu // "0"),
    memory_request: (.spec.containers[].resources.requests.memory // "0")
  }
' > /tmp/pods.json

# Pythonスクリプトで集計
cat > /tmp/binpacking.py << 'EOF'
import json
import re

def parse_cpu(cpu_str):
    """CPU文字列（1000m, 1など）をmillicoresに変換"""
    if cpu_str.endswith('m'):
        return int(cpu_str[:-1])
    else:
        return int(float(cpu_str) * 1000)

def parse_memory(mem_str):
    """メモリ文字列（1Gi, 1000Miなど）をMiBに変換"""
    if mem_str.endswith('Ki'):
        return int(mem_str[:-2]) / 1024
    elif mem_str.endswith('Mi'):
        return int(mem_str[:-2])
    elif mem_str.endswith('Gi'):
        return int(mem_str[:-2]) * 1024
    else:
        return int(mem_str) / 1024 / 1024

# ノードデータ読み込み
with open('/tmp/nodes.json') as f:
    nodes = {}
    for line in f:
        node = json.loads(line)
        nodes[node['name']] = {
            'cpu': parse_cpu(node['allocatable_cpu']),
            'memory': parse_memory(node['allocatable_memory']),
            'used_cpu': 0,
            'used_memory': 0
        }

# Podデータ読み込み
with open('/tmp/pods.json') as f:
    for line in f:
        pod = json.loads(line)
        if pod['node'] in nodes:
            nodes[pod['node']]['used_cpu'] += parse_cpu(pod['cpu_request'])
            nodes[pod['node']]['used_memory'] += parse_memory(pod['memory_request'])

# 効率計算
print("Node Bin Packing Efficiency:")
print("-" * 80)
print(f"{'Node':<20} {'CPU %':<10} {'Memory %':<10} {'Efficiency':<15}")
print("-" * 80)

total_cpu = 0
total_used_cpu = 0
total_memory = 0
total_used_memory = 0

for node_name, data in nodes.items():
    cpu_pct = (data['used_cpu'] / data['cpu']) * 100
    mem_pct = (data['used_memory'] / data['memory']) * 100
    efficiency = min(cpu_pct, mem_pct)  # 制約リソースの効率

    print(f"{node_name:<20} {cpu_pct:>8.1f}% {mem_pct:>9.1f}% {efficiency:>13.1f}%")

    total_cpu += data['cpu']
    total_used_cpu += data['used_cpu']
    total_memory += data['memory']
    total_used_memory += data['used_memory']

print("-" * 80)
cluster_cpu_pct = (total_used_cpu / total_cpu) * 100
cluster_mem_pct = (total_used_memory / total_memory) * 100
cluster_efficiency = min(cluster_cpu_pct, cluster_mem_pct)

print(f"{'CLUSTER TOTAL':<20} {cluster_cpu_pct:>8.1f}% {cluster_mem_pct:>9.1f}% {cluster_efficiency:>13.1f}%")
print()
print("Interpretation:")
print("- High efficiency (>70%): Good resource utilization")
print("- Medium efficiency (40-70%): Acceptable, room for improvement")
print("- Low efficiency (<40%): Poor resource utilization, consider:")
print("  1. Adjusting Pod requests/limits")
print("  2. Using smaller instance types")
print("  3. Consolidating workloads")
EOF

python3 /tmp/binpacking.py
```

---

このドキュメントは、クラスター運用の主要部分をカバーしています。最後に「トラブルシューティング」のドキュメントを作成してPhase 1を完成させます。

## 📚 参考リソース

- [Kubernetes Upgrade Best Practices](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Node Problem Detector](https://github.com/kubernetes/node-problem-detector)
- [Certificate Management](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [Capacity Planning](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)
