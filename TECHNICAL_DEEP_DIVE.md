# Kubernetes技術スタックの全体像と原理

このドキュメントでは、カリキュラムで学ぶすべての技術について、**レイヤー構造と技術的原理**を詳細に解説します。

## 📚 目次

1. [レイヤー構造の全体像](#1-レイヤー構造の全体像)
2. [各レイヤーの詳細説明](#2-各レイヤーの詳細説明)
3. [カリキュラム技術のレイヤーマッピング](#3-カリキュラム技術のレイヤーマッピング)
4. [詳細な技術原理](#4-詳細な技術原理)

---

## 1. レイヤー構造の全体像

Kubernetesエコシステムは、以下の7つのレイヤーで構成されています：

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 7: プラットフォーム抽象化層                                │
│  - Render, Fly.io                                            │
│  (Kubernetesを完全に隠蔽したPaaS)                              │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 6: 高度なオーケストレーション層                            │
│  - Karpenter (ノード管理), Crossplane (インフラ管理)            │
│  (Kubernetesの自動化・拡張)                                    │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 5: サービスメッシュ層                                     │
│  - Linkerd, Istio                                            │
│  (マイクロサービス間通信の制御・観測)                             │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 4: アプリケーション管理層                                 │
│  - Helm (パッケージ管理), Lens (可視化)                         │
│  (Kubernetesリソースの抽象化・管理)                             │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Kubernetesオーケストレーション層                       │
│  - kube-apiserver, kube-scheduler, kube-controller-manager  │
│  - etcd, kubelet, kube-proxy                                │
│  (コンテナのスケジューリング・管理)                              │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: コンテナランタイム層                                   │
│  - containerd, CRI-O                                        │
│  - runc (OCI Runtime)                                       │
│  (コンテナの実行環境)                                          │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: OS・カーネル層                                        │
│  - Linux Kernel (namespace, cgroup, ネットワークスタック)       │
│  - ファイルシステム、ストレージドライバー                         │
│  (リソース分離・制限の基盤)                                     │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Layer 0: ハードウェア層                                        │
│  - CPU (x86_64, ARM64)                                      │
│  - メモリ (RAM)                                              │
│  - ストレージ (SSD, HDD)                                     │
│  - ネットワークインターフェース (NIC)                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 各レイヤーの詳細説明

### Layer 0: ハードウェア層

#### 2.1 CPU（中央処理装置）

**役割**: すべての計算処理の実行

**詳細な動作**:
1. **命令実行サイクル**:
   - フェッチ: メモリから命令を取得
   - デコード: 命令を解釈
   - 実行: ALU（算術論理演算ユニット）で処理
   - ライトバック: 結果をレジスタ/メモリに書き込み

2. **マルチコア処理**:
   - 各コアは独立したスレッドを実行
   - Kubernetesは複数のコアに作業を分散
   - CPUアフィニティでPodを特定コアに固定可能

3. **仮想化支援機能**:
   - Intel VT-x / AMD-V: ハードウェア仮想化
   - Extended Page Tables (EPT): メモリアドレス変換の高速化
   - これによりコンテナのオーバーヘッドが最小化

**Kubernetesとの関係**:
- CPU requests/limits: この層のリソースを予約・制限
- CPUSet: 特定のコアをPodに割り当て
- CPU throttling: cgroup経由でCPU使用を制限

---

#### 2.2 メモリ（RAM）

**役割**: データとプログラムの一時保存

**詳細な動作**:
1. **物理メモリ管理**:
   - DRAMチップへのアドレッシング
   - メモリコントローラーによるアクセス制御
   - メモリバンク間のインターリーブ

2. **メモリ階層**:
   ```
   CPU レジスタ (数十バイト, <1ns)
        ↓
   L1 キャッシュ (32-64KB, ~1ns)
        ↓
   L2 キャッシュ (256KB-1MB, ~4ns)
        ↓
   L3 キャッシュ (数MB-数十MB, ~10ns)
        ↓
   メインメモリ (数GB-数TB, ~100ns)
   ```

3. **NUMA（Non-Uniform Memory Access）**:
   - CPUソケットごとに専用メモリ
   - 他ソケットのメモリアクセスは遅延大
   - Kubernetesの`topologyManager`がNUMAを考慮

**Kubernetesとの関係**:
- Memory requests/limits: 物理メモリの予約・制限
- OOM Killer: メモリ不足時にプロセスを強制終了
- Huge Pages: 大容量メモリページで性能向上

---

#### 2.3 ストレージ

**役割**: データの永続化

**詳細な動作**:
1. **SSD（Solid State Drive）**:
   - NAND フラッシュメモリセル
   - FTL（Flash Translation Layer）でアドレス変換
   - Wear Leveling で寿命延長
   - TRIM コマンドで削除データ管理

2. **ストレージスタック**:
   ```
   アプリケーション
        ↓
   ファイルシステム (ext4, xfs, btrfs)
        ↓
   Volume Manager (LVM)
        ↓
   Block Layer (bio)
        ↓
   I/O Scheduler (mq-deadline, kyber)
        ↓
   デバイスドライバー (nvme, scsi)
        ↓
   物理デバイス (SSD, HDD)
   ```

3. **I/O パターン**:
   - Sequential I/O: 連続アクセス（高速）
   - Random I/O: ランダムアクセス（低速、特にHDD）
   - Direct I/O: ページキャッシュバイパス

**Kubernetesとの関係**:
- PersistentVolume: この層のストレージを抽象化
- StorageClass: ストレージの種類を定義（SSD, HDD等）
- CSI Driver: ストレージプロバイダーと接続

---

#### 2.4 ネットワークインターフェース（NIC）

**役割**: ネットワーク通信の物理層

**詳細な動作**:
1. **パケット送受信**:
   - DMA（Direct Memory Access）でCPU負荷軽減
   - Interrupt Coalescing で割り込み削減
   - RSS（Receive Side Scaling）で複数CPUに分散

2. **オフロード機能**:
   - TSO（TCP Segmentation Offload）: TCP分割をNICで実行
   - Checksum Offload: チェックサム計算をNICで実行
   - VXLAN Offload: オーバーレイネットワークの処理をNICで実行

3. **SR-IOV（Single Root I/O Virtualization）**:
   - 物理NICを複数の仮想NICに分割
   - 各Podに仮想NICを直接割り当て
   - ネットワーク性能が大幅に向上

**Kubernetesとの関係**:
- CNI Plugin: NICを仮想ネットワークに接続
- NetworkPolicy: パケットフィルタリング
- Bandwidth limits: NICの帯域制限

---

### Layer 1: OS・カーネル層

#### 2.5 Linux Kernel

**役割**: ハードウェアの抽象化とリソース管理

**詳細な動作**:

##### 2.5.1 Namespace（名前空間分離）

Namespaceは、プロセスに独立した環境を提供する機構です。

**1. PID Namespace**:
```
ホストOS: PID 1 (systemd)
           ├─ PID 100 (dockerd)
           └─ PID 200 (containerd)

コンテナ内: PID 1 (アプリケーション)  ← 実際のPIDは300だが、コンテナ内では1に見える
            └─ PID 2 (子プロセス)    ← 実際のPIDは301だが、コンテナ内では2に見える
```

**カーネル内部の動作**:
1. `clone(CLONE_NEWPID)` システムコールで新しいPID namespaceを作成
2. カーネルは`task_struct`（プロセス管理構造体）に namespace ポインタを保持
3. `getpid()` 呼び出し時、カーネルは現在のnamespaceを確認し、namespace内のPIDを返す
4. `/proc` ファイルシステムもnamespaceごとに異なる内容を表示

**2. Network Namespace**:
```
ホストOS:
  eth0: 192.168.1.100
  lo: 127.0.0.1

コンテナ1:
  eth0: 10.244.1.5    ← veth pair でホストと接続
  lo: 127.0.0.1

コンテナ2:
  eth0: 10.244.1.6    ← 別のveth pair でホストと接続
  lo: 127.0.0.1
```

**カーネル内部の動作**:
1. `clone(CLONE_NEWNET)` で新しいNetwork namespaceを作成
2. 各namespaceは独立した：
   - ネットワークデバイス一覧
   - ルーティングテーブル
   - iptables ルール
   - ソケット一覧
3. veth pair（仮想Ethernetペア）でnamespace間を接続：
   ```
   [コンテナ namespace]          [ホスト namespace]
          eth0 ←─────veth pair─────→ vethXXX
                                        ↓
                                     bridge (cni0)
   ```

**3. Mount Namespace**:
各コンテナは独立したファイルシステムツリーを持ちます。

**カーネル内部の動作**:
1. `clone(CLONE_NEWNS)` で新しいMount namespaceを作成
2. カーネルは各namespaceごとに`vfsmount`構造体のツリーを管理
3. OverlayFS で効率的なレイヤー管理：
   ```
   [コンテナから見えるファイルシステム]
              ↓
         Overlay FS
              ↓
   ┌──────────┴──────────┐
   │                     │
   Upper Layer        Lower Layers
   (書き込み可能)      (読み取り専用)
   /var/lib/docker     /var/lib/docker
   /overlay2/xxx       /overlay2/base-image
   ```

**4. UTS Namespace**:
ホスト名とドメイン名を分離

**5. IPC Namespace**:
System V IPC、POSIXメッセージキューを分離

**6. User Namespace**:
UID/GIDマッピングを提供：
```
コンテナ内: UID 0 (root) → ホスト: UID 1000 (一般ユーザー)
```

---

##### 2.5.2 Cgroup（Control Groups）

Cgroupは、プロセスグループのリソース使用を制限・測定する機構です。

**Cgroup v2 の階層構造**:
```
/sys/fs/cgroup/
├── cgroup.controllers       # 利用可能なコントローラー一覧
├── cgroup.procs             # このcgroupのプロセス一覧
├── cpu.max                  # CPU制限
├── memory.max               # メモリ制限
└── kubepods.slice/          # Kubernetes管理下のPod
    ├── pod-abc123/
    │   ├── container1/
    │   │   ├── cgroup.procs
    │   │   ├── cpu.max      # "100000 100000" = 1 CPU
    │   │   └── memory.max   # "536870912" = 512Mi
    │   └── container2/
    └── pod-def456/
```

**CPU コントローラー**:

1. **CPU Shares（CPU時間の相対的な割り当て）**:
   ```
   Container A: cpu.weight = 1024
   Container B: cpu.weight = 512

   競合時: A は B の2倍のCPU時間を取得
   ```

2. **CPU Quota（絶対的な制限）**:
   ```
   cpu.max = "50000 100000"
            ↑       ↑
            quota   period

   意味: 100msごとに最大50msのCPU時間
   = 0.5 CPU コア相当
   ```

**カーネル内部の動作**:
1. スケジューラーがプロセスを選択する際、cgroupのquotaをチェック
2. quota超過の場合、プロセスを`TASK_INTERRUPTIBLE`状態にして待機
3. 次のperiodが始まると、quotaがリセットされプロセスが再開

**Memory コントローラー**:

1. **メモリ制限**:
   ```
   memory.max = 536870912  # 512Mi
   memory.high = 469762048 # 448Mi (警告閾値)
   ```

2. **メモリ使用量の追跡**:
   カーネルは以下を個別にカウント：
   - Anonymous memory（ヒープ、スタック）
   - Page cache（ファイルキャッシュ）
   - Kernel memory（カーネルバッファー）

3. **OOM（Out of Memory）処理**:
   ```
   [メモリ割り当て要求]
         ↓
   [Cgroupのmemory.maxをチェック]
         ↓
   [超過している？]
      Yes → OOM Killer 起動
              ↓
         OOM Score 計算
              ↓
         最もスコアの高いプロセスをkill
   ```

**OOM Score の計算**:
```c
oom_score = (使用メモリ / 総メモリ) × 1000
          + oom_score_adj

// Kubernetesは以下のように設定:
// - Guaranteed Pod: oom_score_adj = -998 (killされにくい)
// - Burstable Pod:  oom_score_adj = min(max(2, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999)
// - BestEffort Pod: oom_score_adj = 1000 (最優先でkill)
```

---

##### 2.5.3 Linuxネットワークスタック

**パケット受信の詳細な流れ**:

```
1. [NIC] パケット到着
     ↓
2. [DMA] パケットをRing Bufferにコピー
     ↓
3. [Interrupt] NICが割り込み発行
     ↓
4. [Soft IRQ] ソフトウェア割り込みハンドラー起動
     ↓
5. [netif_receive_skb()] sk_buff構造体を作成
     ↓
6. [iptables/netfilter] パケットフィルタリング
     ├─ PREROUTING chain
     ├─ FORWARD chain (ルーティング対象)
     └─ INPUT chain (ローカル宛)
     ↓
7. [IP層] IPアドレス確認、ルーティング判定
     ↓
8. [Transport層] TCP/UDPヘッダー解析
     ↓
9. [Socket] 該当ソケットのバッファーにデータをコピー
     ↓
10. [Application] recv() システムコールでデータ取得
```

**Kubernetesのネットワーク関連機能がこの層をどう使うか**:

1. **kube-proxy（iptablesモード）**:
   ```
   # Serviceへのトラフィックを振り分けるiptablesルール
   iptables -t nat -A PREROUTING \
     -d 10.96.0.1 -p tcp --dport 80 \
     -j DNAT --to-destination 10.244.1.5:8080
   ```

   カーネル内部の動作:
   - パケットがPREROUTING chainに到達
   - netfilterがルールを評価
   - マッチしたらDestination NATを実行
   - ルーティング判定を再実行

2. **CNI Plugin（Calico）**:
   - BPF（Berkeley Packet Filter）を使用
   - カーネル内でパケット処理を高速化
   - XDP（eXpress Data Path）でNICレベルでの処理も可能

---

##### 2.5.4 ファイルシステム

**VFS（Virtual File System）レイヤー**:

```
[アプリケーション]
      open(), read(), write()
           ↓
    [VFS レイヤー]
      inode, dentry キャッシュ
           ↓
  ┌────────┴────────┐
  │                 │
[ext4]          [xfs]           [OverlayFS]
  │                 │                │
  │                 │       ┌────────┴────────┐
  │                 │       │                 │
  │                 │   [Upper Dir]      [Lower Dirs]
  │                 │
  ↓                 ↓                 ↓
[Block Layer]
      bio (block I/O)
           ↓
   [I/O Scheduler]
      mq-deadline, kyber
           ↓
   [Device Driver]
      nvme, scsi
           ↓
   [Physical Device]
```

**OverlayFS（コンテナで使用）の動作**:

1. **レイヤー構造**:
   ```
   Container Image:
   [Layer 3: アプリケーション]  ← upperdir (読み書き可能)
   [Layer 2: ライブラリ]        ↓
   [Layer 1: OS基本]            ↓ lowerdir (読み取り専用)
   [Layer 0: ベース]            ↓
   ```

2. **Copy-on-Write**:
   ```
   [読み取り]
   アプリが /etc/config を読む
        ↓
   OverlayFS が lowerdir から読み取り

   [書き込み]
   アプリが /etc/config を変更
        ↓
   OverlayFS が lowerdir から upperdir にファイルをコピー
        ↓
   upperdir で変更を実施
        ↓
   以降の読み取りは upperdir から
   ```

3. **カーネル内部の動作**:
   - VFSのinode操作をOverlayFSがフック
   - `ovl_lookup()`: ファイル検索時、上位レイヤーから順に探索
   - `ovl_copy_up()`: 書き込み時にlowerdirからupperdirにコピー

---

### Layer 2: コンテナランタイム層

#### 2.6 containerd / CRI-O

**役割**: コンテナのライフサイクル管理

**アーキテクチャ（containerdの例）**:
```
[kubelet]
    ↓ CRI (Container Runtime Interface)
[containerd]
    ├─ [containerd-shim] ← コンテナプロセスの親
    │       ↓
    │    [runc] ← 実際のコンテナ起動
    │       ↓
    │   [Container Process]
    │
    ├─ [Image Service]
    │   - イメージの pull, push
    │   - レイヤーのキャッシュ管理
    │
    ├─ [Snapshot Service]
    │   - OverlayFS マウント管理
    │   - Copy-on-Write レイヤー
    │
    └─ [Task Service]
        - コンテナプロセスの監視
        - ログ収集
```

**詳細な動作フロー（コンテナ起動）**:

1. **kubeletからのリクエスト**:
   ```
   kubelet → (gRPC/CRI) → containerd
   Request: RunPodSandbox
   {
     "metadata": {"name": "nginx-pod"},
     "linux": {
       "cgroup_parent": "/kubepods/besteffort/pod-abc123"
     }
   }
   ```

2. **Pause コンテナの作成**:
   - Kubernetesは各Podに「Pause コンテナ」を最初に作成
   - Pause コンテナの役割:
     ```
     - Network Namespace の保持
     - IPC Namespace の保持
     - PID Namespace の保持
     ```
   - 実装は単純な無限ループ:
     ```c
     int main() {
       for (;;) pause();  // シグナル待機
     }
     ```

3. **イメージの取得**:
   ```
   containerd
     ↓
   [OCI Distribution Spec]
     ↓
   Container Registry (docker.io)
     ↓
   イメージマニフェスト取得
     ↓
   レイヤー (tarball) をダウンロード
     ↓
   /var/lib/containerd/io.containerd.content.v1.content/
     ↓
   SHA256でコンテンツアドレス指定
   ```

4. **スナップショットの準備**:
   ```
   Snapshot Service:

   1. ベースイメージのレイヤーをマウント (lowerdir)
   2. 新しい書き込み可能レイヤーを作成 (upperdir)
   3. OverlayFS でマウント:
      mount -t overlay overlay \
        -o lowerdir=/lower1:/lower2,\
           upperdir=/upper,\
           workdir=/work \
        /merged
   ```

5. **runc の実行**:
   ```json
   // config.json (OCI Runtime Spec)
   {
     "ociVersion": "1.0.0",
     "process": {
       "terminal": false,
       "user": {"uid": 0, "gid": 0},
       "args": ["nginx", "-g", "daemon off;"],
       "env": ["PATH=/usr/local/sbin:/usr/local/bin"],
       "cwd": "/"
     },
     "root": {
       "path": "/var/lib/containerd/snapshots/123/fs"
     },
     "linux": {
       "namespaces": [
         {"type": "pid"},
         {"type": "network", "path": "/var/run/netns/cni-xxx"},
         {"type": "mount"},
         {"type": "uts"}
       ],
       "cgroupsPath": "/kubepods/besteffort/pod-abc123/container-id"
     }
   }
   ```

6. **runc の内部動作**:
   ```
   1. clone() システムコール with CLONE_NEW* フラグ
      → 新しい namespace を作成

   2. unshare() で現在のプロセスを namespace から分離

   3. setns() で既存の namespace (network) に参加

   4. chroot() または pivot_root() でルートディレクトリ変更

   5. setrlimit() でリソース制限

   6. cgroup にプロセスを追加:
      echo $$ > /sys/fs/cgroup/kubepods.slice/.../cgroup.procs

   7. execve() でコンテナプロセスを実行
   ```

7. **containerd-shim の役割**:
   - runc は起動後すぐに終了
   - shim がコンテナプロセスの親プロセスとして残る
   - 役割:
     ```
     - stdout/stderr をファイルにリダイレクト
     - 終了コードの記録
     - TTY のリサイズ処理
     - containerd が再起動してもコンテナは継続稼働
     ```

---

### Layer 3: Kubernetesオーケストレーション層

#### 2.7 Kubernetesコントロールプレーン

**全体アーキテクチャ**:
```
[kubectl]
    ↓ (HTTPS)
[kube-apiserver] ←────┐
    ↓                 │
[etcd]                │ (watch)
    ↑                 │
    └─────────────────┤
                      │
[kube-scheduler] ─────┤
[kube-controller-manager] ─┤
                           │
┌──────────────────────────┘
│
[kubelet] (各ノード)
    ↓
[Container Runtime]
```

---

##### 2.7.1 kube-apiserver

**役割**: Kubernetes APIのフロントエンド、すべての操作の入り口

**詳細な動作フロー（Pod作成リクエスト）**:

```
1. [認証] Authentication
   kubectl → TLS Client Certificate
        ↓
   apiserver: X.509証明書を検証
        ↓
   User/ServiceAccount を特定

2. [認可] Authorization
   RBAC (Role-Based Access Control):

   - Role/ClusterRole をチェック:
     rules:
     - apiGroups: [""]
       resources: ["pods"]
       verbs: ["create"]

   - RoleBinding でユーザーと紐付け確認

3. [Admission Control]
   複数の Admission Controller が順次実行:

   a) MutatingAdmissionWebhook:
      - PodSecurityPolicy
      - ServiceAccount注入
      - デフォルト値の設定

   b) ValidatingAdmissionWebhook:
      - ResourceQuota チェック
      - PodSecurity Standards
      - カスタム検証ロジック

4. [Schema Validation]
   OpenAPI Schema で YAML/JSON を検証

5. [etcd への書き込み]
   apiserver → etcd (v3 API)

   Key: /registry/pods/default/nginx-pod
   Value: Protobuf エンコードされた Pod オブジェクト

   etcd内部:
   - Raft コンセンサスアルゴリズムで複製
   - Write-Ahead Log (WAL) で永続化
   - B+Tree インデックスで高速検索

6. [Watch 通知]
   apiserver は etcd を watch
        ↓
   変更を検知
        ↓
   全ての watcher に通知
   (scheduler, controller-manager, kubelet等)
```

**Watch メカニズムの詳細**:
```
[Client] (kubectl get pods --watch)
    ↓
[HTTP/1.1 または HTTP/2 ストリーム]
    ↓
[kube-apiserver]
    - etcd を watch
    - 変更イベントをフィルタリング
    - JSON/Protobuf でストリーミング
    ↓
[Client]
    - ADDED, MODIFIED, DELETED イベントを受信
```

---

##### 2.7.2 etcd

**役割**: Kubernetesの全ての状態を保存する分散KVS

**内部アーキテクチャ**:
```
[etcd クラスター]
   Node 1 (Leader)    Node 2 (Follower)    Node 3 (Follower)
       ↓                    ↓                    ↓
   [Raft State Machine]
       ↓
   [B+Tree (in-memory)]
       ↓
   [WAL (Write-Ahead Log)]
       ↓
   [ディスク]
```

**Raft コンセンサスアルゴリズム**:

1. **Leader Election**:
   ```
   起動時 or Leader故障時:

   1. 各ノードはランダムタイムアウト後に Candidate になる
   2. 他のノードに RequestVote RPC を送信
   3. 過半数の票を獲得したらLeaderに昇格
   4. Leader は定期的に Heartbeat を送信
   ```

2. **ログレプリケーション**:
   ```
   [Client] → PUT /registry/pods/default/nginx
        ↓
   [Leader]
     1. ログエントリを追加 (uncommitted)
     2. Follower に AppendEntries RPC
        ↓
   [Follower]
     3. ログエントリを追加
     4. ACK を返す
        ↓
   [Leader]
     5. 過半数のACKを受信
     6. エントリを committed に変更
     7. State Machine に適用
     8. Client に成功を返す
   ```

3. **データ構造**:
   ```
   WAL (Write-Ahead Log):
   [Entry 1] [Entry 2] [Entry 3] ... [Entry N]

   各エントリ:
   {
     "term": 5,
     "index": 123,
     "type": "PUT",
     "key": "/registry/pods/default/nginx",
     "value": <Protobuf データ>
   }

   B+Tree (メモリ内):
   /registry
     ├─ pods
     │   ├─ default
     │   │   ├─ nginx → <データへのポインタ>
     │   │   └─ redis → <データへのポインタ>
     │   └─ kube-system
     └─ services
   ```

4. **スナップショット**:
   - WALが大きくなりすぎないよう、定期的にスナップショット作成
   - 現在の状態をディスクに保存
   - 古いWALエントリを削除

---

##### 2.7.3 kube-scheduler

**役割**: PodをどのNodeに配置するかを決定

**スケジューリングアルゴリズム**:

```
[Podがスケジュール待ちになる]
    ↓
[Scheduling Queue]
  - Active Queue: 即座にスケジュール可能
  - Backoff Queue: 前回失敗したPod
  - Unschedulable Queue: スケジュール不可能なPod
    ↓
[Scheduling Cycle]
    ↓
┌─────────────────────┐
│ 1. Filtering Phase  │  利用可能なNodeを絞り込み
└─────────────────────┘
    ↓
  Filter Plugins:

  a) NodeResourcesFit:
     Pod の requests とNodeの allocatable を比較
     ```
     Node Allocatable CPU: 4 cores
     既存Pod使用量:         2 cores
     新Pod要求:             1.5 cores
     → OK (4 - 2 = 2 > 1.5)
     ```

  b) NodeAffinity:
     Pod の nodeSelector/nodeAffinity をチェック
     ```yaml
     nodeSelector:
       disktype: ssd
     ```

  c) PodTopologySpread:
     Podを複数のゾーン/ノードに分散
     ```yaml
     topologySpreadConstraints:
     - maxSkew: 1
       topologyKey: zone
       whenUnsatisfiable: DoNotSchedule
     ```

  d) TaintToleration:
     Node の taint をPodが許容するか
     ```
     Node taint: key=value:NoSchedule
     Pod toleration: key=value
     → OK
     ```
    ↓
┌─────────────────────┐
│ 2. Scoring Phase    │  最適なNodeをスコアリング
└─────────────────────┘
    ↓
  Score Plugins:

  a) NodeResourcesBalancedAllocation:
     CPU・メモリの使用率がバランスしているNodeを優先
     Score = 10 - abs(cpuFraction - memoryFraction) * 10

  b) ImageLocality:
     イメージが既にキャッシュされているNodeを優先
     Score = sum(イメージサイズ) / 最大サイズ * 10

  c) InterPodAffinity:
     指定されたPodと同じNodeを優先/回避
     ```yaml
     podAffinity:
       requiredDuringSchedulingIgnoredDuringExecution:
       - labelSelector:
           matchExpressions:
           - key: app
             operator: In
             values: [cache]
         topologyKey: kubernetes.io/hostname
     ```

  各Pluginのスコアに重み付けして合計:
  ```
  Node A: 85点
  Node B: 92点 ← 選択
  Node C: 78点
  ```
    ↓
┌─────────────────────┐
│ 3. Reserve          │  Nodeのリソースを仮予約
└─────────────────────┘
    ↓
┌─────────────────────┐
│ 4. Permit           │  最終承認
└─────────────────────┘
    ↓
[Binding Cycle]
    ↓
┌─────────────────────┐
│ 5. PreBind          │  Volumeのマウント等
└─────────────────────┘
    ↓
┌─────────────────────┐
│ 6. Bind             │  Pod.spec.nodeName を設定
└─────────────────────┘
    ↓
[etcd に書き込み]
    ↓
[kubelet が watch で検知]
```

---

##### 2.7.4 kube-controller-manager

**役割**: 望ましい状態（Desired State）と現在の状態（Current State）を一致させる

**主要なController**:

**1. ReplicaSet Controller**:

```
[Reconciliation Loop]
  ↓
1. List all ReplicaSets
   (informer cache から取得、etcd watchで最新化)
  ↓
2. For each ReplicaSet:
   現在のPod数を取得
   (label selector でフィルタ)
  ↓
3. Desired vs Actual を比較:

   Desired: 3 replicas
   Actual:  2 Pods
   → 1 Pod 不足
  ↓
4. 不足している場合:
   Pod Template から新しいPod manifest作成
   ↓
   POST /api/v1/namespaces/default/pods
   ↓
   apiserver → etcd
  ↓
5. 超過している場合:
   最も古いPodを選択
   ↓
   DELETE /api/v1/namespaces/default/pods/nginx-xxx
  ↓
6. 30秒後に再度実行 (resync period)
```

**カーネル内部での動作**:
- Controller は単なるGoプログラム
- informer cache（ローカルメモリ）を使用してAPIサーバーへの負荷を軽減
- Watch イベントで増分更新

**2. Deployment Controller**:

```
[Rolling Update]
  ↓
1. Deployment の変更を検知
   (image: nginx:1.19 → nginx:1.20)
  ↓
2. 新しい ReplicaSet を作成
   名前: nginx-deployment-7d4b8d6f9c
   replicas: 0 (初期値)
  ↓
3. RollingUpdate 戦略に従って段階的に更新:

   maxSurge: 25% (最大25%多く起動可能)
   maxUnavailable: 25% (最大25%停止可能)

   Desired: 4 replicas の場合
   maxSurge: 1, maxUnavailable: 1

   Phase 1:
   Old RS: 4 → 3 (1つ削除)
   New RS: 0 → 1 (1つ作成)

   Phase 2:
   Old RS: 3 → 2
   New RS: 1 → 2

   ... (繰り返し)

   Final:
   Old RS: 0
   New RS: 4
  ↓
4. 古い ReplicaSet の replicas を 0 に設定
   (履歴として保持、revision数は設定可能)
```

**3. Node Controller**:

```
[Node Health Check]
  ↓
1. 全Nodeを監視
   kubelet からの Heartbeat (NodeStatus更新) を確認
  ↓
2. NodeStatus の Conditions をチェック:

   Ready: True/False/Unknown
   DiskPressure: True/False
   MemoryPressure: True/False
   PIDPressure: True/False
  ↓
3. Heartbeat が途絶えた場合:

   40秒: NodeStatus が Unknown に変更
   5分: Node上のPodを eviction 対象にマーク

   (ただし、graceful shutdown期間を考慮)
  ↓
4. Pod Eviction:

   - PodのDeletionTimestamp を設定
   - kubelet が graceful termination を実行
   - timeout後に強制終了
   - scheduler が別のNodeに再配置
```

---

##### 2.7.5 kubelet

**役割**: 各ノードでPodを実行・監視するエージェント

**詳細な動作フロー**:

```
[起動時]
  ↓
1. Node リソースの検出:

   - CPU: /proc/cpuinfo から取得
   - Memory: /proc/meminfo から取得
   - Storage: df コマンドで取得
   - GPU: デバイスプラグインから取得
  ↓
2. Node オブジェクトを作成/更新:

   POST /api/v1/nodes
   {
     "metadata": {"name": "worker-1"},
     "status": {
       "capacity": {
         "cpu": "4",
         "memory": "8Gi",
         "pods": "110"
       },
       "allocatable": {
         "cpu": "3.5",    // system reserved を除外
         "memory": "7.5Gi"
       }
     }
   }
  ↓
3. apiserver を watch:

   GET /api/v1/pods?watch=true&fieldSelector=spec.nodeName=worker-1
   (HTTP/2 ストリームで受信)
  ↓
[Pod 作成イベント受信]
  ↓
4. Pod Spec を解析
  ↓
5. CNI Plugin を呼び出し:

   /opt/cni/bin/bridge <<EOF
   {
     "cniVersion": "0.4.0",
     "name": "k8s-pod-network",
     "type": "bridge",
     "bridge": "cni0",
     "ipam": {
       "type": "host-local",
       "subnet": "10.244.1.0/24"
     }
   }
   EOF

   CNI Plugin の動作:
   - veth pair 作成
   - 一端をPodのnetwork namespaceに移動
   - もう一端をbridgeに接続
   - IPアドレスを割り当て
   - ルーティング設定
  ↓
6. Volume のマウント:

   a) CSI Driver 呼び出し (PersistentVolume の場合):
      NodeStageVolume → NodePublishVolume

   b) ConfigMap/Secret (emptyDirなど):
      tmpfs をマウント
      /var/lib/kubelet/pods/<pod-id>/volumes/...
  ↓
7. Container Runtime (CRI) 呼び出し:

   RunPodSandbox (pause container)
   ↓
   CreateContainer (init containers, if any)
   ↓
   StartContainer
   ↓
   CreateContainer (app containers)
   ↓
   StartContainer
  ↓
8. Container の監視:

   - Liveness Probe:
     HTTP GET http://10.244.1.5:8080/healthz
     失敗 → restart policy に従って再起動

   - Readiness Probe:
     TCP Socket 10.244.1.5:8080
     失敗 → Service の Endpoints から削除

   - Startup Probe:
     起動時の初期化完了を確認
  ↓
9. リソース使用量の収集:

   - cgroup から取得:
     CPU: /sys/fs/cgroup/cpu/kubepods/.../cpu.stat
     Memory: /sys/fs/cgroup/memory/kubepods/.../memory.usage_in_bytes

   - metrics-server に提供
  ↓
10. Node Status の更新:

    定期的 (10秒ごと) に apiserver に送信:
    PATCH /api/v1/nodes/worker-1/status
    {
      "status": {
        "conditions": [
          {"type": "Ready", "status": "True"},
          {"type": "MemoryPressure", "status": "False"}
        ]
      }
    }
```

**Probe の実装詳細**:

```go
// Liveness Probe (HTTP)
func (pb *prober) runProbe(p *v1.Probe, pod *v1.Pod, container *v1.Container) (probe.Result, error) {
    timeout := time.Duration(p.TimeoutSeconds) * time.Second

    if p.HTTPGet != nil {
        scheme := strings.ToLower(string(p.HTTPGet.Scheme))
        host := p.HTTPGet.Host
        if host == "" {
            host = pod.Status.PodIP
        }
        port := p.HTTPGet.Port.IntValue()
        path := p.HTTPGet.Path

        url := fmt.Sprintf("%s://%s:%d%s", scheme, host, port, path)

        // HTTP リクエスト送信
        req, err := http.NewRequest("GET", url, nil)
        for _, header := range p.HTTPGet.HTTPHeaders {
            req.Header.Add(header.Name, header.Value)
        }

        client := &http.Client{Timeout: timeout}
        resp, err := client.Do(req)

        if err != nil {
            return probe.Failure, err
        }
        defer resp.Body.Close()

        // 200-399 は成功
        if resp.StatusCode >= 200 && resp.StatusCode < 400 {
            return probe.Success, nil
        }
        return probe.Failure, nil
    }

    // 他のprobe種別も同様...
}
```

---

##### 2.7.6 kube-proxy

**役割**: Service の抽象化を実現、トラフィックのロードバランシング

**3つの動作モード**:

**1. iptables モード（デフォルト）**:

```
[Service 作成]
  ↓
kube-proxy が watch で検知
  ↓
iptables ルールを追加:

# Service の ClusterIP への通信を NAT
iptables -t nat -A KUBE-SERVICES \
  -d 10.96.0.1/32 -p tcp --dport 80 \
  -j KUBE-SVC-NGINX

# Backend Podへランダムに振り分け
iptables -t nat -A KUBE-SVC-NGINX \
  -m statistic --mode random --probability 0.33 \
  -j KUBE-SEP-POD1

iptables -t nat -A KUBE-SVC-NGINX \
  -m statistic --mode random --probability 0.50 \
  -j KUBE-SEP-POD2

iptables -t nat -A KUBE-SVC-NGINX \
  -j KUBE-SEP-POD3

# 各Pod へ DNAT
iptables -t nat -A KUBE-SEP-POD1 \
  -j DNAT --to-destination 10.244.1.5:8080
```

カーネル内部での動作:
```
[パケット到着]
  dst: 10.96.0.1:80
    ↓
[netfilter PREROUTING chain]
  iptables -t nat チェック
    ↓
[KUBE-SERVICES マッチ]
  → KUBE-SVC-NGINX へジャンプ
    ↓
[確率的選択]
  random probability 0.33 → 33% の確率でPOD1
  失敗 → 次のルールへ
    ↓
[DNAT 実行]
  dst: 10.96.0.1:80 → 10.244.1.5:8080
    ↓
[ルーティング再評価]
  宛先IPが変わったので再ルーティング
    ↓
[パケット転送]
```

**2. IPVS モード**:

より高性能なL4ロードバランサー

```
kube-proxy 起動
  ↓
IPVS カーネルモジュールをロード
  ↓
Service ごとに Virtual Server 作成:

ipvsadm -A -t 10.96.0.1:80 -s rr
         ↑          ↑      ↑
      Virtual IP  Port   Round-Robin

Backend Pod ごとに Real Server 追加:

ipvsadm -a -t 10.96.0.1:80 -r 10.244.1.5:8080 -m
                                              ↑
                                          Masquerade (NAT)

ipvsadm -a -t 10.96.0.1:80 -r 10.244.1.6:8080 -m
ipvsadm -a -t 10.96.0.1:80 -r 10.244.1.7:8080 -m
```

カーネル内部での動作:
```
[パケット到着]
  dst: 10.96.0.1:80
    ↓
[netfilter LOCAL_IN chain]
    ↓
[IPVS マッチング]
  ip_vs_in() 関数が呼ばれる
    ↓
[スケジューリングアルゴリズム実行]
  Round-Robin: 順番に選択
  Least Connection: 接続数が最小のPodを選択
  Source Hash: 送信元IPで決定 (Session Affinity)
    ↓
[Real Server 選択]
  10.244.1.5:8080
    ↓
[Connection Tracking]
  conntrack でセッション管理
  戻りパケットも正しく処理
    ↓
[DNAT 実行]
  dst: 10.96.0.1:80 → 10.244.1.5:8080
```

**3. eBPF/Cilium モード**:

より高速なデータプレーン

```
[BPF Program をカーネルにロード]
  ↓
TC (Traffic Control) に attach
  ↓
パケット処理が BPF で実行される:

int bpf_lb(struct __sk_buff *skb) {
    // パケットヘッダー解析
    struct ethhdr *eth = bpf_hdr_pointer(skb);
    struct iphdr *ip = (void *)eth + sizeof(*eth);

    // Service マッチング
    if (ip->daddr == SERVICE_IP) {
        // Hash計算で Backend選択
        __u32 hash = jhash(ip->saddr, ip->sport);
        __u32 backend = hash % num_backends;

        // Destination を書き換え
        ip->daddr = backends[backend].ip;

        // Checksum 再計算
        ip->check = csum_diff(...);

        return TC_ACT_OK;
    }
}
```

利点:
- カーネル空間で直接処理（ユーザー空間往復なし）
- iptables のルール走査より高速
- パケットコピー不要

---

続きを次のドキュメントで説明します。
