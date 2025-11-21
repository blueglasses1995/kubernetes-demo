# Kubernetes 完全マスター カリキュラム

## 📚 カリキュラム概要

このカリキュラムは、Kubernetesの基礎から本番運用まで、**段階的に一つずつ徹底的に学習**する構成になっています。

## 🎯 学習の原則

**重要: 一度に一つのサービス・技術のみを学習**
- 各章を完全に理解してから次に進む
- 複数の技術を同時に学ばない
- 各章の全演習を完了してから次の章へ

---

## 🗺️ カリキュラムロードマップ

### 📘 Phase 1: 基礎編（1-2ヶ月）

#### Week 1-2: 環境構築とKubernetes基礎
**ディレクトリ**: `environments/`, `01-basic-concepts/`, `02-deployments/`, `03-services/`

**学習内容**:
- MinikubeとKindの環境構築
- Pod、Service、Deploymentの基本
- kubectl コマンドの習得
- アプリケーションデプロイメント
- ネットワーキングの基礎

**習得目標**:
✅ Kubernetesの基本操作とリソース管理ができる
✅ kubectl を使った基本的な操作ができる
✅ Podのライフサイクルを理解している

**学習時間**: 15時間（理論8時間 + 実践7時間）

---

#### Week 3-4: 高度な基本機能
**ディレクトリ**: `04-advanced-topics/`

**学習内容**:
- Rolling Update とロールバック
- Horizontal Pod Autoscaler (HPA)
- ResourceQuota とLimitRange
- NetworkPolicy
- PersistentVolume と StorageClass

**習得目標**:
✅ 本番環境で必要な基本的な運用知識を習得
✅ リソース管理とスケーリングができる
✅ ネットワークポリシーでセキュリティを確保できる

**学習時間**: 10時間（理論6時間 + 実践4時間）

---

### 📗 Phase 2: エコシステムツール編（2-3ヶ月）

#### Week 5-6: パッケージ管理（Helm）
**ディレクトリ**: `05-helm/`

**学習内容**:
- Helmの基礎概念
- Chart作成とテンプレート
- Values ファイルの活用
- Helm Hooks
- リポジトリ管理
- 本番運用パターン

**習得目標**:
✅ Helmを使った効率的なアプリケーション管理ができる
✅ 独自のChartを作成できる
✅ 環境ごとのValues管理ができる

**学習時間**: 14時間（理論8時間 + 実践6時間）

---

#### Week 7-8: 可視化とIDE（Lens）
**ディレクトリ**: `06-lens/`

**学習内容**:
- Lens IDEのセットアップ
- クラスター可視化
- リソース操作とモニタリング
- ログとメトリクスの確認
- トラブルシューティング

**習得目標**:
✅ GUIを使った効率的なクラスター運用ができる
✅ リソースの状態を視覚的に把握できる
✅ Lensを使った問題の迅速な診断ができる

**学習時間**: 6時間（理論4時間 + 実践2時間）

---

### 📕 Phase 3: サービスメッシュ編（2-3ヶ月）

#### Week 9-11: シンプルなサービスメッシュ（Linkerd）
**ディレクトリ**: `07-service-mesh-linkerd/`

**学習内容**:
- サービスメッシュの概念
- Linkerdのインストールと設定
- トラフィック管理（リトライ、タイムアウト）
- 可観測性（メトリクス、トレーシング）
- 自動mTLS
- マルチクラスター連携

**習得目標**:
✅ サービスメッシュの基本概念を理解している
✅ Linkerdを使ったマイクロサービス運用ができる
✅ トラフィック制御とセキュリティ強化ができる

**学習時間**: 18時間（理論10時間 + 実践8時間）

---

#### Week 12-15: 高機能サービスメッシュ（Istio）
**ディレクトリ**: `08-service-mesh-istio/`

**学習内容**:
- Istio Ambientモードの理解
- Gateway と VirtualService
- 高度なトラフィック管理
  - カナリアデプロイメント
  - A/Bテスト
  - ミラーリング
- セキュリティポリシー
  - AuthorizationPolicy
  - PeerAuthentication
- Observability
  - Kiali（サービスグラフ）
  - Jaeger（分散トレーシング）
  - Grafana（メトリクス）
- マルチクラスター構成

**習得目標**:
✅ Istioを使った高度なマイクロサービス運用ができる
✅ 複雑なトラフィック制御を実装できる
✅ サービスメッシュのセキュリティを管理できる

**学習時間**: 25時間（理論15時間 + 実践10時間）

---

### 📙 Phase 4: 高度なオーケストレーション編（2-3ヶ月）

#### Week 16-18: インフラストラクチャ自動化
**ディレクトリ**: `09-advanced-orchestration/`

**学習内容**:
- **Karpenter** - ノード自動スケーリング
  - Karpenterの概念とアーキテクチャ
  - Provisioner設定
  - インスタンス選択戦略
  - コスト最適化
  - Spot インスタンスの活用

- **Crossplane** - Infrastructure as Code
  - Crossplaneの基礎
  - Provider設定
  - クラウドリソース管理
  - Composition作成
  - GitOpsとの統合

**習得目標**:
✅ インフラストラクチャの自動化ができる
✅ コスト最適化された自動スケーリングを実装できる
✅ Kubernetes API経由でクラウドリソースを管理できる

**学習時間**: 20時間（理論12時間 + 実践8時間）

---

### 📔 Phase 5: プラットフォーム抽象化編（1-2ヶ月）

#### Week 19-21: 次世代プラットフォーム
**ディレクトリ**: `10-platform-abstraction/`

**学習内容**:
- **Render** - マネージドKubernetesプラットフォーム
  - Renderの概要とアーキテクチャ
  - GitOpsベースのデプロイメント
  - 自動スケーリング
  - データベース統合

- **Fly.io** - エッジコンピューティングプラットフォーム
  - Fly.ioの概要
  - Firecracker MicroVM
  - Anycast ネットワーク
  - グローバル配置戦略

- プラットフォーム比較と選択基準

**習得目標**:
✅ モダンなプラットフォームの理解と活用ができる
✅ プラットフォームの選択基準を理解している
✅ エッジコンピューティングの概念を理解している

**学習時間**: 14時間（理論8時間 + 実践6時間）

---

### 🏭 Phase 6: 本番環境運用編（3-4ヶ月）★新設★

このPhaseでは、実際の本番環境で必要となる運用スキルを徹底的に習得します。

#### Week 22-24: セキュリティ
**ドキュメント**: `PRODUCTION_SECURITY.md`

**学習内容**:
- **RBAC (Role-Based Access Control)**
  - API Serverでの認可処理フロー
  - Role/ClusterRoleの設計
  - ServiceAccount管理
  - 監査ログ（Audit Logging）

- **Pod Security**
  - Pod Security Standards (PSS)
  - SecurityContext詳細設定
  - Linux Capabilities管理
  - AppArmor/SELinux

- **Secrets管理**
  - Encryption at Rest
  - External Secrets Operator
  - HashiCorp Vault連携
  - Secretsローテーション

- **イメージセキュリティ**
  - Trivyスキャニング
  - Admission Controllerによるポリシー強制

- **Runtime Security**
  - Falcoによる侵入検知

**習得目標**:
✅ 本番環境のセキュリティ要件を満たせる
✅ RBAC設計と実装ができる
✅ Secretsを安全に管理できる
✅ ランタイムセキュリティを実装できる

**学習時間**: 16時間（理論10時間 + 実践6時間）

---

#### Week 25-27: 可観測性（Observability）
**ドキュメント**: `PRODUCTION_OBSERVABILITY.md`

**学習内容**:
- **ログ管理**
  - EFK Stack vs Loki Stack
  - Fluent Bit設定
  - 構造化ログのベストプラクティス

- **メトリクス**
  - Prometheus Operator
  - ServiceMonitor/PodMonitor
  - Recording Rules と Alerting Rules
  - Alertmanager設定

- **分散トレーシング**
  - OpenTelemetry
  - Jaeger/Tempo
  - サンプリング戦略

- **ダッシュボード**
  - Grafana設定
  - 主要メトリクスのダッシュボード作成

**習得目標**:
✅ 包括的なログ収集システムを構築できる
✅ メトリクスベースの監視とアラートを設定できる
✅ 分散トレーシングでパフォーマンス問題を特定できる

**学習時間**: 18時間（理論10時間 + 実践8時間）

---

#### Week 28-29: 災害復旧（DR）
**ドキュメント**: `PRODUCTION_DISASTER_RECOVERY.md`

**学習内容**:
- **etcdバックアップ**
  - 手動バックアップ
  - 自動バックアップ（CronJob）
  - リストア手順

- **Veleroによるアプリケーションバックアップ**
  - Veleroのインストール
  - バックアップとリストア
  - スケジュールバックアップ
  - Volume Snapshots

- **RTO/RPO設計**
  - サービスレベル別の設計
  - バックアップ保持ポリシー

- **Multi-cluster DR**
  - Active-Passive構成
  - Active-Active構成

- **DR訓練**
  - 訓練計画と実施

**習得目標**:
✅ 包括的なバックアップ戦略を設計できる
✅ DR手順を自動化できる
✅ RTO/RPOを満たすシステムを構築できる

**学習時間**: 12時間（理論7時間 + 実践5時間）

---

#### Week 30-31: クラスター運用
**ドキュメント**: `PRODUCTION_CLUSTER_OPERATIONS.md`

**学習内容**:
- **クラスターアップグレード**
  - バージョンスキューポリシー
  - kubeadmアップグレード手順
  - マネージドKubernetes（EKS/GKE/AKS）アップグレード

- **ノードメンテナンス**
  - Drain と Cordon
  - OSパッチング
  - Node Problem Detector

- **証明書管理**
  - 証明書の構造と有効期限
  - 証明書の更新
  - cert-manager自動更新

- **容量計画**
  - クラスターサイジング
  - 成長予測
  - Bin packing効率

**習得目標**:
✅ 無停止でクラスターをアップグレードできる
✅ ノードメンテナンスを安全に実施できる
✅ 適切な容量計画ができる

**学習時間**: 14時間（理論8時間 + 実践6時間）

---

#### Week 32-33: トラブルシューティング
**ドキュメント**: `PRODUCTION_TROUBLESHOOTING.md`

**学習内容**:
- **デバッグツール**
  - kubectl debug
  - Ephemeral Containers
  - k9s（TUI）
  - パケットキャプチャ

- **一般的な問題**
  - ImagePullBackOff
  - CrashLoopBackOff
  - Pending Pods
  - PVC Binding失敗

- **パフォーマンス問題**
  - 高CPU使用率
  - メモリリーク/OOMKilled
  - ディスクI/Oボトルネック

- **ネットワーク問題**
  - DNS解決失敗
  - Service接続問題
  - NetworkPolicy問題
  - Ingress問題

**習得目標**:
✅ 一般的な問題を迅速に診断・解決できる
✅ パフォーマンス問題を特定・解決できる
✅ ネットワーク問題をデバッグできる

**学習時間**: 15時間（理論8時間 + 実践7時間）

---

#### Week 34-36: CI/CD & GitOps
**ドキュメント**: `PRODUCTION_CICD_GITOPS.md`

**学習内容**:
- **GitOps原理**
  - GitOpsの4原則
  - リポジトリ構成戦略
  - App of Apps パターン

- **ArgoCD**
  - アーキテクチャ理解
  - Application CRD
  - Sync Waves と Hooks
  - AppProject（RBAC）
  - Notifications

- **Flux CD**
  - アーキテクチャ理解
  - GitRepository と Kustomization
  - Image Update Automation

- **Progressive Delivery**
  - Canary Deployment（Flagger）
  - Blue-Green Deployment

- **Image Buildパイプライン**
  - GitHub Actions
  - マルチステージビルド
  - Trivy統合

**習得目標**:
✅ GitOpsを使った宣言的なデプロイメントができる
✅ ArgoCD/Fluxを本番運用できる
✅ Progressive Deliveryを実装できる
✅ セキュアなイメージビルドパイプラインを構築できる

**学習時間**: 20時間（理論12時間 + 実践8時間）

---

#### Week 37-39: ネットワーク詳細
**ドキュメント**: `PRODUCTION_NETWORKING.md`

**学習内容**:
- **CNI (Container Network Interface)**
  - CNIアーキテクチャ
  - Calico vs Cilium比較
  - カプセル化方式（VXLAN, BGP）
  - eBPF データプレーン

- **Service と Load Balancing**
  - kube-proxyモード（iptables, IPVS, eBPF）
  - Service Types詳細
  - ExternalTrafficPolicy

- **Ingress Controllers**
  - Nginx Ingress Controller
  - cert-manager統合
  - mTLS設定

- **NetworkPolicy**
  - Default Deny Policy
  - アプリケーション間通信制御
  - Egress制御

- **DNS と CoreDNS**
  - Kubernetes DNS構造
  - CoreDNS設定カスタマイズ
  - Pod DNS設定

**習得目標**:
✅ CNIの選択と設定ができる
✅ 複雑なネットワークポリシーを設計・実装できる
✅ Ingressを使った高度なルーティングができる
✅ DNS問題をトラブルシュートできる

**学習時間**: 18時間（理論10時間 + 実践8時間）

---

#### Week 40-42: ストレージ
**ドキュメント**: `PRODUCTION_STORAGE.md`

**学習内容**:
- **CSI (Container Storage Interface)**
  - CSIアーキテクチャ
  - AWS EBS CSI Driver
  - StorageClass設計
  - Volume拡張

- **StatefulSetとステートフルアプリケーション**
  - StatefulSet動作原理
  - Headless Service
  - 更新戦略
  - データベースクラスター実装

- **Volume Types**
  - emptyDir
  - ConfigMap/Secret
  - Projected Volumes

- **ストレージパフォーマンス**
  - ストレージクラス選択
  - I/Oパフォーマンステスト

- **バックアップとスナップショット**
  - VolumeSnapshot
  - スケジュールスナップショット

**習得目標**:
✅ CSIを使ったストレージ管理ができる
✅ StatefulSetで状態を持つアプリを運用できる
✅ ストレージパフォーマンスを最適化できる
✅ Volume スナップショット戦略を実装できる

**学習時間**: 16時間（理論9時間 + 実践7時間）

---

#### Week 43-45: パフォーマンスチューニング ★新設★
**ドキュメント**: `PRODUCTION_PERFORMANCE_TUNING.md`

**学習内容**:
- **リソース最適化**
  - Requests と Limits の適切な設定
  - VPA (Vertical Pod Autoscaler)
  - Resource Quotas と LimitRanges
  - QoS クラス

- **スケジューリング最適化**
  - Node Affinity と Pod Affinity
  - Taints と Tolerations
  - Priority と Preemption
  - Topology Spread Constraints

- **アプリケーション最適化**
  - コンテナイメージ最適化
  - Startup/Liveness/Readiness Probes調整
  - PodDisruptionBudget

- **ネットワークパフォーマンス**
  - CNI パフォーマンス比較
  - Service Mesh オーバーヘッド削減

- **ストレージパフォーマンス**
  - I/O最適化
  - Local PersistentVolume活用

**習得目標**:
✅ クラスターリソースを最適化できる
✅ スケジューリングを最適化できる
✅ アプリケーションパフォーマンスを向上できる
✅ ボトルネックを特定・解消できる

**学習時間**: 18時間（理論10時間 + 実践8時間）

---

#### Week 46-48: コスト最適化 ★新設★
**ドキュメント**: `PRODUCTION_COST_OPTIMIZATION.md`

**学習内容**:
- **コンピュートコスト最適化**
  - Spotインスタンス活用
  - Karpenter コスト最適化設定
  - Right-sizing（適正サイジング）
  - Cluster Autoscaler最適化

- **ストレージコスト最適化**
  - ストレージクラス選択戦略
  - 不要なPVCの削除
  - Volume Snapshot管理

- **ネットワークコスト最適化**
  - データ転送コスト削減
  - クロスAZトラフィック削減

- **可視性とレポーティング**
  - Kubecost導入
  - コスト配分（Showback/Chargeback）
  - リソース使用率レポート

- **FinOps ベストプラクティス**
  - コストアラート設定
  - 予算管理
  - 継続的な最適化

**習得目標**:
✅ クラスターコストを可視化できる
✅ コンピュート・ストレージ・ネットワークコストを最適化できる
✅ コスト配分を実装できる
✅ FinOpsプラクティスを適用できる

**学習時間**: 16時間（理論9時間 + 実践7時間）

---

### 📓 Phase 7: 実践プロジェクト編（2-3ヶ月）

#### Week 49-54: 総合演習
**ディレクトリ**: `11-production-project/`

**学習内容**:
- マイクロサービスアプリケーションの構築
  - フロントエンド、バックエンド、データベース
  - Service Mesh統合
- CI/CDパイプライン構築
  - GitOps (ArgoCD/Flux)
  - Progressive Delivery
- 包括的な監視
  - ログ、メトリクス、トレーシング
- セキュリティ対策実装
  - RBAC、NetworkPolicy、Pod Security
- 負荷テストと性能最適化
- 障害対応演習
  - カオスエンジニアリング
  - DR訓練
- コスト最適化実践

**習得目標**:
✅ すべての知識を統合した本番運用能力
✅ エンドツーエンドのシステム構築ができる
✅ 本番環境での運用・保守ができる
✅ 障害対応とパフォーマンス改善ができる

**学習時間**: 30時間（理論10時間 + 実践20時間）

---

## 📊 学習進捗チェックリスト

### Phase 1: 基礎編
- [ ] 環境構築（Minikube & Kind）完了
- [ ] Kubernetes基本リソース理解
- [ ] kubectl コマンド習得
- [ ] 高度な基本機能習得

### Phase 2: エコシステムツール編
- [ ] Helm完全習得
- [ ] Lens操作習得

### Phase 3: サービスメッシュ編
- [ ] Linkerd完全習得
- [ ] Istio完全習得

### Phase 4: 高度なオーケストレーション編
- [ ] Karpenter習得
- [ ] Crossplane習得

### Phase 5: プラットフォーム抽象化編
- [ ] Render習得
- [ ] Fly.io習得

### Phase 6: 本番環境運用編 ★拡充★
- [ ] セキュリティ実装習得
- [ ] 可観測性システム構築習得
- [ ] 災害復旧戦略習得
- [ ] クラスター運用習得
- [ ] トラブルシューティング習得
- [ ] CI/CD & GitOps習得
- [ ] ネットワーク詳細習得
- [ ] ストレージ管理習得
- [ ] パフォーマンスチューニング習得 ★新設★
- [ ] コスト最適化習得 ★新設★

### Phase 7: 実践プロジェクト編
- [ ] 総合プロジェクト完了

---

## 🎓 学習時間の目安（更新版）

| Phase | 内容 | 学習時間 | 実践時間 | 合計 |
|-------|------|----------|---------|------|
| **Phase 1** | 基礎編 | 14時間 | 11時間 | 25時間 |
| **Phase 2** | エコシステムツール | 12時間 | 8時間 | 20時間 |
| **Phase 3** | サービスメッシュ | 25時間 | 18時間 | 43時間 |
| **Phase 4** | 高度なオーケストレーション | 12時間 | 8時間 | 20時間 |
| **Phase 5** | プラットフォーム抽象化 | 8時間 | 6時間 | 14時間 |
| **Phase 6** | **本番環境運用** | **93時間** | **72時間** | **165時間** |
| ├─ | セキュリティ | 10時間 | 6時間 | 16時間 |
| ├─ | 可観測性 | 10時間 | 8時間 | 18時間 |
| ├─ | 災害復旧 | 7時間 | 5時間 | 12時間 |
| ├─ | クラスター運用 | 8時間 | 6時間 | 14時間 |
| ├─ | トラブルシューティング | 8時間 | 7時間 | 15時間 |
| ├─ | CI/CD & GitOps | 12時間 | 8時間 | 20時間 |
| ├─ | ネットワーク詳細 | 10時間 | 8時間 | 18時間 |
| ├─ | ストレージ | 9時間 | 7時間 | 16時間 |
| ├─ | パフォーマンスチューニング ★ | 10時間 | 8時間 | 18時間 |
| └─ | コスト最適化 ★ | 9時間 | 7時間 | 16時間 |
| **Phase 7** | 実践プロジェクト | 10時間 | 20時間 | 30時間 |
| **合計** | | **174時間** | **143時間** | **317時間** |

> ⚠️ **重要**: 個人の学習ペースにより時間は変動します。理解を優先し、焦らず進めてください。

---

## 🚀 学習の進め方

### 1. 事前準備
- DockerとDocker Desktopのインストール
- 十分なディスクスペース（最低100GB推奨）
- 学習用のGitHubアカウント
- AWSアカウント（本番環境運用編で使用）

### 2. 各章の学習フロー
```
1. READMEまたはドキュメントを読んで概要を理解
   ↓
2. セットアップ手順に従って環境構築
   ↓
3. 演習1から順番に実施（手を動かす）
   ↓
4. 各演習を完全に理解（なぜそうなるのか）
   ↓
5. トラブルシューティングセクションを確認
   ↓
6. 理解度チェック（各章末）
   ↓
7. 次の章へ進む
```

### 3. 重要な注意点
- **絶対に複数の章を同時に進めない**
- 各演習で手を動かして実際に試す
- エラーが出たら必ずトラブルシューティング
- 理解できない部分は何度も繰り返す
- 環境は定期的にクリーンアップ
- 本番環境運用編では実際のクラウド環境を推奨

---

## 💡 推奨学習環境

### ハードウェア
- **CPU**: 4コア以上（8コア推奨）
- **RAM**: 16GB以上（**32GB強く推奨**）
- **Disk**: SSD 100GB以上の空き容量（200GB推奨）

### ソフトウェア
- **OS**: macOS、Linux、またはWSL2（Windows）
- **Container Runtime**: Docker Desktop
- **ターミナル**: iTerm2、Windows Terminal、またはAlacritty
- **エディタ**: VSCode（Kubernetes拡張機能推奨）
- **クラウド**: AWS/GCP/Azure アカウント（Phase 6以降）

### 推奨ツール
```bash
# 必須
- kubectl
- helm
- k9s

# 本番環境運用編で使用
- argocd CLI
- flux CLI
- velero CLI
- kubectx/kubens
- stern (ログストリーミング)
- trivy
```

---

## 📖 参考書籍・リソース

### 公式ドキュメント
- [Kubernetes公式ドキュメント](https://kubernetes.io/docs/)
- [Helm公式ドキュメント](https://helm.sh/docs/)
- [Linkerd公式ドキュメント](https://linkerd.io/docs/)
- [Istio公式ドキュメント](https://istio.io/docs/)
- [ArgoCD公式ドキュメント](https://argo-cd.readthedocs.io/)
- [Flux公式ドキュメント](https://fluxcd.io/docs/)

### 本番運用リソース
- [Kubernetes Production Best Practices](https://learnk8s.io/production-best-practices)
- [CNCF Cloud Native Landscape](https://landscape.cncf.io/)
- [Kubernetes Failure Stories](https://k8s.af/)

### コミュニティ
- [Kubernetes Slack](https://slack.k8s.io/)
- [CNCF](https://www.cncf.io/)
- [Kubernetes Tokyo](https://k8sjp.connpass.com/)

---

## 🎯 カリキュラム完了後のスキルレベル

このカリキュラムを完了すると：

### 基礎スキル
✅ Kubernetesの基礎から高度な機能まで完全理解
✅ Helmを使った効率的なアプリケーション管理
✅ サービスメッシュを使ったマイクロサービス運用
✅ インフラストラクチャの自動化
✅ モダンなプラットフォームの活用

### 本番環境運用スキル ★拡充★
✅ エンタープライズグレードのセキュリティ実装
✅ 包括的な可観測性システム構築
✅ 災害復旧戦略の設計と実装
✅ クラスターの安全な運用と保守
✅ 複雑な問題の診断と解決
✅ GitOpsを使った継続的デリバリー
✅ 高度なネットワーク設計
✅ ステートフルアプリケーションの運用
✅ **パフォーマンス最適化** ★新設★
✅ **コスト最適化と FinOps** ★新設★

### 到達レベル
**目標**: Kubernetes SRE / Platform Engineer
**認定資格レベル**:
- CKA (Certified Kubernetes Administrator) 合格レベル
- CKAD (Certified Kubernetes Application Developer) 合格レベル
- CKS (Certified Kubernetes Security Specialist) 準備レベル

---

## 📝 学習記録テンプレート

各Phaseで以下を記録することを推奨：

```markdown
### Phase X 学習記録

**期間**: YYYY/MM/DD - YYYY/MM/DD

**学習内容**:
-

**つまづいた点**:
-

**解決方法**:
-

**理解度**: ⭐⭐⭐⭐⭐ (5段階)

**次のPhaseに進む前のチェック**:
- [ ] すべての演習を完了
- [ ] 主要な概念を説明できる
- [ ] トラブルシューティングができる
```

---

## 🎓 修了証明

カリキュラム完了後、以下を実施することを推奨：

1. **総合プロジェクトのGitHubリポジトリ作成**
   - Infrastructure as Code (Terraform/Crossplane)
   - GitOps manifests
   - CI/CD pipelines
   - 運用ドキュメント

2. **ブログ記事執筆**
   - 学習の振り返り
   - つまづいた点と解決方法
   - 本番運用での知見

3. **認定資格取得**
   - CKA取得
   - CKAD取得
   - CKS挑戦

---

**次のステップ**:

Phase 1から開始する場合 → [environments/README.md](./environments/README.md)
Phase 6（本番運用）から開始する場合 → [PRODUCTION_TOPICS_LIST.md](./PRODUCTION_TOPICS_LIST.md)

**頑張ってください！🚀**
