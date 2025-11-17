# Kubernetes本番環境 可観測性（Observability）完全ガイド

このドキュメントは、Kubernetes本番環境で必須となる可観測性の実装を、技術的原理から実践まで徹底的に解説します。

## 📚 目次

1. [ログ管理](#1-ログ管理)
2. [メトリクス](#2-メトリクス)
3. [分散トレーシング](#3-分散トレーシング)
4. [アラート設計](#4-アラート設計)
5. [ダッシュボード](#5-ダッシュボード)

---

## 1. ログ管理

### 1.1 ログ集約アーキテクチャ

#### アーキテクチャ比較

```
┌────────────────────────────────────────────────────────────┐
│ Option 1: EFK Stack (Elasticsearch + Fluent Bit + Kibana) │
└────────────────────────────────────────────────────────────┘

Kubernetes Cluster
┌─────────────────────────────────────────────────────────┐
│ Node 1                        Node 2                    │
│ ┌──────────┐                 ┌──────────┐              │
│ │ Pod Logs │                 │ Pod Logs │              │
│ │ stdout   │                 │ stdout   │              │
│ └────┬─────┘                 └────┬─────┘              │
│      │ JSON logs                  │                    │
│      ▼                            ▼                    │
│ ┌─────────────────┐         ┌─────────────────┐        │
│ │ Fluent Bit      │         │ Fluent Bit      │        │
│ │ (DaemonSet)     │         │ (DaemonSet)     │        │
│ │ ・ログ収集      │         │ ・パース        │        │
│ │ ・フィルタリング│         │ ・エンリッチ    │        │
│ │ ・軽量(~15MB)   │         │                 │        │
│ └────┬────────────┘         └────┬────────────┘        │
│      │                           │                     │
└──────┼───────────────────────────┼─────────────────────┘
       │                           │
       │ Forward                   │
       ▼                           ▼
┌─────────────────────────────────────────────┐
│ Elasticsearch Cluster                       │
│ ┌───────────┐ ┌───────────┐ ┌───────────┐  │
│ │ Master    │ │ Data      │ │ Data      │  │
│ │ Node      │ │ Node      │ │ Node      │  │
│ └───────────┘ └───────────┘ └───────────┘  │
│ ・インデックス作成                          │
│ ・全文検索                                  │
│ ・アグリゲーション                          │
└─────────────────┬───────────────────────────┘
                  │ Query
                  ▼
┌─────────────────────────────────────────────┐
│ Kibana                                      │
│ ・ログ検索UI                                │
│ ・ダッシュボード                            │
│ ・アラート                                  │
└─────────────────────────────────────────────┘


┌────────────────────────────────────────────────────────────┐
│ Option 2: Loki Stack (Loki + Promtail + Grafana)          │
└────────────────────────────────────────────────────────────┘

Kubernetes Cluster
┌─────────────────────────────────────────────────────────┐
│ Node 1                        Node 2                    │
│ ┌──────────┐                 ┌──────────┐              │
│ │ Pod Logs │                 │ Pod Logs │              │
│ └────┬─────┘                 └────┬─────┘              │
│      │                            │                    │
│      ▼                            ▼                    │
│ ┌─────────────────┐         ┌─────────────────┐        │
│ │ Promtail        │         │ Promtail        │        │
│ │ (DaemonSet)     │         │ (DaemonSet)     │        │
│ │ ・ラベル抽出    │         │ ・ログ行のみ送信│        │
│ │ ・軽量          │         │ (インデックスなし)       │
│ └────┬────────────┘         └────┬────────────┘        │
│      │                           │                     │
└──────┼───────────────────────────┼─────────────────────┘
       │ Push (gRPC)               │
       ▼                           ▼
┌─────────────────────────────────────────────┐
│ Loki (StatefulSet)                          │
│ ┌──────────────────────────────────────┐    │
│ │ Distributor → Ingester → Querier    │    │
│ │ ・ログを圧縮保存                     │    │
│ │ ・ラベルのみインデックス             │    │
│ │ ・ログ本文は圧縮してS3/GCS等に保存  │    │
│ └──────────────────────────────────────┘    │
│ コスト: Elasticsearchの1/10以下             │
└─────────────────┬───────────────────────────┘
                  │ LogQL Query
                  ▼
┌─────────────────────────────────────────────┐
│ Grafana                                     │
│ ・Lokiデータソース統合                      │
│ ・メトリクスとログの相関                    │
└─────────────────────────────────────────────┘
```

#### EFK vs Loki 比較表

| 観点 | EFK (Elasticsearch) | Loki |
|------|---------------------|------|
| **インデックス** | 全文インデックス | ラベルのみインデックス |
| **ストレージコスト** | 高い | 低い（1/10程度） |
| **検索速度** | 高速（全文検索） | ラベル検索は高速、全文検索は遅い |
| **リソース使用量** | 大（特にメモリ） | 小 |
| **スケーラビリティ** | 水平スケール可能 | 水平スケール可能 |
| **保持期間** | コスト的に短め(30-90日) | 長期可能(1年+) |
| **学習曲線** | 急（Lucene Query） | 緩やか（LogQL ≒ PromQL） |
| **適用ケース** | 複雑な検索クエリが必要 | コスト重視、Prometheus併用 |

### 1.2 Fluent Bit の詳細実装

#### Fluent Bit のアーキテクチャ

```
Fluent Bit パイプライン:

┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   INPUT     │ →  │   PARSER     │ →  │   FILTER    │
│ (ログ収集)  │    │ (パース)     │    │ (変換/追加)  │
└─────────────┘    └──────────────┘    └─────────────┘
     ↓                                        ↓
     └────────────────────────────────────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │   BUFFER    │
                  │ (メモリ/FS)  │
                  └─────────────┘
                         │
                         ▼
                  ┌─────────────┐
                  │   OUTPUT    │
                  │ (送信先)     │
                  └─────────────┘
```

#### Fluent Bit ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Daemon        off
        Log_Level     info
        Parsers_File  parsers.conf

    # INPUT: Kubernetesコンテナログの収集
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            cri  # containerd/CRI-O形式
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        DB                /var/log/flb-kube.db  # ポジション記録

    # INPUT: systemd journal（kubelet, dockerログ等）
    [INPUT]
        Name            systemd
        Tag             systemd.*
        Read_From_Tail  On
        Strip_Underscores On

    # FILTER: Kubernetes メタデータの追加
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On    # JSONログをパース
        Keep_Log            Off   # 元のlog fieldは削除
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On
        Labels              On    # Pod labels を追加
        Annotations         Off   # Annotations は除外（データ量削減）

    # FILTER: ログレベルの抽出（構造化ログ）
    [FILTER]
        Name    parser
        Match   kube.*
        Key_Name log
        Parser   json-log
        Reserve_Data On  # パース失敗時も元データ保持

    # FILTER: 不要なログの除外
    [FILTER]
        Name    grep
        Match   kube.*
        Exclude kubernetes_namespace_name kube-system
        Exclude log healthcheck

    # FILTER: フィールド名の変更
    [FILTER]
        Name       modify
        Match      kube.*
        Rename     log message
        Add        cluster_name production-eks
        Remove     stream

    # OUTPUT: Elasticsearch
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch.logging.svc
        Port            9200
        Index           kubernetes
        Type            _doc
        Logstash_Format On
        Logstash_Prefix kubernetes
        Logstash_DateFormat %Y.%m.%d
        Retry_Limit     5
        Buffer_Size     False  # チャンクごとに送信
        Trace_Error     On

    # OUTPUT: Loki (オプション)
    [OUTPUT]
        Name        loki
        Match       kube.*
        Host        loki.logging.svc
        Port        3100
        Labels      job=fluentbit, cluster=production
        Label_keys  $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']

  parsers.conf: |
    # CRI形式のパーサー（containerd/CRI-O）
    [PARSER]
        Name        cri
        Format      regex
        Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    # JSON形式のアプリケーションログ
    [PARSER]
        Name        json-log
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        Time_Keep   On

    # Nginx アクセスログ
    [PARSER]
        Name        nginx
        Format      regex
        Regex       ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")
        Time_Key    time
        Time_Format %d/%b/%Y:%H:%M:%S %z
```

#### Fluent Bit DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      # マスターノードにも配置
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule

      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi

        volumeMounts:
        # ログファイルの読み取り
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        # 設定ファイル
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        # ポジションDB（再起動時の重複防止）
        - name: position-db
          mountPath: /var/log

      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      - name: position-db
        emptyDir: {}

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  - pods/logs
  verbs:
  - get
  - list
  - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: logging
```

### 1.3 構造化ログのベストプラクティス

#### アプリケーションでの構造化ログ実装

```go
// Go での構造化ログ例（zap）
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func main() {
    // Production用の設定
    config := zap.NewProductionConfig()
    config.EncoderConfig.TimeKey = "timestamp"
    config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

    logger, _ := config.Build()
    defer logger.Sync()

    // 構造化ログ出力
    logger.Info("User login",
        zap.String("user_id", "12345"),
        zap.String("ip_address", "192.168.1.100"),
        zap.Duration("response_time", 123*time.Millisecond),
        zap.Int("status_code", 200),
    )

    // エラーログ
    logger.Error("Database connection failed",
        zap.String("database", "postgres"),
        zap.String("host", "db.example.com"),
        zap.Error(err),
        zap.Stack("stacktrace"),  # スタックトレースを含める
    )
}

// 出力JSON:
// {
//   "level": "info",
//   "timestamp": "2024-01-15T10:30:00.123Z",
//   "msg": "User login",
//   "user_id": "12345",
//   "ip_address": "192.168.1.100",
//   "response_time": 0.123,
//   "status_code": 200
// }
```

```python
# Python での構造化ログ例（structlog）
import structlog

# ロガーの設定
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

log = structlog.get_logger()

# 構造化ログ出力
log.info("user_login",
         user_id="12345",
         ip_address="192.168.1.100",
         response_time=0.123,
         status_code=200)

# エラーログ
try:
    connect_to_database()
except Exception as e:
    log.error("database_connection_failed",
              database="postgres",
              host="db.example.com",
              exc_info=True)  # 例外情報を含める
```

#### 標準化されたログフィールド

```json
{
  // 必須フィールド
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "info",                          // debug, info, warn, error, fatal
  "message": "User authentication successful",

  // アプリケーション識別
  "service": "auth-service",
  "version": "1.2.3",
  "environment": "production",

  // Kubernetes コンテキスト（Fluent Bitが自動追加）
  "kubernetes": {
    "namespace_name": "prod",
    "pod_name": "auth-service-7d9f8b-xyz",
    "container_name": "auth",
    "labels": {
      "app": "auth-service",
      "version": "v1.2.3"
    }
  },

  // リクエストコンテキスト
  "request": {
    "id": "req-12345",                      // Trace ID
    "method": "POST",
    "path": "/api/v1/login",
    "ip": "192.168.1.100",
    "user_agent": "Mozilla/5.0..."
  },

  // ユーザーコンテキスト
  "user": {
    "id": "user-67890",
    "email": "user@example.com"
  },

  // パフォーマンス
  "performance": {
    "duration_ms": 123,
    "db_query_ms": 45,
    "cache_hit": true
  },

  // エラー情報（エラーログの場合）
  "error": {
    "type": "DatabaseConnectionError",
    "message": "Connection timeout",
    "stack_trace": "...",
    "cause": "Network unreachable"
  }
}
```

---

## 2. メトリクス

### 2.1 Prometheus Operator の詳細

#### Prometheus Operator のアーキテクチャ

```
┌────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                     │
│                                                         │
│ ┌────────────────────────────────────────────────┐     │
│ │ Custom Resource Definitions (CRDs)             │     │
│ │ ├─ Prometheus                                  │     │
│ │ ├─ ServiceMonitor                              │     │
│ │ ├─ PodMonitor                                  │     │
│ │ ├─ PrometheusRule                              │     │
│ │ └─ Alertmanager                                │     │
│ └────────────────────────────────────────────────┘     │
│                      │                                  │
│                      │ watch                            │
│                      ▼                                  │
│ ┌────────────────────────────────────────────────┐     │
│ │ Prometheus Operator (Deployment)               │     │
│ │ ┌────────────────────────────────────────────┐ │     │
│ │ │ Controller Loop:                           │ │     │
│ │ │ 1. Watch CRs                               │ │     │
│ │ │ 2. Generate Prometheus config              │ │     │
│ │ │ 3. Create/Update StatefulSet               │ │     │
│ │ │ 4. Mount ConfigMap/Secrets                 │ │     │
│ │ └────────────────────────────────────────────┘ │     │
│ └────────────────────────────────────────────────┘     │
│                      │                                  │
│                      │ creates/updates                  │
│                      ▼                                  │
│ ┌────────────────────────────────────────────────┐     │
│ │ Prometheus StatefulSet                         │     │
│ │ ┌──────────────────────────────────────────┐   │     │
│ │ │ prometheus-0                             │   │     │
│ │ │ ├─ Config from operator                  │   │     │
│ │ │ ├─ PersistentVolume (TSDB)               │   │     │
│ │ │ └─ Scrape targets from ServiceMonitors   │   │     │
│ │ └──────────────────────────────────────────┘   │     │
│ └────────────────────────────────────────────────┘     │
│                      │                                  │
│                      │ scrape (pull)                    │
│                      ▼                                  │
│ ┌────────────────────────────────────────────────┐     │
│ │ Application Pods                               │     │
│ │ ┌──────────────────┐  ┌──────────────────┐    │     │
│ │ │ myapp-1          │  │ myapp-2          │    │     │
│ │ │ :8080/metrics    │  │ :8080/metrics    │    │     │
│ │ └──────────────────┘  └──────────────────┘    │     │
│ └────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘
```

#### Prometheus CR（Custom Resource）の定義

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: main
  namespace: monitoring
spec:
  # レプリカ数（HA構成）
  replicas: 2

  # データ保持期間
  retention: 30d
  retentionSize: "100GB"

  # リソース設定
  resources:
    requests:
      memory: 2Gi
      cpu: 1000m
    limits:
      memory: 4Gi
      cpu: 2000m

  # ストレージ
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: fast-ssd
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi

  # ServiceMonitor の選択
  serviceMonitorSelector:
    matchLabels:
      prometheus: main

  # PodMonitor の選択
  podMonitorSelector:
    matchLabels:
      prometheus: main

  # PrometheusRule の選択
  ruleSelector:
    matchLabels:
      prometheus: main

  # Alertmanager 連携
  alerting:
    alertmanagers:
    - namespace: monitoring
      name: main
      port: web

  # 外部ラベル（Thanos連携等で使用）
  externalLabels:
    cluster: production-eks
    region: ap-northeast-1

  # セキュリティ設定
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000

  # Service Account
  serviceAccountName: prometheus
```

### 2.2 ServiceMonitor と PodMonitor

#### ServiceMonitor: Service経由でのスクレイプ

```yaml
# 1. アプリケーションのService
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: prod
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
  - name: web
    port: 8080
  - name: metrics  # メトリクスエンドポイント
    port: 9090

---
# 2. ServiceMonitor の定義
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: prod
  labels:
    prometheus: main  # Prometheus CRのselectorにマッチ
spec:
  # 監視対象Serviceの選択
  selector:
    matchLabels:
      app: myapp

  # Namespace の選択（省略時は同一namespace）
  namespaceSelector:
    matchNames:
    - prod
    - staging

  # スクレイプ設定
  endpoints:
  - port: metrics        # Serviceのport名
    interval: 30s        # スクレイプ間隔
    path: /metrics       # メトリクスパス
    scheme: http

    # TLS設定（オプション）
    # tlsConfig:
    #   insecureSkipVerify: false
    #   ca:
    #     secret:
    #       name: myapp-tls
    #       key: ca.crt

    # Basic認証（オプション）
    # basicAuth:
    #   username:
    #     name: myapp-metrics-auth
    #     key: username
    #   password:
    #     name: myapp-metrics-auth
    #     key: password

    # メトリクスの再ラベリング
    relabelings:
    # ラベルの追加
    - sourceLabels: [__meta_kubernetes_pod_node_name]
      targetLabel: node

    # 不要なラベルの削除
    - action: labeldrop
      regex: __meta_kubernetes_pod_label_pod_template_hash

    # メトリクス値の変換
    metricRelabelings:
    # 特定メトリクスの除外
    - sourceLabels: [__name__]
      regex: 'go_.*'       # Go runtime メトリクスを除外
      action: drop
```

#### PodMonitor: Pod直接スクレイプ

```yaml
# PodMonitor: Serviceを経由せずPodから直接メトリクス取得
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pod-monitor
  namespace: prod
  labels:
    prometheus: main
spec:
  # 監視対象Podの選択
  selector:
    matchLabels:
      app: myapp

  # スクレイプ設定
  podMetricsEndpoints:
  - port: metrics        # Pod の container port 名
    interval: 15s
    path: /metrics

  # Namespace の選択
  namespaceSelector:
    matchNames:
    - prod

# PodMonitor vs ServiceMonitor の使い分け:
# ServiceMonitor:
#   ・標準的な用途
#   ・Serviceを経由するためロードバランスされる
#   ・Serviceが存在する場合
#
# PodMonitor:
#   ・StatefulSet等、Pod個別の監視が必要な場合
#   ・Headless Serviceの場合
#   ・DaemonSet（各Nodeのメトリクスを個別取得）
```

### 2.3 Recording Rules と Alerting Rules

#### Recording Rules: 事前集計

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: recording-rules
  namespace: monitoring
  labels:
    prometheus: main
spec:
  groups:
  # グループ1: HTTP メトリクスの集計
  - name: http_metrics
    interval: 30s  # 評価間隔
    rules:
    # 1. アプリケーションごとのQPS
    - record: job:http_requests_total:rate5m
      expr: |
        sum(rate(http_requests_total[5m])) by (job)

    # 2. エンドポイントごとのレイテンシ（99パーセンタイル）
    - record: job:http_request_duration_seconds:p99
      expr: |
        histogram_quantile(0.99,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
        )

    # 3. エラー率
    - record: job:http_requests:error_rate5m
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
        /
        sum(rate(http_requests_total[5m])) by (job)

  # グループ2: リソース使用率
  - name: resource_usage
    interval: 1m
    rules:
    # CPU使用率（Podごと）
    - record: namespace_pod:container_cpu_usage:sum
      expr: |
        sum(rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        by (namespace, pod)

    # メモリ使用率
    - record: namespace_pod:container_memory_usage:sum
      expr: |
        sum(container_memory_working_set_bytes{container!=""})
        by (namespace, pod)

    # ディスクI/O
    - record: node:disk_io_time_seconds:rate5m
      expr: |
        rate(node_disk_io_time_seconds_total[5m])
```

#### Alerting Rules: アラート定義

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: alerting-rules
  namespace: monitoring
  labels:
    prometheus: main
spec:
  groups:
  # グループ1: アプリケーションアラート
  - name: application_alerts
    interval: 30s
    rules:
    # 高エラー率
    - alert: HighErrorRate
      expr: |
        job:http_requests:error_rate5m > 0.05
      for: 5m  # 5分間継続したらアラート
      labels:
        severity: warning
        team: backend
      annotations:
        summary: "High error rate detected"
        description: |
          Error rate is {{ $value | humanizePercentage }} for {{ $labels.job }}
          (threshold: 5%)

    # 高レイテンシ
    - alert: HighLatency
      expr: |
        job:http_request_duration_seconds:p99 > 1.0
      for: 10m
      labels:
        severity: warning
        team: backend
      annotations:
        summary: "High latency detected"
        description: |
          99th percentile latency is {{ $value }}s for {{ $labels.job }}
          (threshold: 1s)

    # サービスダウン
    - alert: ServiceDown
      expr: |
        up{job="myapp"} == 0
      for: 1m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "Service is down"
        description: |
          {{ $labels.job }} on {{ $labels.instance }} is down

  # グループ2: インフラアラート
  - name: infrastructure_alerts
    interval: 1m
    rules:
    # ノードダウン
    - alert: NodeDown
      expr: |
        up{job="node-exporter"} == 0
      for: 5m
      labels:
        severity: critical
        team: infrastructure
      annotations:
        summary: "Node is down"
        description: "Node {{ $labels.instance }} is down"

    # 高CPU使用率
    - alert: HighCPUUsage
      expr: |
        (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) > 0.8
      for: 10m
      labels:
        severity: warning
        team: infrastructure
      annotations:
        summary: "High CPU usage"
        description: |
          CPU usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}
          (threshold: 80%)

    # 高メモリ使用率
    - alert: HighMemoryUsage
      expr: |
        (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.9
      for: 10m
      labels:
        severity: warning
        team: infrastructure
      annotations:
        summary: "High memory usage"
        description: |
          Memory usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}
          (threshold: 90%)

    # ディスク容量不足
    - alert: DiskSpaceLow
      expr: |
        (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.*"} / node_filesystem_size_bytes)) > 0.85
      for: 10m
      labels:
        severity: warning
        team: infrastructure
      annotations:
        summary: "Disk space is low"
        description: |
          Disk usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}:{{ $labels.mountpoint }}
          (threshold: 85%)

  # グループ3: Kubernetesアラート
  - name: kubernetes_alerts
    interval: 1m
    rules:
    # Pod再起動が頻繁
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Pod is crash looping"
        description: |
          Pod {{ $labels.namespace }}/{{ $labels.pod }} is restarting frequently

    # Podが起動しない
    - alert: PodNotReady
      expr: |
        kube_pod_status_phase{phase!~"Running|Succeeded"} > 0
      for: 15m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Pod is not ready"
        description: |
          Pod {{ $labels.namespace }}/{{ $labels.pod }} is in {{ $labels.phase }} phase

    # HPA が上限に達している
    - alert: HPAMaxedOut
      expr: |
        kube_horizontalpodautoscaler_status_current_replicas
        >=
        kube_horizontalpodautoscaler_spec_max_replicas
      for: 15m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "HPA has reached maximum replicas"
        description: |
          HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} is at maximum capacity
```

### 2.4 Alertmanager の設定

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: main-config
  namespace: monitoring
spec:
  # ルート設定
  route:
    receiver: default
    groupBy: ['alertname', 'cluster', 'service']
    groupWait: 30s         # 同じグループのアラートを30秒待ってまとめる
    groupInterval: 5m      # グループ化されたアラートの再送間隔
    repeatInterval: 12h    # 同じアラートの再送間隔

    routes:
    # Critical アラートは即座にPagerDutyへ
    - match:
        severity: critical
      receiver: pagerduty
      groupWait: 10s
      repeatInterval: 1h

    # Warningアラートはslackへ
    - match:
        severity: warning
      receiver: slack-warnings

    # チームごとのルーティング
    - match:
        team: backend
      receiver: slack-backend

    - match:
        team: infrastructure
      receiver: slack-infrastructure

  # Receiver定義
  receivers:
  # デフォルト（すべてのアラート）
  - name: default
    slackConfigs:
    - apiURL:
        name: slack-webhook
        key: url
      channel: '#alerts-all'
      title: '{{ .GroupLabels.alertname }}'
      text: |
        {{ range .Alerts }}
        *Alert:* {{ .Labels.alertname }}
        *Severity:* {{ .Labels.severity }}
        *Description:* {{ .Annotations.description }}
        {{ end }}

  # PagerDuty（Critical用）
  - name: pagerduty
    pagerdutyConfigs:
    - routingKey:
        name: pagerduty-key
        key: routing-key
      description: '{{ .GroupLabels.alertname }}'
      severity: '{{ .CommonLabels.severity }}'

  # Slack - Warnings
  - name: slack-warnings
    slackConfigs:
    - apiURL:
        name: slack-webhook
        key: url
      channel: '#alerts-warnings'
      color: 'warning'

  # Slack - Backend team
  - name: slack-backend
    slackConfigs:
    - apiURL:
        name: slack-webhook
        key: url
      channel: '#team-backend-alerts'

  # Email
  - name: email
    emailConfigs:
    - to: 'ops-team@example.com'
      from: 'alertmanager@example.com'
      smarthost: 'smtp.gmail.com:587'
      authUsername: 'alertmanager@example.com'
      authPassword:
        name: smtp-password
        key: password
      headers:
        Subject: '[{{ .Status }}] {{ .GroupLabels.alertname }}'

  # 抑制ルール（Inhibition）
  inhibitRules:
  # NodeDownの時は、そのNode上のPodアラートを抑制
  - sourceMatch:
      alertname: NodeDown
    targetMatch:
      alertname: PodNotReady
    equal: ['instance']
```

---

## 3. 分散トレーシング

### 3.1 OpenTelemetry の詳細

#### OpenTelemetry アーキテクチャ

```
┌───────────────────────────────────────────────────────────┐
│ アプリケーション（Go例）                                   │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ import "go.opentelemetry.io/otel"                     │ │
│ │                                                       │ │
│ │ tracer := otel.Tracer("myapp")                        │ │
│ │ ctx, span := tracer.Start(ctx, "handleRequest")       │ │
│ │ defer span.End()                                      │ │
│ │                                                       │ │
│ │ // 自動計装: HTTP client, DB, gRPC                    │ │
│ └───────────────────────────────────────────────────────┘ │
└─────────────────────────┬─────────────────────────────────┘
                          │ OTLP (gRPC/HTTP)
                          ▼
┌───────────────────────────────────────────────────────────┐
│ OpenTelemetry Collector (DaemonSet/Sidecar)              │
│ ┌───────────────────────────────────────────────────────┐ │
│ │ Receiver → Processor → Exporter                       │ │
│ │                                                       │ │
│ │ ・バッチ処理                                          │ │
│ │ ・サンプリング                                        │ │
│ │ ・フィルタリング                                      │ │
│ │ ・属性の追加/削除                                     │ │
│ └───────────────────────────────────────────────────────┘ │
└─────────────────────────┬─────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Jaeger       │  │ Tempo        │  │ Zipkin       │
│ (UIあり)     │  │ (Grafana連携)│  │ (レガシー)    │
└──────────────┘  └──────────────┘  └──────────────┘
```

#### OpenTelemetry Collector の設定

```yaml
# OpenTelemetry Collector ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      # OTLP receiver (gRPC)
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      # Jaeger receiver (後方互換性)
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_http:
            endpoint: 0.0.0.0:14268

      # Zipkin receiver
      zipkin:
        endpoint: 0.0.0.0:9411

    processors:
      # バッチ処理（効率化）
      batch:
        timeout: 10s
        send_batch_size: 1024

      # サンプリング（本番環境で必須）
      probabilistic_sampler:
        sampling_percentage: 10  # 10%のtraceのみ保存

      # Tail sampling（エラーのtraceは100%保存）
      tail_sampling:
        policies:
        # エラーは必ず保存
        - name: error-policy
          type: status_code
          status_code:
            status_codes: [ERROR]
        # 遅いリクエストは必ず保存
        - name: slow-requests
          type: latency
          latency:
            threshold_ms: 1000
        # それ以外は10%
        - name: probabilistic-policy
          type: probabilistic
          probabilistic:
            sampling_percentage: 10

      # Kubernetes属性の追加
      k8sattributes:
        auth_type: "serviceAccount"
        passthrough: false
        extract:
          metadata:
          - k8s.namespace.name
          - k8s.deployment.name
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.node.name
          labels:
          - tag_name: app
            key: app
            from: pod

      # リソース属性の追加
      resource:
        attributes:
        - key: cluster.name
          value: production-eks
          action: upsert
        - key: environment
          value: production
          action: upsert

    exporters:
      # Jaeger exporter
      jaeger:
        endpoint: jaeger-collector.observability.svc:14250
        tls:
          insecure: true

      # Tempo exporter (Grafana)
      otlp/tempo:
        endpoint: tempo.observability.svc:4317
        tls:
          insecure: true

      # ログ出力（デバッグ用）
      logging:
        loglevel: info

    service:
      pipelines:
        traces:
          receivers: [otlp, jaeger, zipkin]
          processors: [k8sattributes, resource, tail_sampling, batch]
          exporters: [jaeger, otlp/tempo]

---
# OpenTelemetry Collector Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: observability
spec:
  replicas: 3
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:0.91.0
        args:
        - --config=/conf/config.yaml
        ports:
        - containerPort: 4317  # OTLP gRPC
        - containerPort: 4318  # OTLP HTTP
        - containerPort: 14250 # Jaeger gRPC
        - containerPort: 9411  # Zipkin
        volumeMounts:
        - name: config
          mountPath: /conf
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
```

#### アプリケーションでの計装（Go例）

```go
package main

import (
    "context"
    "log"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
    "go.opentelemetry.io/otel/trace"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func initTracer() (*sdktrace.TracerProvider, error) {
    // OTLP exporter の設定
    ctx := context.Background()

    conn, err := grpc.DialContext(
        ctx,
        "otel-collector.observability.svc:4317",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
    )
    if err != nil {
        return nil, err
    }

    exporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
    if err != nil {
        return nil, err
    }

    // Resource の定義（サービス識別情報）
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("myapp"),
            semconv.ServiceVersion("1.2.3"),
            semconv.DeploymentEnvironment("production"),
            attribute.String("k8s.namespace", "prod"),
        ),
    )
    if err != nil {
        return nil, err
    }

    // Tracer Provider の作成
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        // サンプリング設定
        sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.1)), // 10%
    )

    otel.SetTracerProvider(tp)
    return tp, nil
}

func main() {
    tp, err := initTracer()
    if err != nil {
        log.Fatal(err)
    }
    defer func() {
        if err := tp.Shutdown(context.Background()); err != nil {
            log.Printf("Error shutting down tracer provider: %v", err)
        }
    }()

    // Tracer の取得
    tracer := otel.Tracer("myapp")

    // Span の作成
    ctx := context.Background()
    ctx, span := tracer.Start(ctx, "main.handleRequest")
    defer span.End()

    // 属性の追加
    span.SetAttributes(
        attribute.String("user.id", "user123"),
        attribute.String("http.method", "POST"),
        attribute.String("http.route", "/api/v1/orders"),
    )

    // 子Spanの作成
    processOrder(ctx, tracer)

    // イベントの記録
    span.AddEvent("Order processed successfully")
}

func processOrder(ctx context.Context, tracer trace.Tracer) {
    ctx, span := tracer.Start(ctx, "processOrder")
    defer span.End()

    // データベースクエリ
    queryDB(ctx, tracer)

    // 外部API呼び出し
    callExternalAPI(ctx, tracer)
}

func queryDB(ctx context.Context, tracer trace.Tracer) {
    ctx, span := tracer.Start(ctx, "database.query")
    defer span.End()

    span.SetAttributes(
        attribute.String("db.system", "postgresql"),
        attribute.String("db.statement", "SELECT * FROM orders WHERE id = $1"),
    )

    // クエリ実行（疑似）
    time.Sleep(50 * time.Millisecond)
}

func callExternalAPI(ctx context.Context, tracer trace.Tracer) {
    ctx, span := tracer.Start(ctx, "http.client.request")
    defer span.End()

    span.SetAttributes(
        attribute.String("http.url", "https://api.example.com/payment"),
        attribute.String("http.method", "POST"),
    )

    // API呼び出し（疑似）
    time.Sleep(100 * time.Millisecond)

    span.SetAttributes(
        attribute.Int("http.status_code", 200),
    )
}
```

---

このドキュメントは可観測性の主要部分をカバーしています。続けて「災害復旧（Disaster Recovery）」のドキュメントを作成します。

## 📚 参考リソース

- [Fluent Bit Documentation](https://docs.fluentbit.io/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Grafana Loki](https://grafana.com/oss/loki/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Jaeger](https://www.jaegertracing.io/)
