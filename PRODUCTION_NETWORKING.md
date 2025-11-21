# Kubernetes本番環境 ネットワーク完全ガイド

このドキュメントは、Kubernetes本番環境でのネットワークの実装を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [CNI (Container Network Interface)](#1-cni-container-network-interface)
2. [Service と Load Balancing](#2-service-と-load-balancing)
3. [Ingress Controllers](#3-ingress-controllers)
4. [NetworkPolicy](#4-networkpolicy)
5. [DNS と CoreDNS](#5-dns-と-coredns)
6. [Service Mesh詳細](#6-service-mesh詳細)

---

## 1. CNI (Container Network Interface)

### 1.1 CNI の役割とアーキテクチャ

#### Pod ネットワーキングの要件

```
Kubernetes ネットワークモデル:

1. すべてのPodは一意のIPアドレスを持つ
2. すべてのPodは、NATなしで他のすべてのPodと通信できる
3. すべてのNodeは、NATなしですべてのPodと通信できる
4. PodがNodeからどう見えるかと、Pod自身から見た自分のIPは同じ

┌──────────────────────────────────────────────────────────┐
│ Node 1                                                   │
│ ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│ │ Pod A        │  │ Pod B        │  │ Pod C        │   │
│ │ 10.244.1.10  │  │ 10.244.1.11  │  │ 10.244.1.12  │   │
│ └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│        └──────────────────┼──────────────────┘           │
│                           │                              │
│                    ┌──────▼────────┐                     │
│                    │ CNI Plugin    │                     │
│                    │ (veth pairs)  │                     │
│                    └──────┬────────┘                     │
│                           │                              │
│                    ┌──────▼────────┐                     │
│                    │ Bridge (cni0) │                     │
│                    └──────┬────────┘                     │
│                           │                              │
│                    ┌──────▼────────┐                     │
│                    │ Node eth0     │                     │
│                    │ 192.168.1.10  │                     │
└────────────────────┴───────────────┴──────────────────────┘
                             │
                             │ Underlay Network
                             │
┌────────────────────┬───────▼────────┬──────────────────────┐
│ Node 2             │                │                      │
│ 192.168.1.11       │                │                      │
│ Pod CIDR:          │                │                      │
│ 10.244.2.0/24      │                │                      │
└────────────────────┴────────────────┴──────────────────────┘
```

#### CNI プラグインの動作

```go
// CNI プラグインの疑似コード
type CNIPlugin interface {
    // Podネットワークのセットアップ
    AddNetwork(ctx context.Context, config *NetworkConfig) (*Result, error)

    // Podネットワークの削除
    DeleteNetwork(ctx context.Context, config *NetworkConfig) error

    // 設定確認
    CheckNetwork(ctx context.Context, config *NetworkConfig) error
}

// kubelet がPod作成時に実行
func (k *Kubelet) setupPodNetwork(pod *Pod) error {
    // 1. CNI設定の読み込み
    config := loadCNIConfig("/etc/cni/net.d/10-calico.conflist")

    // 2. CNI ADD コマンド実行
    result, err := cniPlugin.AddNetwork(ctx, &NetworkConfig{
        ContainerID: pod.ID,
        Netns:       pod.NetworkNamespace,
        IfName:      "eth0",
        Args:        pod.Annotations,
    })

    // 3. 結果を取得
    // result.IPs[0].Address = "10.244.1.10/24"
    // result.Routes[0].Dst = "0.0.0.0/0"
    // result.Routes[0].GW = "10.244.1.1"

    return err
}
```

### 1.2 CNI プラグイン比較

#### Calico

```yaml
# Calico のインストール（Operator方式）
---
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # CNI プラグイン設定
  calicoNetwork:
    # IPプール
    ipPools:
    - name: default-ipv4-ippool
      cidr: 10.244.0.0/16
      encapsulation: VXLAN  # VXLAN, IPIP, None
      natOutgoing: Enabled
      nodeSelector: all()

  # ネットワークポリシー
  variant: Calico  # Calico または Tigera Secure

  # コンポーネント設定
  componentResources:
  - componentName: Node
    resourceRequirements:
      requests:
        cpu: 250m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

---
# BGP Configuration（IPIP/VXLANなしの場合）
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true  # Full mesh BGP
  asNumber: 64512

---
# eBPF データプレーン（高性能）
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    linuxDataplane: BPF  # eBPFモード
    hostPorts: Enabled
```

**Calico の特徴:**
- **データプレーン**: iptables または eBPF
- **ルーティング**: BGP（デフォルト）、VXLAN、IPIP
- **NetworkPolicy**: フル機能サポート
- **パフォーマンス**: 高速（特にeBPFモード）
- **スケール**: 大規模クラスター対応（数千ノード）

#### Cilium

```yaml
# Cilium のインストール（Helm）
# helm install cilium cilium/cilium --version 1.14.0 \
#   --namespace kube-system \
#   --set ipam.mode=kubernetes \
#   --set tunnel=vxlan \
#   --set bpf.masquerade=true

---
# CiliumConfig CRD
apiVersion: cilium.io/v2alpha1
kind: CiliumConfig
metadata:
  name: cilium-config
spec:
  # eBPFベースのデータプレーン
  bpf:
    masquerade: true
    tproxy: true
    hostLegacyRouting: false

  # IPAM設定
  ipam:
    mode: kubernetes
    operator:
      clusterPoolIPv4PodCIDRList:
      - 10.244.0.0/16

  # Hubble（可観測性）
  hubble:
    enabled: true
    relay:
      enabled: true
    ui:
      enabled: true

  # Bandwidth Manager（帯域制御）
  bandwidthManager: true

  # Host Firewall
  hostFirewall: true

  # kube-proxy replacement
  kubeProxyReplacement: strict  # iptables不要
```

**Cilium の特徴:**
- **データプレーン**: 完全eBPF（kube-proxy不要）
- **可観測性**: Hubble（フローログ、サービスマップ）
- **セキュリティ**: L7 NetworkPolicy、Identity-based
- **パフォーマンス**: 最高速（eBPF JIT）
- **機能**: Service Mesh（Cilium Service Mesh）

#### CNI比較表

| CNI | データプレーン | カプセル化 | NetworkPolicy | kube-proxy replacement | 可観測性 | 学習曲線 |
|-----|-------------|----------|--------------|----------------------|---------|---------|
| **Calico** | iptables / eBPF | VXLAN/IPIP/BGP | フル | 部分的 | 標準 | 緩やか |
| **Cilium** | eBPF | VXLAN/Geneve | フル+L7 | 完全 | Hubble | 急 |
| **Flannel** | iptables | VXLAN/host-gw | なし | なし | なし | 非常に緩やか |
| **Weave** | iptables | VXLAN | 基本 | なし | WeaveScope | 緩やか |
| **Canal** | iptables | VXLAN | フル | なし | 標準 | 緩やか |

### 1.3 カプセル化方式の選択

```
┌─────────────────────────────────────────────────────────┐
│ VXLAN (Virtual eXtensible LAN)                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Original Packet                             │
│ ┌────────┬────────┬──────────────┐         │
│ │ Eth    │ IP     │ Payload      │         │
│ │ Header │ Header │              │         │
│ └────────┴────────┴──────────────┘         │
└─────────────────────────────────────────────┘
                  │
                  ▼ VXLAN カプセル化
┌─────────────────────────────────────────────┐
│ VXLAN Packet                                │
│ ┌───────┬────────┬─────────┬──────────────┐│
│ │Outer  │Outer   │VXLAN    │Original      ││
│ │Eth    │IP      │Header   │Packet        ││
│ │Header │Header  │(VNI)    │              ││
│ └───────┴────────┴─────────┴──────────────┘│
└─────────────────────────────────────────────┘

利点:
- L2 over L3（異なるサブネット間でもL2通信）
- クラウド環境で動作（AWS VPC等）
- 最大1600万のネットワークセグメント

欠点:
- オーバーヘッド（50バイト追加）
- パフォーマンス低下（~5-10%）


┌─────────────────────────────────────────────────────────┐
│ BGP (Border Gateway Protocol)                           │
└─────────────────────────────────────────────────────────┘

Node 1                          Node 2
┌─────────────┐  BGP Session  ┌─────────────┐
│ 10.244.1.0/24├──────────────►│10.244.2.0/24│
│ via 192.168.1.10              │via 192.168.1.11
└─────────────┘                └─────────────┘

BGP Advertisement:
"10.244.1.0/24にルーティングするには192.168.1.10に送信せよ"

利点:
- カプセル化なし（ネイティブルーティング）
- 最高のパフォーマンス
- 既存のネットワーク機器と統合可能

欠点:
- ネットワーク設定が必要
- クラウド環境では制限あり（VPC Peering等が必要）
```

---

## 2. Service と Load Balancing

### 2.1 kube-proxy のモード

#### iptables モード

```bash
# kube-proxy が生成する iptables ルール

# Service: myapp (ClusterIP: 10.96.100.200:8080)
# Endpoints: 10.244.1.10:8080, 10.244.1.11:8080, 10.244.1.12:8080

# 1. KUBE-SERVICES チェーン
-A KUBE-SERVICES -d 10.96.100.200/32 -p tcp -m tcp --dport 8080 \
   -j KUBE-SVC-XXXXXX

# 2. KUBE-SVC-XXXXXX チェーン（ロードバランシング）
# 1/3の確率で最初のEndpoint
-A KUBE-SVC-XXXXXX -m statistic --mode random --probability 0.33333 \
   -j KUBE-SEP-AAAA

# 1/2の確率で2番目のEndpoint（残り2つのうち）
-A KUBE-SVC-XXXXXX -m statistic --mode random --probability 0.50000 \
   -j KUBE-SEP-BBBB

# 残りは3番目のEndpoint
-A KUBE-SVC-XXXXXX -j KUBE-SEP-CCCC

# 3. KUBE-SEP-AAAA チェーン（DNAT）
-A KUBE-SEP-AAAA -p tcp -m tcp \
   -j DNAT --to-destination 10.244.1.10:8080

-A KUBE-SEP-BBBB -p tcp -m tcp \
   -j DNAT --to-destination 10.244.1.11:8080

-A KUBE-SEP-CCCC -p tcp -m tcp \
   -j DNAT --to-destination 10.244.1.12:8080
```

**iptables モードの問題:**
- Serviceが多いとルール数が膨大（O(n²)）
- ルール更新が遅い
- ロードバランシングがランダムのみ

#### IPVS モード

```bash
# kube-proxy IPVS モード設定
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  scheduler: rr  # round-robin
  # 他のオプション: lc (least connection), dh (destination hashing), sh (source hashing)
  strictARP: true
  tcpTimeout: 0s
  tcpFinTimeout: 0s
  udpTimeout: 0s

# IPVS の確認
ipvsadm -Ln

# 出力例:
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
# TCP  10.96.100.200:8080 rr
#   -> 10.244.1.10:8080             Masq    1      0          0
#   -> 10.244.1.11:8080             Masq    1      0          0
#   -> 10.244.1.12:8080             Masq    1      0          0
```

**IPVS モードの利点:**
- スケーラブル（O(1)のルックアップ）
- 豊富なロードバランシングアルゴリズム
- より良いパフォーマンス
- Connection追跡

#### eBPF モード（Cilium）

```yaml
# Cilium kube-proxy replacement
apiVersion: cilium.io/v2alpha1
kind: CiliumConfig
metadata:
  name: cilium-config
spec:
  kubeProxyReplacement: strict

  # ロードバランシングアルゴリズム
  loadBalancer:
    algorithm: maglev  # または random
    mode: dsr          # Direct Server Return

  # SessionAffinity
  sessionAffinity: true

  # NodePort設定
  nodePort:
    enabled: true
    mode: dsr

  # External IPs
  externalIPs:
    enabled: true
```

**eBPF モードの利点:**
- カーネル空間で処理（最高速）
- iptables/IPVS不要
- より詳細な可観測性
- Maglev（一貫性ハッシュ）対応

### 2.2 Service Types

#### ClusterIP

```yaml
# ClusterIP - クラスター内部のみアクセス可能
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080

  # SessionAffinity（同じクライアントを同じPodに）
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3時間
```

#### NodePort

```yaml
# NodePort - すべてのNodeの特定ポートで公開
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767の範囲

  # External Traffic Policy
  externalTrafficPolicy: Local  # または Cluster（デフォルト）

# externalTrafficPolicy の違い:
#
# Cluster（デフォルト）:
# - すべてのNodeで受信可能
# - クライアントIPが失われる（SNAT）
# - クロスノードのトラフィック発生
#
# Local:
# - そのNodeにPodがある場合のみ受信
# - クライアントIPが保持される
# - クロスノードトラフィックなし
# - 負荷分散が不均等になる可能性
```

#### LoadBalancer

```yaml
# LoadBalancer - クラウドプロバイダーのLBを作成
apiVersion: v1
kind: Service
metadata:
  name: web
  annotations:
    # AWS
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:region:account-id:certificate/xxxxx
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"

    # GCP
    # cloud.google.com/load-balancer-type: "Internal"

    # Azure
    # service.beta.kubernetes.io/azure-load-balancer-internal: "true"

spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8080

  loadBalancerSourceRanges:  # アクセス元IP制限
  - 203.0.113.0/24
  - 198.51.100.0/24
```

#### Headless Service

```yaml
# Headless Service - Podに直接アクセス（StatefulSet等）
apiVersion: v1
kind: Service
metadata:
  name: cassandra
spec:
  clusterIP: None  # Headless
  selector:
    app: cassandra
  ports:
  - port: 9042

# DNSレコード:
# cassandra.default.svc.cluster.local
#   → 10.244.1.10  (cassandra-0.cassandra.default.svc.cluster.local)
#   → 10.244.1.11  (cassandra-1.cassandra.default.svc.cluster.local)
#   → 10.244.1.12  (cassandra-2.cassandra.default.svc.cluster.local)

# StatefulSet との組み合わせ:
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
spec:
  serviceName: cassandra  # ← Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: cassandra
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      containers:
      - name: cassandra
        image: cassandra:4.1
        env:
        - name: CASSANDRA_SEEDS
          value: "cassandra-0.cassandra.default.svc.cluster.local"
```

---

## 3. Ingress Controllers

### 3.1 Nginx Ingress Controller

#### インストール

```bash
# Helm でインストール
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=3 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.admissionWebhooks.enabled=true \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.config.use-forwarded-headers="true" \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"="true" \
  --set controller.podAnnotations."prometheus\.io/port"="10254"
```

#### Ingress リソース

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: prod
  annotations:
    # SSL Redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Rate Limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "10"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"

    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Client Body Size
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # Canary（A/Bテスト）
    # nginx.ingress.kubernetes.io/canary: "true"
    # nginx.ingress.kubernetes.io/canary-weight: "20"

spec:
  ingressClassName: nginx

  # TLS設定
  tls:
  - hosts:
    - myapp.example.com
    - api.myapp.example.com
    secretName: myapp-tls  # cert-manager が自動生成

  rules:
  # ホスト1: メインアプリケーション
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80

      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080

  # ホスト2: API専用
  - host: api.myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080
```

#### ConfigMap によるグローバル設定

```yaml
# ingress-nginx ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  # クライアントボディサイズ
  proxy-body-size: "50m"

  # タイムアウト
  proxy-connect-timeout: "60"
  proxy-read-timeout: "60"
  proxy-send-timeout: "60"

  # KeepAlive
  keep-alive: "75"
  keep-alive-requests: "100"

  # gzip圧縮
  use-gzip: "true"
  gzip-level: "5"
  gzip-types: "text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript"

  # HTTP/2
  use-http2: "true"

  # Real IP
  use-forwarded-headers: "true"
  compute-full-forwarded-for: "true"
  use-proxy-protocol: "false"

  # Access Log
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] [$proxy_alternative_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id'

  # SSL
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"
  ssl-prefer-server-ciphers: "true"

  # HSTS
  hsts: "true"
  hsts-max-age: "31536000"
  hsts-include-subdomains: "true"

  # バッファサイズ
  proxy-buffer-size: "16k"
  proxy-buffers-number: "4"
```

### 3.2 cert-manager との統合

```yaml
# ClusterIssuer - Let's Encrypt
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key

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
          accessKeyID: AKIAIOSFODNN7EXAMPLE
          secretAccessKeySecretRef:
            name: route53-credentials
            key: secret-access-key

---
# Certificate自動発行
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls  # cert-manager が自動作成
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
              number: 80
```

### 3.3 mTLS (Mutual TLS)

```yaml
# Ingress でのクライアント証明書認証
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-api
  annotations:
    # クライアント証明書検証
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "prod/client-ca"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: server-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080

---
# Client CA Secret
apiVersion: v1
kind: Secret
metadata:
  name: client-ca
  namespace: prod
type: Opaque
data:
  ca.crt: <base64 encoded CA certificate>
```

---

## 4. NetworkPolicy

### 4.1 Default Deny Policy

```yaml
# すべての Ingress/Egress をデフォルト拒否
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}  # すべてのPod
  policyTypes:
  - Ingress
  - Egress
  # ingress/egress が空 = すべて拒否
```

### 4.2 アプリケーション間通信の制御

```yaml
# Frontend → Backend のみ許可
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
  - Ingress

  ingress:
  # frontendからのトラフィックのみ許可
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Backend → Database のみ許可
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-allow-backend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: database

  policyTypes:
  - Ingress

  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```

### 4.3 Namespace 間の通信制御

```yaml
# Monitoring Namespace から全Namespace へのアクセスを許可
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: prod
spec:
  podSelector: {}

  policyTypes:
  - Ingress

  ingress:
  # monitoring namespaceからのアクセスを許可
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090  # Prometheusメトリクス
```

### 4.4 Egress制御（外部アクセス制限）

```yaml
# 特定の外部APIのみ許可
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: myapp

  policyTypes:
  - Egress

  egress:
  # DNS（kube-dns/CoreDNS）への通信を許可
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # 特定の外部IP（APIサーバー等）への通信を許可
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # 外部APIのCIDR
    ports:
    - protocol: TCP
      port: 443

  # クラスター内の特定Serviceへの通信を許可
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
```

---

## 5. DNS と CoreDNS

### 5.1 Kubernetes DNS の仕組み

```
DNS レコードの構造:

Service:
<service-name>.<namespace>.svc.<cluster-domain>

例:
myapp.prod.svc.cluster.local → 10.96.100.200 (ClusterIP)

Headless Service + StatefulSet:
<pod-name>.<service-name>.<namespace>.svc.<cluster-domain>

例:
cassandra-0.cassandra.prod.svc.cluster.local → 10.244.1.10
cassandra-1.cassandra.prod.svc.cluster.local → 10.244.1.11

Pod（オプション）:
<pod-ip-with-dashes>.<namespace>.pod.<cluster-domain>

例:
10-244-1-10.prod.pod.cluster.local → 10.244.1.10
```

### 5.2 CoreDNS 設定

```yaml
# CoreDNS ConfigMap
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

        # Kubernetes プラグイン
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          ttl 30
        }

        # Prometheus メトリクス
        prometheus :9153

        # 外部DNSへのフォワーディング
        forward . /etc/resolv.conf {
          max_concurrent 1000
          policy random
        }

        # キャッシュ
        cache 30

        # ループ検知
        loop

        # 設定のリロード
        reload

        # ロードバランス
        loadbalance round_robin
    }

    # カスタムドメイン
    example.com:53 {
        errors
        cache 30
        forward . 10.0.0.10 10.0.0.11  # 社内DNSサーバー
    }

    # 特定ドメインのスタブ設定
    consul.local:53 {
        errors
        cache 30
        forward . 10.0.1.100:8600
    }
```

### 5.3 Pod の DNS 設定

```yaml
# Pod レベルでのDNS設定カスタマイズ
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns
spec:
  # DNS Policy
  dnsPolicy: None  # Default, ClusterFirst, ClusterFirstWithHostNet, None

  # カスタムDNS設定
  dnsConfig:
    nameservers:
    - 10.96.0.10      # CoreDNS
    - 8.8.8.8         # Google DNS（fallback）

    searches:
    - prod.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    - example.com     # カスタムドメイン

    options:
    - name: ndots
      value: "2"      # ドットが2個未満ならsearch domainsを試す
    - name: edns0     # EDNS0サポート
    - name: timeout
      value: "1"
    - name: attempts
      value: "3"

  containers:
  - name: app
    image: myapp:1.0
```

このドキュメントは、Kubernetesネットワークの主要部分をカバーしています。

## 📚 参考リソース

- [Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [CNI Specification](https://github.com/containernetworking/cni)
- [Calico Documentation](https://docs.tigera.io/calico/latest/about/)
- [Cilium Documentation](https://docs.cilium.io/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [NetworkPolicy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)
