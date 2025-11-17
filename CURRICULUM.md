# Kubernetes 完全マスター カリキュラム

## 📚 カリキュラム概要

このカリキュラムは、Kubernetesの基礎から最先端の技術まで、**段階的に一つずつ徹底的に学習**する構成になっています。

## 🎯 学習の原則

**重要: 一度に一つのサービス・技術のみを学習**
- 各章を完全に理解してから次に進む
- 複数の技術を同時に学ばない
- 各章の全演習を完了してから次の章へ

## 🗺️ カリキュラムロードマップ

### 📘 Phase 1: 基礎編（1-2ヶ月）

#### Week 1-2: 環境構築とKubernetes基礎
- **environments/** - MinikubeとKindの環境構築
- **01-basic-concepts/** - Pod、Service、Deployment
- **02-deployments/** - アプリケーションデプロイメント
- **03-services/** - ネットワーキングの基礎

✅ **習得目標**: Kubernetesの基本操作とリソース管理

#### Week 3-4: 高度な基本機能
- **04-advanced-topics/** - Rolling Update、HPA、ResourceQuota、NetworkPolicy、PersistentVolume

✅ **習得目標**: 本番環境で必要な基本的な運用知識

---

### 📗 Phase 2: エコシステムツール編（2-3ヶ月）

#### Week 5-6: パッケージ管理
- **05-helm/** - Helmによるアプリケーション管理
  - Helmの基礎
  - Chart作成
  - リポジトリ管理
  - 本番運用パターン

✅ **習得目標**: Helmを使った効率的なアプリケーション管理

#### Week 7-8: 可視化とIDE
- **06-lens/** - Lens IDEによるクラスター管理
  - Lensのセットアップ
  - クラスター可視化
  - リソース操作
  - トラブルシューティング

✅ **習得目標**: GUIを使った効率的なクラスター運用

---

### 📕 Phase 3: サービスメッシュ編（2-3ヶ月）

#### Week 9-11: シンプルなサービスメッシュ
- **07-service-mesh-linkerd/** - Linkerdの完全マスター
  - サービスメッシュの概念
  - Linkerdのインストール
  - トラフィック管理
  - 可観測性（メトリクス、トレーシング）
  - セキュリティ（mTLS）

✅ **習得目標**: サービスメッシュの基本概念とLinkerd運用

#### Week 12-15: 高機能サービスメッシュ
- **08-service-mesh-istio/** - Istioの完全マスター
  - Istio Ambientモードの理解
  - トラフィック管理（Advanced）
  - セキュリティポリシー
  - Observability（Kiali、Jaeger、Grafana）
  - マルチクラスター構成

✅ **習得目標**: Istioを使った高度なマイクロサービス運用

---

### 📙 Phase 4: 高度なオーケストレーション編（2-3ヶ月）

#### Week 16-18: インフラストラクチャ自動化
- **09-advanced-orchestration/**
  - **Karpenter** - ノード自動スケーリング
    - Karpenterの概念
    - プロビジョナー設定
    - コスト最適化
  - **Crossplane** - Infrastructure as Code
    - Crossplaneの基礎
    - クラウドリソース管理
    - Composition作成

✅ **習得目標**: インフラストラクチャの自動化と最適化

---

### 📔 Phase 5: プラットフォーム抽象化編（1-2ヶ月）

#### Week 19-21: 次世代プラットフォーム
- **10-platform-abstraction/**
  - **Render** - マネージドKubernetesプラットフォーム
  - **Fly.io** - エッジコンピューティングプラットフォーム
  - プラットフォーム比較と選択基準

✅ **習得目標**: モダンなプラットフォームの理解と活用

---

### 📓 Phase 6: 実践プロジェクト編（1-2ヶ月）

#### Week 22-24: 総合演習
- **11-production-project/** - 本番環境を想定した総合プロジェクト
  - マイクロサービスアプリケーションの構築
  - CI/CDパイプライン
  - モニタリングとロギング
  - セキュリティ対策
  - 障害対応演習

✅ **習得目標**: すべての知識を統合した本番運用能力

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

### Phase 6: 実践プロジェクト編
- [ ] 総合プロジェクト完了

## 🎓 各章の学習時間の目安

| 章 | 学習時間 | 実践時間 | 合計 |
|----|---------|---------|------|
| 環境構築 | 2時間 | 1時間 | 3時間 |
| 01-03 基礎 | 8時間 | 4時間 | 12時間 |
| 04 高度な基本 | 6時間 | 4時間 | 10時間 |
| 05 Helm | 8時間 | 6時間 | 14時間 |
| 06 Lens | 4時間 | 2時間 | 6時間 |
| 07 Linkerd | 10時間 | 8時間 | 18時間 |
| 08 Istio | 15時間 | 10時間 | 25時間 |
| 09 Orchestration | 12時間 | 8時間 | 20時間 |
| 10 Platform | 8時間 | 6時間 | 14時間 |
| 11 Project | 10時間 | 15時間 | 25時間 |
| **合計** | **83時間** | **64時間** | **147時間** |

## 🚀 学習の進め方

### 1. 事前準備
- DockerとDocker Desktopのインストール
- 十分なディスクスペース（最低50GB推奨）
- 学習用のGitHubアカウント

### 2. 各章の学習フロー
```
1. READMEを読んで概要を理解
   ↓
2. セットアップ手順に従って環境構築
   ↓
3. 演習1から順番に実施
   ↓
4. 各演習を完全に理解
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

## 💡 推奨学習環境

### ハードウェア
- CPU: 4コア以上
- RAM: 16GB以上（推奨32GB）
- Disk: SSD 100GB以上の空き容量

### ソフトウェア
- macOS、Linux、またはWSL2（Windows）
- Docker Desktop
- ターミナル（iTerm2、Windows Terminal等）
- エディタ（VSCode推奨）

## 📖 参考書籍・リソース

### 公式ドキュメント
- [Kubernetes公式ドキュメント](https://kubernetes.io/docs/)
- [Helm公式ドキュメント](https://helm.sh/docs/)
- [Linkerd公式ドキュメント](https://linkerd.io/docs/)
- [Istio公式ドキュメント](https://istio.io/docs/)

### コミュニティ
- [Kubernetes Slack](https://slack.k8s.io/)
- [CNCF](https://www.cncf.io/)

## 🎯 カリキュラム完了後のスキルレベル

このカリキュラムを完了すると：

✅ Kubernetesの基礎から高度な機能まで完全理解
✅ Helmを使った効率的なアプリケーション管理
✅ サービスメッシュを使ったマイクロサービス運用
✅ インフラストラクチャの自動化
✅ モダンなプラットフォームの活用
✅ 本番環境での運用能力

**目標レベル**: Kubernetes運用エキスパート / CKA（Certified Kubernetes Administrator）レベル以上

---

次のステップ: [environments/README.md](./environments/README.md) から環境構築を開始してください。
