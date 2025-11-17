# Kubernetes技術スタック詳解 Part 3: 最上位レイヤーとマッピング

## 目次

1. [Layer 6: 高度なオーケストレーション層](#layer-6-高度なオーケストレーション層)
2. [Layer 7: プラットフォーム抽象化層](#layer-7-プラットフォーム抽象化層)
3. [カリキュラム全体のレイヤーマッピング](#カリキュラム全体のレイヤーマッピング)
4. [まとめ: レイヤー間の相互作用](#まとめ-レイヤー間の相互作用)

---

## Layer 6: 高度なオーケストレーション層

### 6.1 Karpenter - ノード自動プロビジョニング

**レイヤー位置**: Kubernetesの上、インフラストラクチャ管理の自動化

**アーキテクチャ**:
```
[Kubernetes Scheduler]
    ↓ (Podがスケジュール不可)
[Karpenter Controller]
    ↓ (EC2 API 呼び出し)
[AWS EC2]
    ↓ (新しいノード起動)
[kubelet 登録]
```

**詳細な動作原理**:

#### 6.1.1 Podの監視とノードプロビジョニング

```
[Pod作成]
  ↓
1. Kubernetes Scheduler が配置を試行
  ↓
2. すべてのノードで条件を満たせず:

   Events:
   ```
   Warning  FailedScheduling  pod/my-app-xxx
   0/3 nodes are available:
   1 Insufficient cpu,
   2 node(s) had taint {node.kubernetes.io/not-ready},
   preemption: 0/3 nodes are available
   ```
  ↓
3. Karpenter Controller が検知:

   ```go
   // Karpenter のコード（概念的）
   func (c *Controller) Reconcile(ctx context.Context) {
       // Unschedulable Pods を取得
       pods := c.getPendingPods()

       for _, pod := range pods {
           // Podの requirements を解析
           requirements := c.extractRequirements(pod)

           // 既存ノードで配置可能か再チェック
           if c.canScheduleOnExistingNodes(pod) {
               continue
           }

           // 新しいノードが必要
           c.provisionNode(requirements)
       }
   }
   ```
  ↓
4. 要件の抽出:

   ```go
   type Requirements struct {
       CPU              resource.Quantity  // 500m
       Memory           resource.Quantity  // 1Gi
       InstanceTypes    []string          // ["t3.medium", "t3.large"]
       Architecture     string            // "amd64"
       CapacityType     string            // "spot" or "on-demand"
       Zone             string            // "us-east-1a"
   }

   // Podから要件を抽出
   requirements := Requirements{
       CPU:    pod.Spec.Containers[0].Resources.Requests.Cpu(),
       Memory: pod.Spec.Containers[0].Resources.Requests.Memory(),
   }

   // NodeSelector/Affinity から追加要件
   if pod.Spec.NodeSelector["karpenter.k8s.aws/instance-category"] != "" {
       requirements.InstanceTypes = filterByCategory(...)
   }
   ```
  ↓
5. 最適なインスタンスタイプの選択:

   ```go
   func (p *Provisioner) selectInstanceType(req Requirements) string {
       candidates := []InstanceType{}

       // すべてのインスタンスタイプをチェック
       for _, it := range p.instanceTypes {
           if it.CPU >= req.CPU &&
              it.Memory >= req.Memory &&
              it.Architecture == req.Architecture {
               candidates = append(candidates, it)
           }
       }

       // 価格でソート（Spot優先、安い順）
       sort.Slice(candidates, func(i, j int) bool {
           if req.CapacityType == "spot" {
               return candidates[i].SpotPrice < candidates[j].SpotPrice
           }
           return candidates[i].OnDemandPrice < candidates[j].OnDemandPrice
       })

       // 最も安いインスタンスタイプを選択
       return candidates[0].Name  // "t3.medium"
   }
   ```
  ↓
6. EC2インスタンスの起動:

   ```go
   // AWS EC2 API 呼び出し
   runInstancesInput := &ec2.RunInstancesInput{
       ImageId:      aws.String("ami-xxxxx"),  // EKS Optimized AMI
       InstanceType: aws.String("t3.medium"),
       MinCount:     aws.Int64(1),
       MaxCount:     aws.Int64(1),

       // UserData で kubelet を設定
       UserData: aws.String(base64.StdEncoding.EncodeToString([]byte(`
#!/bin/bash
/etc/eks/bootstrap.sh my-cluster \
  --kubelet-extra-args '--node-labels=karpenter.sh/provisioner-name=default'
       `))),

       // Spot or On-Demand
       InstanceMarketOptions: &ec2.InstanceMarketOptionsRequest{
           MarketType: aws.String("spot"),
           SpotOptions: &ec2.SpotMarketOptions{
               SpotInstanceType: aws.String("one-time"),
           },
       },

       // IAM Instance Profile
       IamInstanceProfile: &ec2.IamInstanceProfileSpecification{
           Name: aws.String("KarpenterNodeInstanceProfile"),
       },

       // Networking
       NetworkInterfaces: []*ec2.InstanceNetworkInterfaceSpecification{{
           DeviceIndex:              aws.Int64(0),
           AssociatePublicIpAddress: aws.Bool(false),
           SubnetId:                 aws.String("subnet-xxxxx"),
           Groups:                   []*string{aws.String("sg-xxxxx")},
       }},

       TagSpecifications: []*ec2.TagSpecification{{
           ResourceType: aws.String("instance"),
           Tags: []*ec2.Tag{
               {Key: aws.String("Name"), Value: aws.String("karpenter-node")},
               {Key: aws.String("karpenter.sh/provisioner-name"), Value: aws.String("default")},
           },
       }},
   }

   result, err := ec2Client.RunInstances(runInstancesInput)
   ```
  ↓
7. ノードの起動プロセス:

   a) EC2インスタンス起動（AWS側）
   b) UserDataスクリプト実行:
      ```bash
      # /etc/eks/bootstrap.sh の内容
      - Docker/containerd 起動
      - kubelet 設定ファイル生成
      - kubelet 起動
      ```
   c) kubelet がクラスターに登録:
      ```
      POST /api/v1/nodes
      {
        "metadata": {
          "name": "ip-10-0-1-123.ec2.internal",
          "labels": {
            "karpenter.sh/provisioner-name": "default",
            "node.kubernetes.io/instance-type": "t3.medium",
            "topology.kubernetes.io/zone": "us-east-1a"
          }
        },
        "status": {
          "capacity": {"cpu": "2", "memory": "4Gi"},
          "allocatable": {"cpu": "1.93", "memory": "3.5Gi"}
        }
      }
      ```
  ↓
8. Scheduler がPodを新しいノードに配置
```

#### 6.1.2 Consolidation（ノードの集約）

```
[定期的なチェック（30秒ごと）]
  ↓
1. 各ノードの使用率を計算:

   ```go
   func (c *Consolidator) findUnderutilizedNodes() []*Node {
       underutilized := []*Node{}

       for _, node := range c.listNodes() {
           allocated := c.calculateAllocated(node)
           capacity := node.Status.Allocatable

           cpuUtilization := float64(allocated.CPU) / float64(capacity.CPU)
           memUtilization := float64(allocated.Memory) / float64(capacity.Memory)

           // 使用率が30%未満
           if cpuUtilization < 0.3 && memUtilization < 0.3 {
               underutilized = append(underutilized, node)
           }
       }

       return underutilized
   }
   ```
  ↓
2. ノードを削減できるかシミュレーション:

   ```go
   func (c *Consolidator) canConsolidate(nodes []*Node) bool {
       // 全Podを取得
       allPods := c.getPodsOnNodes(nodes)

       // 残りのノードに配置できるか試行
       remainingNodes := c.listNodes()
       for _, nodeToRemove := range nodes {
           remainingNodes = remove(remainingNodes, nodeToRemove)
       }

       for _, pod := range allPods {
           if !c.canSchedule(pod, remainingNodes) {
               // 配置不可 → 削減できない
               return false
           }
       }

       return true  // すべて配置可能 → 削減OK
   }
   ```
  ↓
3. ノードを削除:

   a) Cordon（新しいPodの配置を停止）:
      ```bash
      kubectl cordon ip-10-0-1-123.ec2.internal
      ```

   b) Drain（既存Podを退避）:
      ```bash
      kubectl drain ip-10-0-1-123.ec2.internal \
        --ignore-daemonsets \
        --delete-emptydir-data
      ```

      内部処理:
      - Pod に DeletionTimestamp を設定
      - Pod の Graceful Termination:
        1. SIGTERM送信
        2. preStop hook 実行
        3. terminationGracePeriodSeconds 待機（default 30s）
        4. SIGKILL送信（強制終了）

   c) ノード削除:
      ```bash
      kubectl delete node ip-10-0-1-123.ec2.internal
      ```

   d) EC2インスタンス終了:
      ```go
      ec2Client.TerminateInstances(&ec2.TerminateInstancesInput{
          InstanceIds: []*string{aws.String("i-xxxxx")},
      })
      ```
```

---

### 6.2 Crossplane - Infrastructure as Code

**レイヤー位置**: Kubernetesの上、クラウドリソース管理

**アーキテクチャ**:
```
[Kubernetes API]
    ↓
[Crossplane CRDs]
  - XRD (Composite Resource Definition)
  - Composition
  - Managed Resources
    ↓
[Provider Controllers]
  - provider-aws
  - provider-azure
  - provider-gcp
    ↓
[Cloud Provider APIs]
```

**詳細な動作原理**:

#### 6.2.1 Managed Resource の作成

```
[S3 Bucket作成リクエスト]
  ↓
1. kubectl apply:

   ```yaml
   apiVersion: s3.aws.upbound.io/v1beta1
   kind: Bucket
   metadata:
     name: my-app-bucket
   spec:
     forProvider:
       region: us-east-1
       tags:
         Environment: prod
     providerConfigRef:
       name: default
   ```
  ↓
2. Crossplane Controller が検知:

   ```go
   // Provider Controllerのコード（概念的）
   func (c *BucketController) Reconcile(ctx context.Context, req reconcile.Request) {
       // Bucket リソースを取得
       bucket := &s3v1beta1.Bucket{}
       err := c.client.Get(ctx, req.NamespacedName, bucket)

       // AWS Credentials を取得
       pc := &awsv1beta1.ProviderConfig{}
       c.client.Get(ctx, bucket.Spec.ProviderConfigRef.Name, pc)

       awsConfig := c.getAWSConfig(pc)
       s3Client := s3.NewFromConfig(awsConfig)

       // 実際のS3バケットの状態を確認
       _, err = s3Client.HeadBucket(ctx, &s3.HeadBucketInput{
           Bucket: aws.String(bucket.Spec.ForProvider.BucketName),
       })

       if err != nil {
           // バケットが存在しない → 作成
           c.createBucket(s3Client, bucket)
       } else {
           // 存在する → 更新が必要か確認
           c.updateBucket(s3Client, bucket)
       }

       // Statusを更新
       bucket.Status.AtProvider.Arn = "arn:aws:s3:::my-app-bucket"
       bucket.Status.Conditions = []Condition{{Type: "Ready", Status: "True"}}
       c.client.Status().Update(ctx, bucket)
   }
   ```
  ↓
3. AWS S3 API 呼び出し:

   ```go
   func (c *BucketController) createBucket(client *s3.Client, bucket *Bucket) error {
       input := &s3.CreateBucketInput{
           Bucket: aws.String(bucket.Spec.ForProvider.BucketName),
           CreateBucketConfiguration: &types.CreateBucketConfiguration{
               LocationConstraint: types.BucketLocationConstraint(
                   bucket.Spec.ForProvider.Region,
               ),
           },
       }

       _, err := client.CreateBucket(context.TODO(), input)
       if err != nil {
           return err
       }

       // タグを設定
       _, err = client.PutBucketTagging(context.TODO(), &s3.PutBucketTaggingInput{
           Bucket: aws.String(bucket.Spec.ForProvider.BucketName),
           Tagging: &types.Tagging{
               TagSet: []types.Tag{
                   {Key: aws.String("Environment"), Value: aws.String("prod")},
               },
           },
       })

       return err
   }
   ```
  ↓
4. AWSでリソースが作成される
  ↓
5. Crossplane が継続的に監視:

   - 30秒ごとにリソースの状態を確認
   - Drift Detection（手動変更の検出）
   - 不一致があれば修正
```

#### 6.2.2 Composition（複合リソース）

```
[XRD (Composite Resource Definition) 作成]
  ↓
1. カスタムAPIを定義:

   ```yaml
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
                 parameters:
                   type: object
                   properties:
                     size:
                       type: string
                       enum: [small, medium, large]
                     storageGB:
                       type: integer
               required: [parameters]
   ```
  ↓
2. Composition を作成:

   ```yaml
   apiVersion: apiextensions.crossplane.io/v1
   kind: Composition
   metadata:
     name: dynamodb-composition
   spec:
     compositeTypeRef:
       apiVersion: example.com/v1alpha1
       kind: XDatabase

     resources:
     # DynamoDB Table
     - name: table
       base:
         apiVersion: dynamodb.aws.upbound.io/v1beta1
         kind: Table
         spec:
           forProvider:
             region: us-east-1
             attribute:
             - name: id
               type: S
             hashKey: id
       patches:
       - type: FromCompositeFieldPath
         fromFieldPath: spec.parameters.size
         toFieldPath: spec.forProvider.billingMode
         transforms:
         - type: map
           map:
             small: PAY_PER_REQUEST
             medium: PROVISIONED
             large: PROVISIONED

     # CloudWatch Alarm
     - name: alarm
       base:
         apiVersion: cloudwatch.aws.upbound.io/v1beta1
         kind: MetricAlarm
         spec:
           forProvider:
             region: us-east-1
             comparisonOperator: GreaterThanThreshold
             evaluationPeriods: 2
             metricName: ConsumedReadCapacityUnits
             namespace: AWS/DynamoDB
             period: 300
             statistic: Average
             threshold: 80
   ```
  ↓
3. ユーザーがDatabaseを作成:

   ```yaml
   apiVersion: example.com/v1alpha1
   kind: Database
   metadata:
     name: my-app-db
     namespace: production
   spec:
     parameters:
       size: medium
       storageGB: 100
   ```
  ↓
4. Crossplane が処理:

   ```go
   func (c *CompositionController) Reconcile(ctx context.Context, db *Database) {
       // Composition を取得
       comp := c.getComposition(db)

       // Composite Resource (XDatabase) を作成
       xr := &XDatabase{
           Spec: db.Spec,
       }
       c.client.Create(ctx, xr)

       // Composition の各リソースをレンダリング
       for _, resource := range comp.Spec.Resources {
           // Patches を適用
           rendered := c.applyPatches(resource, db.Spec)

           // Managed Resource を作成
           mr := c.createManagedResource(rendered)
           c.client.Create(ctx, mr)
       }

       // StatusをClaimに反映
       db.Status.ResourceRef = xr.GetReference()
       c.client.Status().Update(ctx, db)
   }
   ```
  ↓
5. 複数のリソースが自動作成:

   - DynamoDB Table
   - CloudWatch Alarm
   - （必要に応じて）IAM Role, VPC Endpoint等
  ↓
6. すべてのリソースの状態を集約:

   ```
   Database "my-app-db"
     Status: Ready
     Resources:
       - DynamoDB Table: arn:aws:dynamodb:us-east-1:...:table/my-app-db
       - CloudWatch Alarm: arn:aws:cloudwatch:us-east-1:...:alarm:my-app-db-alarm
   ```
```

---

## Layer 7: プラットフォーム抽象化層

### 7.1 Render - シンプルなPaaS

**レイヤー位置**: Kubernetes完全隠蔽、アプリケーション中心のAPI

**内部アーキテクチャ（推測）**:
```
[Render API]
    ↓
[Render Orchestrator]
    ├─ Git Watcher
    ├─ Build System
    └─ Deployment Manager
        ↓
[Kubernetes (GKE/EKS)]
    ↓
[Infrastructure (GCP/AWS)]
```

**詳細な動作フロー**:

```
[GitHubにコミット]
  ↓
1. Render Webhook 受信:

   POST https://api.render.com/webhooks/github
   {
     "repository": "user/myapp",
     "branch": "main",
     "commit": "abc123..."
   }
  ↓
2. Git Repository をクローン:

   ```bash
   git clone https://github.com/user/myapp.git /tmp/build-abc123
   cd /tmp/build-abc123
   git checkout abc123
   ```
  ↓
3. Buildpack または Dockerfile を検出:

   a) Dockerfile がある場合:
      ```bash
      docker build -t render.io/user/myapp:abc123 .
      docker push render.io/user/myapp:abc123
      ```

   b) Buildpack の場合（Node.js検出）:
      ```bash
      # package.json を検出
      # Node.js Buildpack を使用
      pack build render.io/user/myapp:abc123 \
        --builder heroku/buildpacks:20
      ```
  ↓
4. Kubernetes Manifest を生成（内部処理）:

   render.yaml から変換:
   ```yaml
   # render.yaml
   services:
   - type: web
     name: myapp
     env: node
     buildCommand: npm install
     startCommand: npm start
     envVars:
     - key: DATABASE_URL
       fromDatabase:
         name: mydb
         property: connectionString
   ```

   ↓ 変換 ↓

   ```yaml
   # 内部的に生成されるKubernetes Manifest
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp-abc123
     namespace: user-namespace
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: myapp
         version: abc123
     template:
       spec:
         containers:
         - name: app
           image: render.io/user/myapp:abc123
           ports:
           - containerPort: 10000
           env:
           - name: DATABASE_URL
             valueFrom:
               secretKeyRef:
                 name: mydb-connection
                 key: url
           resources:
             requests:
               memory: "512Mi"
               cpu: "500m"
             limits:
               memory: "1Gi"
               cpu: "1000m"
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: myapp
   spec:
     type: LoadBalancer
     selector:
       app: myapp
       version: abc123
     ports:
     - port: 443
       targetPort: 10000
   ```
  ↓
5. Blue-Green Deployment:

   a) 新しいバージョンをデプロイ（Green）:
      ```bash
      kubectl apply -f deployment-abc123.yaml
      ```

   b) Health Check が成功するまで待機:
      ```bash
      kubectl wait --for=condition=available deployment/myapp-abc123
      ```

   c) Service を新バージョンに切り替え:
      ```yaml
      selector:
        app: myapp
        version: abc123  # 古いバージョンから切り替え
      ```

   d) 古いバージョンを削除:
      ```bash
      kubectl delete deployment myapp-old
      ```
  ↓
6. SSL証明書の自動管理:

   - Let's Encrypt で証明書取得
   - cert-manager (Kubernetes) で自動更新
   - Ingress に証明書を設定
```

---

### 7.2 Fly.io - エッジコンピューティングプラットフォーム

**レイヤー位置**: Kubernetes非使用、独自のオーケストレーション

**アーキテクチャ**:
```
[Fly.io Control Plane]
    ↓
[Fly Machines API]
    ↓
[Firecracker MicroVMs]
  （各リージョンのホスト上）
```

**詳細な動作原理**:

#### 7.2.1 Anycast Networking

```
[ユーザーのリクエスト]
  DNS: myapp.fly.dev → 66.241.124.15 (Anycast IP)
    ↓
[Anycast ルーティング]

  世界中の複数データセンターが同じIPアドレスを広告:

  ```
  Tokyo DC:     BGP Announce 66.241.124.15/32
  Singapore DC: BGP Announce 66.241.124.15/32
  Sydney DC:    BGP Announce 66.241.124.15/32
  ```

  インターネット上のルーターは、最も「近い」DCにルーティング:
  - 日本のユーザー → Tokyo DC
  - シンガポールのユーザー → Singapore DC
  - オーストラリアのユーザー → Sydney DC
    ↓
[エッジプロキシ]
  最寄りのDCでリクエストを受信
    ↓
1. アプリインスタンスがローカルにあるか確認:

   ```
   Tokyo DC:
   - myapp-instance-1: Running
   - myapp-instance-2: Running

   Singapore DC:
   - myapp-instance-3: Running
   ```

2. ローカルにある場合:
   → 即座に転送（低レイテンシ）

3. ローカルにない場合:
   → WireGuard VPN経由で他のDCに転送
    ↓
[Firecracker MicroVM]
  リクエストを処理
    ↓
[レスポンス]
  同じ経路で返却
```

#### 7.2.2 Firecracker MicroVM

```
[Fly Machine 起動]
  ↓
1. Firecracker VMM (Virtual Machine Monitor) を起動:

   ```rust
   // Firecracker API (概念的)
   let vm_config = VmConfig {
       vcpu_count: 1,
       mem_size_mib: 256,
       kernel_image_path: "/path/to/vmlinux",
       rootfs_path: "/path/to/rootfs.ext4",
       boot_args: "console=ttyS0 reboot=k panic=1",
   };

   let firecracker = Firecracker::new(vm_config);
   firecracker.start();
   ```
  ↓
2. 起動プロセス（超高速、<1秒）:

   a) KVM (Kernel Virtual Machine) を使用:
      ```
      /dev/kvm デバイスを開く
      ioctl(KVM_CREATE_VM) → VM作成
      ioctl(KVM_CREATE_VCPU) → vCPU作成
      ```

   b) ミニマムなLinuxカーネルをロード:
      - カーネルサイズ: ~5MB（通常のLinuxカーネルは10-20MB）
      - 不要なドライバー・モジュールを削除
      - 起動時間: 125ms

   c) ルートファイルシステムをマウント:
      - OverlayFS でイメージレイヤーをマウント
      - virtio-blk でブロックデバイスとして提供

   d) init プロセス起動:
      ```
      /sbin/init (PID 1)
        ↓
      アプリケーションプロセス起動
      ```
  ↓
3. ネットワーク設定:

   a) TAP device作成:
      ```bash
      ip tuntap add tap0 mode tap
      ip addr add 172.16.0.1/24 dev tap0
      ip link set tap0 up
      ```

   b) VM内部のネットワーク:
      ```
      VM: eth0 172.16.0.2/24
      Host: tap0 172.16.0.1/24 (gateway)
      ```

   c) iptables でNAT:
      ```bash
      iptables -t nat -A POSTROUTING -s 172.16.0.0/24 -j MASQUERADE
      ```
  ↓
4. WireGuard VPN でグローバル接続:

   各VM には WireGuard インターフェースが自動設定:
   ```
   [Interface]
   PrivateKey = <生成された秘密鍵>
   Address = fdaa:0:1234::2/48

   [Peer]
   PublicKey = <FlyネットワークのPublicKey>
   AllowedIPs = fdaa::/16
   Endpoint = gateway.fly.io:51820
   ```

   これにより、全世界のVM間でプライベートネットワーク構築
```

#### 7.2.3 Fly Postgres（分散データベース）

```
[Fly Postgres クラスター]
  ↓
1. Stolon（PostgreSQL HA ツール）を使用:

   ```
   [Leader Node] (Tokyo)
     PostgreSQL (Read-Write)
       ↓
     WAL Replication
       ↓
   [Replica Node] (Singapore)
     PostgreSQL (Read-Only)
       ↓
   [Replica Node] (Sydney)
     PostgreSQL (Read-Only)
   ```
  ↓
2. Consul でLeader Election:

   ```
   Tokyo Node:
     - Consul Agent 起動
     - Leader Lock 取得を試行
     - 成功 → PostgreSQL を Primary モードで起動

   Singapore Node:
     - Consul Agent 起動
     - Leader が存在 → Replica モードで起動
     - Tokyo からWAL Streaming
   ```
  ↓
3. WAL (Write-Ahead Log) Replication:

   ```
   [Write Transaction on Leader]
   BEGIN;
   INSERT INTO users (name) VALUES ('Alice');
   COMMIT;
     ↓
   WAL Record 作成:
   {
     "lsn": "0/1234ABCD",
     "type": "INSERT",
     "relation": "users",
     "data": {"name": "Alice"}
   }
     ↓
   WAL を Replica にストリーミング:
   TCP接続でバイナリデータ送信
     ↓
   [Replica]
   WAL を受信
     ↓
   WAL を適用（Replay）:
   INSERT INTO users (name) VALUES ('Alice');
     ↓
   レプリケーション完了
   ```
  ↓
4. Failover:

   ```
   [Leader障害]
     ↓
   Consul がヘルスチェック失敗を検知
     ↓
   新しいLeaderを選出（Raft Consensus）
     ↓
   Singapore Node が Leader に昇格:
   - PostgreSQL を promote:
     pg_ctl promote
   - Primary モードに変更
   - WAL Streaming の向きを変更
     ↓
   アプリケーションの接続先を自動変更
   （Consul DNS: postgres.internal → 新Leader）
   ```
```

---

## カリキュラム全体のレイヤーマッピング

### カリキュラムと技術スタックの対応表

| カリキュラム章 | 主要レイヤー | 関連する下位レイヤー | 学習する技術原理 |
|--------------|------------|-------------------|----------------|
| **environments/** | Layer 2-3 | Layer 0-1 | コンテナランタイム、Kubernetesコンポーネント |
| **01-basic-concepts** | Layer 3 | Layer 0-2 | Pod作成、namespace/cgroup、ネットワーク |
| **02-deployments** | Layer 3 | Layer 1-2 | ReplicaSet Controller、Scheduler |
| **03-services** | Layer 3 | Layer 1-2 | kube-proxy、iptables/IPVS、ネットワークスタック |
| **04-advanced-topics** | Layer 3 | Layer 0-2 | HPA、PersistentVolume、NetworkPolicy |
| **05-helm** | Layer 4 | Layer 3 | Template Engine、Release Management |
| **06-lens** | Layer 4 | Layer 3 | Kubernetes API、Watch、リソース可視化 |
| **07-linkerd** | Layer 5 | Layer 2-4 | Proxy Injection、mTLS、Metrics |
| **08-istio** | Layer 5 | Layer 2-4 | Envoy、xDS API、Distributed Tracing |
| **09-karpenter** | Layer 6 | Layer 0-3 | EC2 API、Scheduling、AutoScaling |
| **09-crossplane** | Layer 6 | Layer 3 | CRD、Cloud Provider API、GitOps |
| **10-render** | Layer 7 | Layer 0-6 | Buildpack、Blue-Green Deployment |
| **10-flyio** | Layer 7 | Layer 0-2 | Firecracker、Anycast、WireGuard |

---

## まとめ: レイヤー間の相互作用

### 実例: Pod作成から通信まで（全レイヤー）

```
[kubectl apply -f pod.yaml] (Layer 4: User)
    ↓
[kube-apiserver] (Layer 3)
  - 認証・認可
  - Admission Control
  - etcd への書き込み
    ↓
[kube-scheduler] (Layer 3)
  - Filtering (NodeResourcesFit, NodeAffinity)
  - Scoring (バランシング)
  - Node 選択
    ↓
[kubelet] (Layer 3)
  - CNI Plugin 呼び出し
    ↓
[CNI Plugin] (Layer 2-3)
  - veth pair 作成 (Layer 1: netlink API)
  - IP割り当て (Layer 3)
  - iptables 設定 (Layer 1: netfilter)
    ↓
[Container Runtime] (Layer 2)
  - runc 実行
    ↓
[runc] (Layer 2)
  - clone() with CLONE_NEW* (Layer 1: syscall)
  - namespace 作成 (Layer 1: kernel)
  - cgroup 設定 (Layer 1: /sys/fs/cgroup)
  - pivot_root() (Layer 1: syscall)
  - execve() (Layer 1: syscall)
    ↓
[Linux Kernel] (Layer 1)
  - Namespace 分離
  - Cgroup リソース制限
  - ネットワークスタック
  - OverlayFS
    ↓
[Hardware] (Layer 0)
  - CPU でプロセス実行
  - メモリにデータ配置
  - NIC でパケット送受信
```

### Linkerd mTLS通信の全レイヤー追跡

```
[App A: HTTP GET /api] (Application)
    ↓
[connect(10.96.0.1, 80)] (syscall)
    ↓
[iptables REDIRECT] (Layer 1: netfilter)
  → localhost:4143 (linkerd-proxy)
    ↓
[linkerd-proxy] (Layer 5)
  - Destination Service に問い合わせ (Layer 3: Kubernetes API)
  - Backend Pod を選択
  - TLS Handshake (Layer 1: OpenSSL/BoringSSL)
  - HTTP リクエスト転送
    ↓
[Encrypted Packet] (Layer 1)
  - TCP/IP Stack
  - NIC から送信 (Layer 0: DMA)
    ↓
[Network] (Physical Layer 0)
  - Ethernetフレーム
  - IPパケット
    ↓
[Destination Node NIC] (Layer 0)
  - Interrupt
  - DMA to Ring Buffer
    ↓
[Destination iptables] (Layer 1)
  → linkerd-proxy
    ↓
[linkerd-proxy decrypt] (Layer 5)
  - TLS decrypt
  - Plain HTTP
    ↓
[App B receives request] (Application)
```

---

## 各レイヤーでの主なシステムコール・API

### Layer 0 (Hardware)
- **CPU命令**: MOV, ADD, JMP, CALL, RET
- **メモリアクセス**: Load/Store命令
- **I/O**: IN/OUT命令（x86）、MMIO

### Layer 1 (OS/Kernel)
- **Namespace**: clone(CLONE_NEW*), setns(), unshare()
- **Cgroup**: open(), write() to /sys/fs/cgroup/*
- **Network**: socket(), bind(), listen(), accept(), connect(), sendto(), recvfrom()
- **iptables**: setsockopt(IP_SO_SET_REPLACE)
- **ファイルシステム**: mount(), umount(), pivot_root()

### Layer 2 (Container Runtime)
- **CRI (gRPC)**: RunPodSandbox, CreateContainer, StartContainer
- **OCI Runtime**: runc create, runc start

### Layer 3 (Kubernetes)
- **API**: GET/POST/PUT/DELETE /api/v1/*
- **Watch**: GET /api/v1/pods?watch=true
- **Exec**: WebSocket /api/v1/namespaces/*/pods/*/exec

### Layer 4 (Application Management)
- **Helm**: Go template rendering
- **Lens**: Kubernetes API + Electron IPC

### Layer 5 (Service Mesh)
- **xDS API (gRPC)**: StreamAggregatedResources
- **Envoy**: Listener/Cluster/Route/Endpoint Discovery

### Layer 6 (Orchestration)
- **Cloud Provider API**: EC2 RunInstances, S3 CreateBucket
- **Kubernetes CRD**: Custom Resource CRUD

### Layer 7 (Platform)
- **Platform API**: HTTP REST API (Render, Fly.io)
- **Buildpack**: CNB (Cloud Native Buildpacks) API

---

これで、カリキュラムのすべての技術について、CPUからプラットフォームまでの全レイヤーでの動作原理を説明しました。各技術がどのレイヤーに位置し、下位レイヤーの機能をどのように活用しているかが理解できるはずです。
