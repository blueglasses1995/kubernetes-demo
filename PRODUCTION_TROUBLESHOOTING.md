# Kubernetes本番環境 トラブルシューティング完全ガイド

このドキュメントは、Kubernetes本番環境で発生する一般的な問題のトラブルシューティングを、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [デバッグツール](#1-デバッグツール)
2. [一般的な問題](#2-一般的な問題)
3. [パフォーマンス問題](#3-パフォーマンス問題)
4. [ネットワーク問題](#4-ネットワーク問題)

---

## 1. デバッグツール

### 1.1 kubectl debug

#### Ephemeral Containers を使ったデバッグ

```bash
# 実行中のPodにデバッグコンテナを追加（Kubernetes 1.25+）
kubectl debug -it pod-name --image=busybox --target=container-name

# 例: nginxコンテナのファイルシステムを調査
kubectl debug -it nginx-pod --image=busybox:1.36 --target=nginx

# デバッグコンテナ内で:
/ # ls -la /usr/share/nginx/html/
/ # ps aux
/ # netstat -tunlp

# Node上で直接デバッグ（hostネットワーク、hostPID）
kubectl debug node/node-1 -it --image=ubuntu:22.04

# Node上でroot権限でコマンド実行
root@node-1:/# chroot /host
root@node-1:/# systemctl status kubelet
root@node-1:/# journalctl -u kubelet -n 100
```

#### デバッグ用Podのテンプレート

```yaml
# debug-pod.yaml - 完全装備のデバッグPod
apiVersion: v1
kind: Pod
metadata:
  name: debug-tools
  namespace: default
spec:
  # 特定ノードで実行
  nodeName: node-1

  # ホストネットワークを使用
  hostNetwork: true
  hostPID: true
  hostIPC: true

  containers:
  - name: debug
    image: nicolaka/netshoot:latest  # ネットワークデバッグツール満載
    command: ["/bin/bash"]
    args: ["-c", "sleep infinity"]

    securityContext:
      privileged: true

    volumeMounts:
    # ホストのルートファイルシステムをマウント
    - name: host-root
      mountPath: /host
      readOnly: true

  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory

  # 完了後も残す（手動削除まで）
  restartPolicy: Never
```

```bash
# デバッグPodの使用例
kubectl apply -f debug-pod.yaml
kubectl exec -it debug-tools -- bash

# コンテナ内で利用可能なツール:
# - tcpdump: パケットキャプチャ
# - nmap: ポートスキャン
# - curl, wget: HTTP リクエスト
# - dig, nslookup: DNS クエリ
# - netstat, ss: ネットワーク接続状態
# - iperf3: ネットワーク帯域測定
# - ethtool: NIC設定確認

# 例: 特定サービスへの接続確認
bash-5.1# curl -v http://myapp.prod.svc.cluster.local:8080/health

# 例: DNS解決確認
bash-5.1# dig myapp.prod.svc.cluster.local

# 例: パケットキャプチャ
bash-5.1# tcpdump -i eth0 -nn 'port 8080'
```

### 1.2 kubectl-sniff (パケットキャプチャ)

```bash
# kubectl-sniff のインストール
kubectl krew install sniff

# Pod のネットワークトラフィックをキャプチャ
kubectl sniff pod-name -n namespace -o capture.pcap

# 特定のコンテナをターゲット
kubectl sniff pod-name -c container-name -n namespace

# フィルタを適用
kubectl sniff pod-name -f "port 8080" -n namespace

# Wiresharkで直接開く
kubectl sniff pod-name -n namespace -o - | wireshark -k -i -
```

### 1.3 k9s（TUIツール）

```bash
# k9s のインストール
brew install derailed/k9s/k9s

# 起動
k9s

# 主要なショートカット:
# 0: すべてのNamespaceを表示
# :pods: Pod一覧
# :svc: Service一覧
# :deploy: Deployment一覧
# d: 詳細表示
# l: ログ表示
# e: 編集
# shift+f: ポートフォワード
# ctrl+k: kill（削除）
# /: フィルタ
# ?: ヘルプ

# Podのログをストリーム表示
# 1. :pods でPod一覧
# 2. 対象Podを選択
# 3. l キーでログ表示
# 4. 0-9 キーで前のログを表示
```

---

## 2. 一般的な問題

### 2.1 ImagePullBackOff

#### 原因と診断

```bash
# Podの状態確認
kubectl get pods
# NAME                     READY   STATUS             RESTARTS   AGE
# myapp-7d9f8b4c5d-abc12   0/1     ImagePullBackOff   0          2m

# イベント確認
kubectl describe pod myapp-7d9f8b4c5d-abc12

# 確認すべきイベント:
# Events:
#   Type     Reason     Age                  From               Message
#   ----     ------     ----                 ----               -------
#   Normal   Scheduled  2m                   default-scheduler  Successfully assigned default/myapp-7d9f8b4c5d-abc12 to node-1
#   Normal   Pulling    30s (x4 over 2m)     kubelet            Pulling image "myregistry.io/myapp:v1.2.3"
#   Warning  Failed     29s (x4 over 2m)     kubelet            Failed to pull image "myregistry.io/myapp:v1.2.3": rpc error: code = Unknown desc = Error response from daemon: pull access denied for myregistry.io/myapp, repository does not exist or may require 'docker login'
#   Warning  Failed     29s (x4 over 2m)     kubelet            Error: ErrImagePull
#   Normal   BackOff    15s (x6 over 2m)     kubelet            Back-off pulling image "myregistry.io/myapp:v1.2.3"
#   Warning  Failed     15s (x6 over 2m)     kubelet            Error: ImagePullBackOff
```

#### 原因別の対処法

```yaml
# 原因1: イメージが存在しない
# 対処: イメージ名とタグを確認

# ローカルでイメージを確認
docker pull myregistry.io/myapp:v1.2.3

# タグ一覧を確認（レジストリAPI）
curl -X GET https://myregistry.io/v2/myapp/tags/list

---
# 原因2: プライベートレジストリの認証失敗
# 対処: ImagePullSecret を作成

# Docker credentialsからSecretを作成
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# Podで使用
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myregistry.io/myapp:v1.2.3
  imagePullSecrets:
  - name: regcred  # ← ImagePullSecretを指定

---
# 原因3: レートリミット（Docker Hub）
# 対処: 認証を使用するか、別のレジストリを使用

# Docker Hubの認証情報でSecret作成
kubectl create secret docker-registry dockerhub \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword

---
# 原因4: ネットワーク問題
# 対処: Nodeからレジストリへの接続確認

# Node上で確認
kubectl debug node/node-1 -it --image=curlimages/curl
# curl -v https://myregistry.io/v2/

# プロキシ設定の確認
kubectl get cm -n kube-system kube-proxy -o yaml | grep -i proxy
```

### 2.2 CrashLoopBackOff

#### 診断フロー

```bash
# Pod状態確認
kubectl get pods
# NAME                     READY   STATUS             RESTARTS   AGE
# myapp-7d9f8b4c5d-xyz     0/1     CrashLoopBackOff   5          10m

# ログ確認（現在のコンテナ）
kubectl logs myapp-7d9f8b4c5d-xyz

# 前回クラッシュ時のログ確認
kubectl logs myapp-7d9f8b4c5d-xyz --previous

# リアルタイムログストリーム
kubectl logs -f myapp-7d9f8b4c5d-xyz

# 詳細情報
kubectl describe pod myapp-7d9f8b4c5d-xyz
```

#### 原因別の対処法

```yaml
# 原因1: アプリケーションエラー
# 対処: ログからエラーを特定

# 例: データベース接続エラー
# Logs:
# Error: connect ECONNREFUSED 10.100.1.5:5432
# at TCPConnectWrap.afterConnect [as oncomplete] (net.js:1144:16)

# 確認事項:
# - Serviceが存在するか
# - Endpointが存在するか
# - ネットワークポリシーで許可されているか

kubectl get svc postgres
kubectl get endpoints postgres
kubectl get networkpolicies

---
# 原因2: Liveness Probe の失敗
# 対処: Probeの設定を見直し

apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30  # アプリ起動を待つ（短すぎると即座に失敗）
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3      # 3回連続失敗でコンテナを再起動

# Probe失敗の確認
kubectl describe pod myapp | grep -A 10 "Liveness"
# Liveness:       http-get http://:8080/healthz delay=30s timeout=5s period=10s #success=1 #failure=3
# Warning  Unhealthy  2m (x5 over 3m)  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 500

---
# 原因3: OOMKilled
# 対処: メモリ制限を増やす

# OOMの確認
kubectl describe pod myapp | grep -i oom
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137

# メモリ使用量の確認
kubectl top pod myapp

# メモリ制限を増やす
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        memory: "256Mi"
      limits:
        memory: "1Gi"  # 増やす

---
# 原因4: 設定ミス（環境変数、ConfigMap等）
# 対処: 設定を確認

# Pod内の環境変数確認
kubectl exec myapp -- env

# ConfigMapの内容確認
kubectl get cm myapp-config -o yaml

# Secretの内容確認（base64デコード）
kubectl get secret myapp-secret -o json | jq -r '.data | to_entries[] | .key + ": " + (.value | @base64d)'
```

### 2.3 Pending Pods

#### 診断

```bash
# Pending Pod の確認
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# myapp-7d9f8b4c5d-pending 0/1     Pending   0          5m

# イベント確認
kubectl describe pod myapp-7d9f8b4c5d-pending

# Pendingの一般的な原因:
# 1. リソース不足（CPU/メモリ）
# 2. PVCがバインドされていない
# 3. Node Selectorにマッチするノードがない
# 4. Taints/Tolerations の不一致
# 5. PodDisruptionBudget による制限
```

#### 原因別の対処法

```bash
# 原因1: リソース不足
# Events:
# Warning  FailedScheduling  1m (x10 over 5m)  default-scheduler  0/3 nodes are available: 3 Insufficient cpu.

# ノードのリソース確認
kubectl describe nodes | grep -A 5 "Allocated resources"

# 出力例:
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#   Resource           Requests      Limits
#   --------           --------      ------
#   cpu                7500m (94%)   8000m (100%)
#   memory             24Gi (77%)    30Gi (96%)

# 対処:
# - 不要なPodを削除
# - ノードを追加
# - Pod のリソース要求を減らす

---
# 原因2: PVCがバインドされていない
# Events:
# Warning  FailedScheduling  1m (x10 over 5m)  default-scheduler  persistentvolumeclaim "myapp-data" not found

# PVC状態確認
kubectl get pvc
# NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# myapp-data   Pending   -        -          -              fast-ssd       5m

# PVCの詳細確認
kubectl describe pvc myapp-data

# Events:
# Warning  ProvisioningFailed  1m (x5 over 5m)  persistentvolume-controller  storageclass.storage.k8s.io "fast-ssd" not found

# 対処:
# - StorageClass が存在するか確認
# - PV が利用可能か確認（Static Provisioning の場合）

kubectl get sc
kubectl get pv

---
# 原因3: Node Selector 不一致
# Events:
# Warning  FailedScheduling  1m (x10 over 5m)  default-scheduler  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.

# Pod の nodeSelector 確認
kubectl get pod myapp-7d9f8b4c5d-pending -o yaml | grep -A 3 nodeSelector

# ノードのラベル確認
kubectl get nodes --show-labels

# 対処:
# - nodeSelector を修正
# - ノードに適切なラベルを追加

kubectl label node node-1 disktype=ssd

---
# 原因4: Taints/Tolerations
# Events:
# Warning  FailedScheduling  1m (x10 over 5m)  default-scheduler  0/3 nodes are available: 1 node(s) had taints that the pod didn't tolerate, 2 node(s) had untolerated taint {node-role.kubernetes.io/master: }.

# ノードのTaints確認
kubectl describe node node-1 | grep Taints

# Toleration追加
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
  containers:
  - name: app
    image: myapp:1.0
```

### 2.4 PVC Binding Failures

```bash
# PVC が Pending 状態
kubectl get pvc
# NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# myapp-data   Pending   -        -          -              gp2            10m

# 詳細確認
kubectl describe pvc myapp-data

# 原因1: StorageClass が存在しない
# Events:
# Warning  ProvisioningFailed  1m (x5 over 10m)  persistentvolume-controller  storageclass.storage.k8s.io "gp2" not found

# StorageClass 一覧確認
kubectl get sc

# StorageClass 作成（AWS EBS例）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# 原因2: CSI Driver が動作していない
# CSI Controller Podの確認
kubectl get pods -n kube-system | grep csi

# 出力例:
# ebs-csi-controller-7d9f8b-abc12     6/6     Running   0          30d
# ebs-csi-controller-7d9f8b-def34     6/6     Running   0          30d
# ebs-csi-node-xyz                    3/3     Running   0          30d

# CSI Driverログ確認
kubectl logs -n kube-system ebs-csi-controller-7d9f8b-abc12 -c ebs-plugin

---
# 原因3: クォータ超過（クラウドプロバイダー）
# Events:
# Warning  ProvisioningFailed  1m  persistentvolume-controller  Failed to provision volume with StorageClass "gp2": rpc error: code = ResourceExhausted desc = volume limit exceeded

# AWS の場合、インスタンスタイプごとのEBSボリューム数制限
# https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html

# 対処:
# - 不要なボリュームを削除
# - より大きいインスタンスタイプに変更
```

---

## 3. パフォーマンス問題

### 3.1 High CPU Usage

#### 診断

```bash
# Pod の CPU 使用率確認
kubectl top pods --all-namespaces --sort-by=cpu

# 上位10件
kubectl top pods --all-namespaces --sort-by=cpu | head -n 11

# 特定Podの詳細
kubectl top pod myapp-7d9f8b4c5d-xyz --containers

# 出力:
# POD                       NAME     CPU(cores)   MEMORY(bytes)
# myapp-7d9f8b4c5d-xyz      app      950m         512Mi
# myapp-7d9f8b4c5d-xyz      sidecar  50m          64Mi

# CPU使用率が高い場合の確認事項

# 1. CPU Throttling の確認
kubectl describe pod myapp-7d9f8b4c5d-xyz

# Container内でthrottling状態を確認
kubectl exec myapp-7d9f8b4c5d-xyz -- cat /sys/fs/cgroup/cpu/cpu.stat

# throttled_time が増加している場合、CPU制限に達している
```

#### 対処法

```yaml
# 1. CPU Limits を増やす
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        resources:
          requests:
            cpu: 500m
          limits:
            cpu: 2000m  # 増やす

---
# 2. Horizontal Pod Autoscaler (HPA) を設定
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # CPU 70%でスケール

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60

---
# 3. プロファイリングツールを使用

# Go アプリの場合: pprof
# アプリケーションに pprof エンドポイントを追加
import _ "net/http/pprof"

# ポートフォワード
kubectl port-forward pod/myapp-7d9f8b4c5d-xyz 6060:6060

# CPU プロファイル取得
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Python アプリの場合: py-spy
kubectl exec -it myapp-7d9f8b4c5d-xyz -- py-spy top --pid 1
```

### 3.2 High Memory Usage / OOMKilled

#### 診断

```bash
# OOM で終了したPodの確認
kubectl get pods --all-namespaces --field-selector status.phase=Failed

# Pod詳細確認
kubectl describe pod myapp-7d9f8b4c5d-oom

# Last State:
#   Terminated:
#     Reason:       OOMKilled
#     Exit Code:    137
#     Started:      Mon, 15 Jan 2024 10:00:00 +0900
#     Finished:     Mon, 15 Jan 2024 10:05:23 +0900

# メモリ使用量の確認
kubectl top pod myapp-7d9f8b4c5d-xyz

# Nodeのメモリプレッシャー確認
kubectl describe node node-1 | grep -A 5 "MemoryPressure"

# Container内のメモリ統計
kubectl exec myapp-7d9f8b4c5d-xyz -- cat /sys/fs/cgroup/memory/memory.stat
```

#### メモリリーク調査

```bash
# Go アプリの場合
kubectl port-forward pod/myapp-7d9f8b4c5d-xyz 6060:6060

# ヒーププロファイル取得
go tool pprof http://localhost:6060/debug/pprof/heap

# インタラクティブモードで分析
(pprof) top10
(pprof) list <function-name>

# Java アプリの場合
kubectl exec -it myapp-7d9f8b4c5d-xyz -- jmap -heap 1

# ヒープダンプ取得
kubectl exec -it myapp-7d9f8b4c5d-xyz -- jmap -dump:format=b,file=/tmp/heap.bin 1

# ヒープダンプをローカルにコピー
kubectl cp myapp-7d9f8b4c5d-xyz:/tmp/heap.bin ./heap.bin

# Eclipse MAT 等で分析
```

#### 対処法

```yaml
# 1. メモリ制限を増やす
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        resources:
          requests:
            memory: "512Mi"
          limits:
            memory: "2Gi"  # 増やす

---
# 2. Vertical Pod Autoscaler (VPA) を使用
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"  # 自動的にリソース要求を調整
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        memory: "256Mi"
      maxAllowed:
        memory: "4Gi"

---
# 3. メモリリミットなし（requestsのみ）
# OOMよりも、Nodeのメモリ不足でevictionされる方がマシな場合
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        resources:
          requests:
            memory: "512Mi"
          # limits を設定しない
```

### 3.3 Disk I/O Bottlenecks

```bash
# Node の I/O 確認
kubectl debug node/node-1 -it --image=ubuntu:22.04
chroot /host

# iostat でディスクI/O確認
apt-get update && apt-get install -y sysstat
iostat -x 1

# 出力:
# Device   r/s    w/s    rkB/s    wkB/s  %util
# nvme0n1  120.0  450.0  4800.0   18000.0  98.5  ← 高負荷

# どのプロセスがI/Oを使用しているか
iotop -o

# Pod内でのI/O確認
kubectl exec -it myapp-7d9f8b4c5d-xyz -- sh

# ファイルシステムのI/O統計
cat /proc/self/io
# rchar: 1234567890      # 読み取りバイト数
# wchar: 9876543210      # 書き込みバイト数
# syscr: 123456          # read() システムコール回数
# syscw: 654321          # write() システムコール回数
```

#### 対処法

```yaml
# 1. より高速なストレージクラスを使用
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  storageClassName: fast-ssd  # io1, gp3 等の高速ストレージ
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi

---
# AWS EBS io2 の例
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iopsPerGB: "50"     # 50 IOPS/GB
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# 2. Local PersistentVolume を使用（最高速）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-1
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1

---
# 3. キャッシュを活用
# アプリケーション層でのキャッシング（Redis等）
# ファイルシステムキャッシュを意識した実装
```

---

## 4. ネットワーク問題

### 4.1 DNS Resolution Failures

#### 診断

```bash
# Pod内からDNS解決テスト
kubectl exec -it myapp-7d9f8b4c5d-xyz -- nslookup kubernetes.default

# 失敗する場合:
# Server:         10.96.0.10
# Address:        10.96.0.10:53
#
# ** server can't find kubernetes.default: NXDOMAIN

# CoreDNS Pod の状態確認
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS ログ確認
kubectl logs -n kube-system -l k8s-app=kube-dns

# DNS設定確認
kubectl exec -it myapp-7d9f8b4c5d-xyz -- cat /etc/resolv.conf

# 出力:
# search default.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
# options ndots:5
```

#### 対処法

```yaml
# 1. CoreDNS ConfigMap 確認
kubectl get cm -n kube-system coredns -o yaml

# 例: カスタムドメインの追加
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    # カスタムドメイン
    example.com:53 {
        errors
        cache 30
        forward . 8.8.8.8 8.8.4.4
    }

---
# 2. NodeLocal DNSCache を導入（パフォーマンス改善）
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

# NodeLocal DNSCache DaemonSet の確認
kubectl get ds -n kube-system node-local-dns

---
# 3. Pod の DNSConfig を カスタマイズ
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
    - 10.96.0.10       # CoreDNS
    - 8.8.8.8          # Google DNS (fallback)
    searches:
    - default.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:
    - name: ndots
      value: "2"
    - name: edns0
```

### 4.2 Service Connectivity Issues

```bash
# Service の確認
kubectl get svc myapp

# Endpoint の確認（実際のPod IPが登録されているか）
kubectl get endpoints myapp

# 出力例:
# NAME    ENDPOINTS                                               AGE
# myapp   10.244.1.10:8080,10.244.2.15:8080,10.244.3.20:8080     10d

# Endpoint が空の場合、Serviceのselectorを確認
kubectl get svc myapp -o yaml | grep -A 5 selector
kubectl get pods -l app=myapp  # selectorにマッチするPodが存在するか

# Service経由での接続テスト
kubectl run test-pod --image=curlimages/curl -it --rm -- curl http://myapp.default.svc.cluster.local:8080

# 直接PodIPへの接続テスト
kubectl run test-pod --image=curlimages/curl -it --rm -- curl http://10.244.1.10:8080

# Serviceは疎通するがPod IPは疎通しない → kube-proxy の問題
# Pod IPは疎通するがServiceは疎通しない → Service定義の問題
```

#### kube-proxy のトラブルシューティング

```bash
# kube-proxy Podの確認
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# kube-proxy ログ確認
kubectl logs -n kube-system -l k8s-app=kube-proxy

# kube-proxy の mode 確認
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"
# 出力: Using iptables Proxy

# iptables ルールの確認（Node上）
kubectl debug node/node-1 -it --image=ubuntu:22.04
chroot /host

# Service の iptables ルール確認
iptables-save | grep myapp

# 出力例:
# -A KUBE-SERVICES -d 10.96.100.200/32 -p tcp -m tcp --dport 8080 -j KUBE-SVC-XXXXXX
# -A KUBE-SVC-XXXXXX -m statistic --mode random --probability 0.33333 -j KUBE-SEP-AAAA
# -A KUBE-SVC-XXXXXX -m statistic --mode random --probability 0.50000 -j KUBE-SEP-BBBB
# -A KUBE-SVC-XXXXXX -j KUBE-SEP-CCCC
```

### 4.3 Network Policy Issues

```bash
# Network Policy の確認
kubectl get networkpolicies -n prod

# 特定Podに適用されているNetwork Policyを確認
kubectl describe pod myapp-7d9f8b4c5d-xyz | grep -A 10 "Labels"

# Network Policy の詳細
kubectl describe networkpolicy allow-frontend -n prod

# Network Policyのテスト
# 送信元Pod
kubectl run test-source --image=curlimages/curl -n prod -- sleep 3600

# 宛先Podへの接続テスト
kubectl exec -n prod test-source -- curl -m 5 http://myapp:8080

# タイムアウトする場合、Network Policyで拒否されている可能性
```

#### Network Policy デバッグ例

```yaml
# 問題: frontendからbackendに接続できない

# 現在のNetwork Policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
          tier: web  # ← 問題: frontendにtier=webラベルがない
    ports:
    - protocol: TCP
      port: 8080

# frontendのラベル確認
kubectl get pod -n prod -l app=frontend --show-labels
# NAME                        READY   STATUS    LABELS
# frontend-7d9f8b4c5d-xyz     1/1     Running   app=frontend

# 修正案1: tier=web ラベルを追加
kubectl label pod -n prod -l app=frontend tier=web

# 修正案2: Network Policyを修正（tierラベルを削除）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # tier: webを削除
    ports:
    - protocol: TCP
      port: 8080
```

### 4.4 Ingress Issues

```bash
# Ingress の確認
kubectl get ingress -n prod

# Ingress 詳細
kubectl describe ingress myapp-ingress -n prod

# Events:
#   Type    Reason  Age   From                      Message
#   ----    ------  ----  ----                      -------
#   Normal  Sync    1m    nginx-ingress-controller  Scheduled for sync

# Ingress Controller Pod の確認
kubectl get pods -n ingress-nginx

# Ingress Controller ログ
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Backend Service の疎通確認
kubectl run test --image=curlimages/curl -it --rm -- curl http://myapp.prod.svc.cluster.local:8080

# TLS 証明書の確認
kubectl get secret -n prod myapp-tls -o yaml

# 証明書の有効期限確認
kubectl get secret -n prod myapp-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

---

このドキュメントは、Kubernetes本番環境でのトラブルシューティングの主要な問題とその解決方法をカバーしています。

## 📚 参考リソース

- [Kubernetes Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/)
- [kubectl debug Documentation](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#debug)
- [Troubleshooting Applications](https://kubernetes.io/docs/tasks/debug/debug-application/)
- [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)
