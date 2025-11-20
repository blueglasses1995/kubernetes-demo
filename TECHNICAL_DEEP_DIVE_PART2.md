# Kubernetes技術スタック詳解 Part 2: 上位レイヤーとカリキュラム技術

## 目次

1. [Layer 4: アプリケーション管理層](#layer-4-アプリケーション管理層)
2. [Layer 5: サービスメッシュ層](#layer-5-サービスメッシュ層)
3. [Layer 6: 高度なオーケストレーション層](#layer-6-高度なオーケストレーション層)
4. [Layer 7: プラットフォーム抽象化層](#layer-7-プラットフォーム抽象化層)
5. [カリキュラム技術のレイヤーマッピング](#カリキュラム技術のレイヤーマッピング)

---

## Layer 4: アプリケーション管理層

### 4.1 Helm - Kubernetesパッケージマネージャー

**レイヤー位置**: Kubernetes APIの上、アプリケーション管理の抽象化

**アーキテクチャ**:
```
[Helm CLI]
    ↓
[Chart (YAML Templates)]
    ↓
[Template Engine (Go template)]
    ↓
[Rendered Kubernetes Manifests]
    ↓
[kubectl apply]
    ↓
[kube-apiserver]
```

**詳細な動作原理**:

#### 4.1.1 Template Engine の動作

```
[Chart ディレクトリ]
templates/
  deployment.yaml:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: {{ include "mychart.fullname" . }}
      labels:
        {{- include "mychart.labels" . | nindent 4 }}
    spec:
      replicas: {{ .Values.replicaCount }}
    ```

values.yaml:
    ```yaml
    replicaCount: 3
    image:
      repository: nginx
      tag: "1.19"
    ```

[helm install の実行]
  ↓
1. Chart ファイルを読み込み
  ↓
2. values.yaml をパース（YAML → Go構造体）
  ↓
3. Template Engine 実行:

   a) _helpers.tpl の define を評価:
      ```
      {{- define "mychart.fullname" -}}
      {{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 -}}
      {{- end -}}
      ```

   b) 変数展開:
      {{ .Values.replicaCount }} → 3
      {{ include "mychart.fullname" . }} → "my-release-mychart"

   c) 関数実行:
      | nindent 4  # 4スペースインデント
      | quote      # クォートで囲む
      | default "value"  # デフォルト値

   d) 制御構文:
      {{- if .Values.service.enabled }}
      {{- range .Values.env }}
      {{- with .Values.image }}
  ↓
4. 最終的なYAMLを生成:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: my-release-mychart
     labels:
       app.kubernetes.io/name: mychart
       app.kubernetes.io/instance: my-release
   spec:
     replicas: 3
   ```
  ↓
5. kubectl apply 相当の処理:
   - YAML を JSON に変換
   - HTTP POST /apis/apps/v1/namespaces/default/deployments
   - kube-apiserver へ送信
```

#### 4.1.2 Release Management

**Secret based backend（Helm 3）**:

```
[helm install my-release ./mychart]
  ↓
1. Chart のレンダリング（上記）
  ↓
2. Release オブジェクトを作成:
   ```json
   {
     "name": "my-release",
     "namespace": "default",
     "version": 1,
     "chart": {
       "metadata": {...},
       "values": {...}
     },
     "manifest": "---\napiVersion: apps/v1\n...",
     "info": {
       "first_deployed": "2024-01-01T00:00:00Z",
       "status": "deployed"
     }
   }
   ```
  ↓
3. Release を Secret として保存:
   ```bash
   kubectl create secret generic \
     sh.helm.release.v1.my-release.v1 \
     --from-literal=release=<base64 encoded release data>
   ```

   Label:
     owner: helm
     status: deployed
     name: my-release
     version: "1"
  ↓
4. Kubernetes リソースを作成
  ↓
[helm upgrade の実行]
  ↓
1. 既存 Release (v1) を取得
  ↓
2. 新しい manifest をレンダリング
  ↓
3. Three-Way Merge:
   - Previous manifest (v1の状態)
   - Current state (kubectl get で取得)
   - New manifest (v2の内容)
  ↓
4. 差分を kubectl apply
  ↓
5. 新しい Release (v2) を Secret として保存
   sh.helm.release.v1.my-release.v2
  ↓
6. 古い Release は保持（rollback用）
```

**Three-Way Merge の詳細**:

```
Previous (v1):
  replicas: 3
  image: nginx:1.19

Current (kubectl get):
  replicas: 5  ← 手動で変更された
  image: nginx:1.19

New (v2):
  replicas: 3
  image: nginx:1.20

[Merge アルゴリズム]
1. Previous と Current を比較:
   replicas が 3 → 5 に変更されている
   → これは手動変更（drift）

2. Previous と New を比較:
   image が 1.19 → 1.20 に変更
   → これは意図的な変更

3. 最終結果:
   replicas: 3  ← New の値で上書き（driftを修正）
   image: nginx:1.20  ← New の値
```

---

### 4.2 Lens - Kubernetes IDE

**レイヤー位置**: kubectl の上、可視化レイヤー

**アーキテクチャ**:
```
[Lens UI (Electron)]
    ↓
[Lens Extension API]
    ↓
[kubectl proxy] または [直接 kube-apiserver]
    ↓
[kube-apiserver]
```

**詳細な動作原理**:

#### 4.2.1 クラスター接続

```
[Lens 起動]
  ↓
1. kubeconfig を読み込み:
   ~/.kube/config

   ```yaml
   apiVersion: v1
   clusters:
   - cluster:
       certificate-authority-data: <base64>
       server: https://kubernetes.docker.internal:6443
     name: docker-desktop
   contexts:
   - context:
       cluster: docker-desktop
       user: docker-desktop
     name: docker-desktop
   users:
   - name: docker-desktop
     user:
       client-certificate-data: <base64>
       client-key-data: <base64>
   ```
  ↓
2. TLS接続確立:

   a) Client Certificate を読み込み
   b) HTTPS接続:
      ```
      ClientHello (TLS 1.3)
        ↓
      ServerHello + Certificate
        ↓
      Certificate Verify (Client Cert)
        ↓
      Finished
      ```
  ↓
3. API Discovery:
   GET /api
   GET /apis

   レスポンス:
   ```json
   {
     "versions": ["v1"],
     "serverAddressByClientCIDRs": [...]
   }

   {
     "groups": [
       {"name": "apps", "versions": [{"version": "v1"}]},
       {"name": "batch", "versions": [{"version": "v1"}]}
     ]
   }
   ```
  ↓
4. すべてのリソースタイプを取得:
   - /api/v1
   - /apis/apps/v1
   - /apis/batch/v1
   - etc.

   Lens内部でスキーマをキャッシュ
```

#### 4.2.2 リソース可視化

```
[Pod一覧表示]
  ↓
1. HTTP リクエスト:
   GET /api/v1/namespaces/default/pods

   レスポンス:
   ```json
   {
     "kind": "PodList",
     "items": [
       {
         "metadata": {"name": "nginx-xxx"},
         "spec": {...},
         "status": {
           "phase": "Running",
           "conditions": [...],
           "containerStatuses": [...]
         }
       }
     ]
   }
   ```
  ↓
2. Lens UIでレンダリング:

   a) Status Icon の決定:
      ```typescript
      function getPodStatusIcon(pod: Pod): Icon {
        if (pod.status.phase === "Running") {
          // すべてのコンテナがReadyか？
          const allReady = pod.status.containerStatuses
            .every(c => c.ready);

          return allReady ? GreenCheckmark : YellowWarning;
        }
        if (pod.status.phase === "Pending") return YellowClock;
        if (pod.status.phase === "Failed") return RedX;
      }
      ```

   b) メトリクスの取得:
      ```
      GET /apis/metrics.k8s.io/v1beta1/namespaces/default/pods/nginx-xxx

      {
        "containers": [{
          "name": "nginx",
          "usage": {
            "cpu": "100m",
            "memory": "128Mi"
          }
        }]
      }
      ```

   c) グラフ描画:
      - Prometheus からメトリクスを取得（Lens Metrics 拡張）
      - Chart.js でグラフレンダリング
```

#### 4.2.3 Watch メカニズム

```
[Lens がリソースを監視]
  ↓
1. Watch API を使用:
   GET /api/v1/namespaces/default/pods?watch=true

   HTTP/1.1 Chunked Transfer Encoding または HTTP/2 Stream
  ↓
2. イベントストリームを受信:
   ```json
   {"type":"ADDED","object":{...}}
   {"type":"MODIFIED","object":{...}}
   {"type":"DELETED","object":{...}}
   ```
  ↓
3. Lens 内部の状態更新:

   ```typescript
   class PodStore {
     private pods: Map<string, Pod> = new Map();

     handleWatchEvent(event: WatchEvent) {
       switch (event.type) {
         case "ADDED":
           this.pods.set(event.object.metadata.name, event.object);
           this.notifyListeners();
           break;
         case "MODIFIED":
           this.pods.set(event.object.metadata.name, event.object);
           this.notifyListeners();
           break;
         case "DELETED":
           this.pods.delete(event.object.metadata.name);
           this.notifyListeners();
           break;
       }
     }
   }
   ```
  ↓
4. UI の自動更新:
   React/MobX の Observer パターンで自動再レンダリング
```

---

## Layer 5: サービスメッシュ層

### 5.1 サービスメッシュの基本原理

**レイヤー位置**: Kubernetesの上、アプリケーション間通信の制御

**基本アーキテクチャ**:
```
┌─────────────────────────────────────────────────────────┐
│ Control Plane                                           │
│  - サービスディスカバリー                                 │
│  - 証明書管理（mTLS）                                    │
│  - ポリシー配信                                          │
└─────────────────────────────────────────────────────────┘
             ↓ (gRPC/xDS API)
┌─────────────────────────────────────────────────────────┐
│ Data Plane                                              │
│                                                         │
│  Pod A               Pod B               Pod C          │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐    │
│  │App│Proxy│        │App│Proxy│        │App│Proxy│    │
│  └─────────┘        └─────────┘        └─────────┘    │
│      ↓ mTLS             ↓ mTLS             ↓           │
│      └─────────────────→                               │
└─────────────────────────────────────────────────────────┘
```

---

### 5.2 Linkerd - 軽量サービスメッシュ

**アーキテクチャ**:
```
[Linkerd Control Plane]
  ├─ linkerd-destination (サービスディスカバリー)
  ├─ linkerd-identity (証明書管理)
  └─ linkerd-proxy-injector (自動注入)
       ↓
[Data Plane]
  linkerd2-proxy (Rust製の超軽量プロキシ)
```

**詳細な動作原理**:

#### 5.2.1 Proxy Injection

```
[Podが作成される]
  ↓
1. kube-apiserver が Admission Webhook を呼び出し:

   POST https://linkerd-proxy-injector.linkerd.svc:443/mutate
   {
     "request": {
       "object": {
         "kind": "Pod",
         "metadata": {"annotations": {"linkerd.io/inject": "enabled"}},
         "spec": {
           "containers": [{
             "name": "app",
             "image": "myapp:1.0"
           }]
         }
       }
     }
   }
  ↓
2. linkerd-proxy-injector の処理:

   ```go
   func (w *Webhook) Mutate(pod *corev1.Pod) {
       // Init Container を追加（iptables設定）
       pod.Spec.InitContainers = append(
           pod.Spec.InitContainers,
           corev1.Container{
               Name:  "linkerd-init",
               Image: "cr.l5d.io/linkerd/proxy-init:v2.x.x",
               SecurityContext: &corev1.SecurityContext{
                   Capabilities: &corev1.Capabilities{
                       Add: []corev1.Capability{"NET_ADMIN"},
                   },
               },
           },
       )

       // Proxy Container を追加
       pod.Spec.Containers = append(
           pod.Spec.Containers,
           corev1.Container{
               Name:  "linkerd-proxy",
               Image: "cr.l5d.io/linkerd/proxy:stable-2.x.x",
               Ports: []corev1.ContainerPort{
                   {Name: "linkerd-proxy", ContainerPort: 4143},
                   {Name: "linkerd-admin", ContainerPort: 4191},
               },
               Env: []corev1.EnvVar{
                   {Name: "LINKERD2_PROXY_DESTINATION_SVC_ADDR", Value: "linkerd-destination.linkerd.svc.cluster.local:8086"},
                   {Name: "LINKERD2_PROXY_IDENTITY_SVC_ADDR", Value: "linkerd-identity.linkerd.svc.cluster.local:8080"},
               },
           },
       )
   }
   ```
  ↓
3. 変更されたPod Specが返される
  ↓
4. kubelet がPodを起動:

   a) linkerd-init が実行:
      ```bash
      # すべての送信トラフィックをプロキシにリダイレクト
      iptables -t nat -N PROXY_INIT_REDIRECT
      iptables -t nat -A PROXY_INIT_REDIRECT \
        -p tcp \
        -j REDIRECT --to-port 4143

      # アプリからの送信をリダイレクト
      iptables -t nat -A OUTPUT \
        -m owner ! --uid-owner 2102 \  # proxyを除外
        -j PROXY_INIT_REDIRECT

      # 受信トラフィックもプロキシ経由
      iptables -t nat -A PREROUTING \
        -p tcp \
        -j REDIRECT --to-port 4143
      ```

   b) linkerd-proxy が起動
   c) アプリコンテナが起動
```

#### 5.2.2 mTLS の自動化

```
[linkerd-proxy 起動]
  ↓
1. linkerd-identity に証明書をリクエスト:

   gRPC Certify RPC:
   ```protobuf
   message CertifyRequest {
     string token = 1;  // Kubernetes ServiceAccount Token
     string identity = 2;  // Pod のServiceAccount
   }
   ```
  ↓
2. linkerd-identity の処理:

   a) Token を検証（TokenReview API）:
      ```bash
      POST /apis/authentication.k8s.io/v1/tokenreviews
      {
        "spec": {"token": "eyJh..."}
      }

      Response:
      {
        "status": {
          "authenticated": true,
          "user": {
            "username": "system:serviceaccount:default:my-app"
          }
        }
      }
      ```

   b) 証明書を発行:
      ```
      CA Private Key で署名
        ↓
      X.509 証明書を生成:
      Subject: CN=my-app.default.serviceaccount.identity.linkerd.cluster.local
      SAN: URI:my-app.default.serviceaccount.identity.linkerd.cluster.local
      Validity: 24 hours
      ```

   c) 証明書を返す
  ↓
3. linkerd-proxy が証明書を受け取り、メモリに保存
  ↓
[アプリが外部サービスに接続]
  ↓
1. アプリ: connect(10.96.0.1, 80)
  ↓
2. iptables でプロキシにリダイレクト
  ↓
3. linkerd-proxy が接続を受け取る
  ↓
4. Destination Service に問い合わせ:

   gRPC Get RPC:
   ```protobuf
   message GetDestination {
     string scheme = 1;  // "k8s"
     string path = 2;    // "my-service.default.svc.cluster.local:80"
   }
   ```

   Response:
   ```protobuf
   message Update {
     repeated WeightedAddr addrs = 1;
   }

   message WeightedAddr {
     Addr addr = 1;  // 10.244.1.5:8080
     uint32 weight = 2;
     TlsIdentity tls_identity = 3;  // "my-app.default.sa..."
   }
   ```
  ↓
5. バックエンドPodを選択（ロードバランシング）
  ↓
6. mTLS接続を確立:

   a) TLS Handshake:
      ```
      ClientHello
        ↓
      ServerHello + Server Certificate
        ↓
      Client Certificate
        ↓
      Certificate Verify
        ↓
      [暗号化通信開始]
      ```

   b) 証明書検証:
      - Subject/SANが期待値と一致するか
      - CAで署名されているか
      - 有効期限内か
  ↓
7. HTTPリクエストを転送
  ↓
8. レスポンスを受け取り、アプリに返す
```

#### 5.2.3 メトリクス収集

```
[linkerd-proxy]
  ↓
1. すべてのリクエストを計測:

   ```rust
   // Rust コード（概念的）
   struct Metrics {
       request_total: Counter,
       request_duration: Histogram,
       response_total: LabeledCounter,  // status_code ラベル付き
   }

   fn handle_request(req: Request) -> Response {
       let start = Instant::now();

       metrics.request_total.inc();

       let response = backend.send(req);

       let duration = start.elapsed();
       metrics.request_duration.observe(duration);

       metrics.response_total
           .with_label("status_code", response.status)
           .inc();

       response
   }
   ```
  ↓
2. Prometheus形式でメトリクスを公開:

   GET :4191/metrics

   ```
   # TYPE request_total counter
   request_total{direction="outbound",target_addr="10.244.1.5:8080"} 1523

   # TYPE response_latency_ms histogram
   response_latency_ms_bucket{le="1"} 245
   response_latency_ms_bucket{le="5"} 892
   response_latency_ms_bucket{le="10"} 1204
   response_latency_ms_bucket{le="+Inf"} 1523
   response_latency_ms_sum 8432.5
   response_latency_ms_count 1523

   # TYPE response_total counter
   response_total{status_code="200"} 1450
   response_total{status_code="500"} 73
   ```
  ↓
3. Prometheus が scrape:

   ```yaml
   scrape_configs:
   - job_name: 'linkerd-proxy'
     kubernetes_sd_configs:
     - role: pod
     relabel_configs:
     - source_labels: [__meta_kubernetes_pod_container_name]
       action: keep
       regex: ^linkerd-proxy$
     - source_labels: [__meta_kubernetes_pod_container_port_name]
       action: keep
       regex: ^linkerd-admin$
   ```
  ↓
4. Linkerd Vizダッシュボードで可視化:

   PromQL クエリ:
   ```
   # Success Rate
   sum(rate(response_total{status_code=~"2.."}[1m]))
   /
   sum(rate(response_total[1m]))

   # P99 Latency
   histogram_quantile(0.99, rate(response_latency_ms_bucket[1m]))
   ```
```

---

### 5.3 Istio - エンタープライズサービスメッシュ

**アーキテクチャ（Ambient Mode）**:
```
[Control Plane]
  istiod
    ├─ xDS Server (Envoy設定配信)
    ├─ CA (証明書管理)
    └─ Webhook (注入)

[Data Plane - Ambient Mode]
  ztunnel (各ノードに1つ、DaemonSet)
    ↓ L4 処理 (mTLS, routing)
  waypoint proxy (オプション、L7処理)
    ↓ L7処理 (HTTP routing, retries, etc)
```

**従来のSidecarモードとの違い**:

```
[Sidecar Mode]
  Pod
  ├─ App Container
  └─ Envoy Sidecar  ← 各Podに1つ

  欠点:
  - Podごとにプロキシのリソース消費
  - アプリの再起動が必要

[Ambient Mode]
  Pod
  └─ App Container (変更なし)

  ノード上:
  ztunnel DaemonSet  ← ノードごとに1つ

  利点:
  - リソース効率的
  - アプリの変更不要
```

**詳細な動作原理**:

#### 5.3.1 Ambient Mode の通信フロー

```
[Pod A から Pod B への通信]
  ↓
1. App: connect(service-b, 80)
  ↓
2. Pod A のネットワーク namespace で redirect:

   eBPF プログラムまたは iptables:
   ```c
   // eBPF プログラム（概念的）
   SEC("cgroup/connect4")
   int connect4_prog(struct bpf_sock_addr *ctx) {
       // ztunnel のアドレスに書き換え
       ctx->user_ip4 = ZTUNNEL_IP;
       ctx->user_port = ZTUNNEL_PORT;
       return 1;
   }
   ```
  ↓
3. ztunnel が接続を受け取る
  ↓
4. ztunnel の処理:

   a) Service Discovery:
      istiod の xDS API に問い合わせ
      ```
      StreamAggregatedResources RPC

      TypeUrl: type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment

      Response:
      {
        "cluster_name": "service-b.default.svc.cluster.local",
        "endpoints": [{
          "lb_endpoints": [{
            "endpoint": {
              "address": {"socket_address": {"address": "10.244.2.3", "port": 8080}}
            },
            "metadata": {
              "filter_metadata": {
                "istio": {
                  "workload": "service-b-v1",
                  "service_account": "service-b"
                }
              }
            }
          }]
        }]
      }
      ```

   b) mTLS接続:
      ztunnel → (mTLS) → ztunnel (Pod B のノード)

   c) パケット転送:
      暗号化されたパケットをネットワーク経由で送信
  ↓
5. Pod B のノードの ztunnel が受信
  ↓
6. mTLS復号化、Pod B に転送
  ↓
7. App B が受信
```

#### 5.3.2 Waypoint Proxy（L7処理）

```
[L7機能が必要な場合]
  ↓
1. Waypoint Proxy をデプロイ:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1beta1
   kind: Gateway
   metadata:
     name: waypoint
     labels:
       istio.io/waypoint-for: service-b
   spec:
     gatewayClassName: istio-waypoint
     listeners:
     - name: mesh
       port: 15008
       protocol: HBONE  # HTTP-Based Overlay Network Environment
   ```
  ↓
2. ztunnel → Waypoint Proxy → ztunnel → Pod

   通信フロー:
   ```
   Pod A
     ↓ (plain HTTP)
   ztunnel A
     ↓ (mTLS + HBONE)
   Waypoint Proxy (Envoy)
     ├─ HTTPルーティング
     ├─ Retry, Timeout
     ├─ Circuit Breaking
     └─ Fault Injection
     ↓ (mTLS)
   ztunnel B
     ↓ (plain HTTP)
   Pod B
   ```
  ↓
3. Envoy (Waypoint) の設定:

   istiod から xDS で配信:
   ```json
   {
     "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
     "name": "service-b",
     "virtual_hosts": [{
       "name": "service-b.default.svc.cluster.local",
       "domains": ["service-b.default.svc.cluster.local"],
       "routes": [{
         "match": {"prefix": "/"},
         "route": {
           "cluster": "outbound|8080||service-b.default.svc.cluster.local",
           "retry_policy": {
             "retry_on": "5xx",
             "num_retries": 3
           },
           "timeout": "10s"
         }
       }]
     }]
   }
   ```
```

#### 5.3.3 Observability (Kiali + Jaeger)

**Distributed Tracing の仕組み**:

```
[リクエストの経路]
  frontend → backend → database

[Trace の生成]
  ↓
1. frontend の Envoy がTraceを開始:

   HTTP Headers:
   ```
   x-request-id: 550e8400-e29b-41d4-a716-446655440000
   x-b3-traceid: 80f198ee56343ba864fe8b2a57d3eff7
   x-b3-spanid: e457b5a2e4d86bd1
   x-b3-sampled: 1
   ```

   Span 作成:
   ```json
   {
     "traceId": "80f198ee56343ba864fe8b2a57d3eff7",
     "spanId": "e457b5a2e4d86bd1",
     "operationName": "frontend.default.svc.cluster.local:80/*",
     "startTime": 1640000000000000,
     "tags": {
       "http.method": "GET",
       "http.url": "/api/products",
       "component": "proxy"
     }
   }
   ```
  ↓
2. backend の Envoy が親Spanを継承:

   新しいSpan:
   ```json
   {
     "traceId": "80f198ee56343ba864fe8b2a57d3eff7",  // 同じTrace ID
     "spanId": "05e3ac9a4f6e3b90",                  // 新しいSpan ID
     "parentSpanId": "e457b5a2e4d86bd1",            // 親Span ID
     "operationName": "backend.default.svc.cluster.local:80/*",
     "startTime": 1640000000100000
   }
   ```
  ↓
3. Jaeger Collector に送信:

   Envoy → (gRPC) → Jaeger Collector
  ↓
4. Jaeger が Trace を再構築:

   ```
   Trace: 80f198ee56343ba864fe8b2a57d3eff7
     Span: e457b5a2e4d86bd1 (frontend) [0ms - 150ms]
       Span: 05e3ac9a4f6e3b90 (backend) [10ms - 120ms]
         Span: 7c5f7f2e9e8d1a3b (database) [15ms - 115ms]
   ```
  ↓
5. Kiali でサービスグラフを可視化:

   ```
   [Frontend] --92%成功--> [Backend] --98%成功--> [Database]
      ↓ 8% 失敗
   [Error]
   ```

   PromQL クエリ:
   ```
   # Service Graph
   sum(rate(istio_requests_total[1m])) by (source_workload, destination_workload)

   # Error Rate
   sum(rate(istio_requests_total{response_code=~"5.."}[1m]))
   /
   sum(rate(istio_requests_total[1m]))
   ```
```

---

続きは次のドキュメントで説明します。
