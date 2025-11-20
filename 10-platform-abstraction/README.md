# 10 - Platform Abstraction: Render & Fly.io

## 📚 この章で学ぶこと

この章では、Kubernetesを抽象化したモダンなプラットフォームを**徹底的に**学習します。

**重要**: Advanced Orchestrationの章（09）を完全に理解してからこの章に進んでください。

## 🎯 学習目標

### Render
- [ ] Renderの概念を理解する
- [ ] アプリケーションをデプロイできる
- [ ] 環境変数とシークレットを管理できる
- [ ] データベースを統合できる

### Fly.io
- [ ] Fly.ioの概念を理解する
- [ ] グローバル分散アプリケーションをデプロイできる
- [ ] エッジコンピューティングを活用できる
- [ ] Fly Postgresを使える

## 📖 目次

### Part 1: Render
1. [Renderとは](#renderとは)
2. [演習1: Renderのセットアップ](#演習1-renderのセットアップ)
3. [演習2: Webサービスのデプロイ](#演習2-webサービスのデプロイ)
4. [演習3: データベース統合](#演習3-データベース統合)
5. [演習4: Infrastructure as Code](#演習4-infrastructure-as-code)

### Part 2: Fly.io
6. [Fly.ioとは](#flyioとは)
7. [演習5: Fly.ioのセットアップ](#演習5-flyioのセットアップ)
8. [演習6: アプリケーションデプロイ](#演習6-アプリケーションデプロイ)
9. [演習7: グローバル分散](#演習7-グローバル分散)
10. [演習8: Fly Postgres](#演習8-fly-postgres)

### Part 3: 比較と選択
11. [プラットフォーム比較](#プラットフォーム比較)
12. [理解度チェック](#理解度チェック)

---

# Part 1: Render

## Renderとは

### Kubernetesの抽象化

Renderは、**Kubernetesを完全に抽象化**したPaaS（Platform as a Service）です。

**従来の課題**:
- Kubernetesの複雑さ
- YAMLファイルの管理
- インフラストラクチャの運用

**Renderの解決策**:
- GitHubと連携した自動デプロイ
- 設定はGUI or YAML
- マネージドデータベース
- 自動SSL、CDN、DDoS保護

### Renderの特徴

1. **シンプル**: Gitプッシュで自動デプロイ
2. **フルスタック**: Web、Worker、Cron、Postgres、Redis
3. **料金透明**: 明確な料金体系
4. **プライベートサービス**: VPC内での通信

### ユースケース

- スタートアップのMVP
- 中小規模のWebアプリケーション
- マイクロサービス（規模による）

---

## 演習1: Renderのセットアップ

**目標**: Renderアカウントを作成し、初期設定を行う

### ステップ1-1: アカウント作成

```
1. https://render.com/ にアクセス
2. "Get Started" をクリック
3. GitHubアカウントで連携
4. 無料プランで開始
```

### ステップ1-2: GitHubリポジトリの準備

サンプルアプリケーションをフォーク：

```bash
# Node.js Express アプリ
git clone https://github.com/render-examples/express-hello-world.git
cd express-hello-world

# 自分のGitHubにプッシュ
git remote set-url origin https://github.com/YOUR_USERNAME/express-hello-world.git
git push
```

### ステップ1-3: Renderダッシュボード

Renderダッシュボードの確認：
- **Services**: Webサービス、Worker
- **Databases**: Postgres、Redis
- **Static Sites**: 静的サイト
- **Cron Jobs**: スケジュールジョブ

**✅ チェックポイント**:
- Renderアカウントを作成できる
- GitHubリポジトリを準備できる

---

## 演習2: Webサービスのデプロイ

**目標**: GitHubからアプリケーションを自動デプロイする

### ステップ2-1: 新しいWebサービスの作成

```
1. ダッシュボードで "New +" > "Web Service"
2. GitHubリポジトリを選択: express-hello-world
3. 設定：
   - Name: my-express-app
   - Environment: Node
   - Build Command: npm install
   - Start Command: npm start
   - Plan: Free
4. "Create Web Service" をクリック
```

### ステップ2-2: デプロイの監視

```
# Renderが自動的に：
1. コードをクローン
2. ビルドを実行
3. コンテナを起動
4. HTTPSエンドポイントを公開

# デプロイログを確認
# 数分でデプロイ完了
```

### ステップ2-3: アプリケーションへのアクセス

```
# 自動生成されたURL
https://my-express-app.onrender.com

# 自動SSL証明書も付与されている
```

### ステップ2-4: 継続的デプロイ

```bash
# ローカルで変更
echo "console.log('Updated!');" >> index.js

# Gitにプッシュ
git add .
git commit -m "Update app"
git push

# Renderが自動的にデプロイ！
```

**✅ チェックポイント**:
- WebサービスをGitHubから作成できる
- 自動デプロイが動作する
- HTTPSでアクセスできる

---

## 演習3: データベース統合

**目標**: Postgres データベースを追加して接続する

### ステップ3-1: Postgresデータベースの作成

```
1. ダッシュボードで "New +" > "PostgreSQL"
2. 設定：
   - Name: my-postgres-db
   - Database: myapp
   - User: myapp_user
   - Region: 同じリージョン
   - Plan: Free
3. "Create Database" をクリック
```

### ステップ3-2: 接続情報の取得

```
# Renderが自動生成：
- Internal Database URL (VPC内)
- External Database URL (外部)

# 例：
postgres://user:pass@host:5432/database
```

### ステップ3-3: 環境変数の設定

```
1. Webサービスの設定 > "Environment"
2. 環境変数を追加：
   DATABASE_URL = [Internal Database URL]
3. "Save Changes"

# アプリケーションが自動再デプロイ
```

### ステップ3-4: アプリケーションからの接続

`index.js`を更新：

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: false
  }
});

app.get('/db', async (req, res) => {
  const result = await pool.query('SELECT NOW()');
  res.json(result.rows[0]);
});
```

**✅ チェックポイント**:
- Postgresデータベースを作成できる
- 環境変数で接続情報を管理できる
- アプリケーションからDBに接続できる

---

## 演習4: Infrastructure as Code

**目標**: render.yamlでインフラを定義する

### ステップ4-1: render.yamlの作成

リポジトリルートに `render.yaml` を作成：

```yaml
services:
  - type: web
    name: my-express-app
    env: node
    buildCommand: npm install
    startCommand: npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        fromDatabase:
          name: my-postgres-db
          property: connectionString

databases:
  - name: my-postgres-db
    databaseName: myapp
    user: myapp_user
    plan: free

  - name: my-redis
    plan: free
```

### ステップ4-2: Blueprint によるデプロイ

```
1. ダッシュボードで "Blueprints" > "New Blueprint Instance"
2. リポジトリを選択
3. render.yaml が自動検出される
4. すべてのリソースが一括作成！
```

### ステップ4-3: 環境の複製

```yaml
# 本番環境とステージング環境を分離
# render.yaml をブランチごとに管理

# production ブランチ
services:
  - type: web
    name: myapp-prod
    branch: production

# staging ブランチ
services:
  - type: web
    name: myapp-staging
    branch: staging
```

**✅ チェックポイント**:
- render.yamlでインフラを定義できる
- Blueprintで一括デプロイできる
- Infrastructure as Codeを実践できる

---

# Part 2: Fly.io

## Fly.ioとは

### エッジコンピューティングプラットフォーム

Fly.ioは、**グローバルに分散したアプリケーション**を簡単に構築できるプラットフォームです。

**特徴**:
1. **Anycast**: 世界中のデータセンターにデプロイ
2. **低レイテンシ**: ユーザーに最も近いリージョンで実行
3. **Dockerネイティブ**: 任意のDockerイメージを実行可能
4. **WireGuard VPN**: プライベートネットワーク

### Renderとの違い

| 特徴 | Render | Fly.io |
|------|--------|--------|
| デプロイ方法 | Git連携 | flyctl CLI |
| グローバル分散 | ❌ | ✅ |
| エッジコンピューティング | ❌ | ✅ |
| Dockerサポート | 限定的 | 完全 |
| 学習曲線 | 低い | 中程度 |

---

## 演習5: Fly.ioのセットアップ

**目標**: Fly.io CLIをインストールしてログインする

### ステップ5-1: flyctl のインストール

```bash
# macOS / Linux
curl -L https://fly.io/install.sh | sh

# Windowsでは
powershell -Command "iwr https://fly.io/install.ps1 -useb | iex"

# インストール確認
flyctl version
```

### ステップ5-2: アカウント作成とログイン

```bash
# サインアップ（ブラウザが開く）
flyctl auth signup

# または既存アカウントでログイン
flyctl auth login
```

### ステップ5-3: リージョンの確認

```bash
# 利用可能なリージョン一覧
flyctl platform regions

# 例：
# nrt (Tokyo, Japan)
# sjc (San Jose, California)
# ams (Amsterdam, Netherlands)
# syd (Sydney, Australia)
```

**✅ チェックポイント**:
- flyctl をインストールできる
- Fly.ioにログインできる

---

## 演習6: アプリケーションデプロイ

**目標**: Node.jsアプリケーションをFly.ioにデプロイする

### ステップ6-1: サンプルアプリの準備

```bash
# Node.js Expressアプリを作成
mkdir flyio-demo && cd flyio-demo
npm init -y
npm install express

# index.js
cat > index.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Fly.io!',
    region: process.env.FLY_REGION || 'local'
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
EOF
```

### ステップ6-2: Fly.ioアプリの初期化

```bash
# fly.toml を生成
flyctl launch

# 対話的に設定：
# App Name: flyio-demo-yourname
# Region: nrt (Tokyo)
# PostgreSQL: No（後で追加）
# Deploy now: Yes
```

生成された `fly.toml`:
```toml
app = "flyio-demo"
primary_region = "nrt"

[build]
  builder = "heroku/buildpacks:20"

[env]
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true

[[services]]
  protocol = "tcp"
  internal_port = 8080

  [[services.ports]]
    port = 80
    handlers = ["http"]

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]
```

### ステップ6-3: デプロイ

```bash
# デプロイ
flyctl deploy

# ステータス確認
flyctl status

# ログ確認
flyctl logs
```

### ステップ6-4: アプリケーションへのアクセス

```bash
# ブラウザで開く
flyctl open

# URL: https://flyio-demo.fly.dev
# レスポンス：
# {
#   "message": "Hello from Fly.io!",
#   "region": "nrt"
# }
```

**✅ チェックポイント**:
- Fly.ioアプリを初期化できる
- アプリケーションをデプロイできる
- HTTPSでアクセスできる

---

## 演習7: グローバル分散

**目標**: 複数リージョンにアプリケーションをデプロイする

### ステップ7-1: リージョンの追加

```bash
# 現在のリージョンを確認
flyctl regions list

# 新しいリージョンを追加（例：シンガポール、シドニー）
flyctl regions add sin syd

# 確認
flyctl regions list
# nrt (Tokyo) - PRIMARY
# sin (Singapore)
# syd (Sydney)
```

### ステップ7-2: スケールアウト

```bash
# 各リージョンに1つずつインスタンスをデプロイ
flyctl scale count 3

# ステータス確認（各リージョンにインスタンスが配置される）
flyctl status
```

### ステップ7-3: Anycastの動作確認

```bash
# 異なるロケーションからアクセス
# ユーザーに最も近いリージョンが自動選択される

curl https://flyio-demo.fly.dev
# { "region": "nrt" }  # 日本からアクセス

# VPN経由でオーストラリアからアクセス
# { "region": "syd" }  # シドニーが選択される
```

### ステップ7-4: リージョン間の通信

```javascript
// 他のリージョンのインスタンスと通信
const got = require('got');

app.get('/regions', async (req, res) => {
  const regions = ['nrt', 'sin', 'syd'];
  const results = await Promise.all(
    regions.map(async (region) => {
      const url = `http://${region}.${process.env.FLY_APP_NAME}.internal:8080/`;
      const response = await got(url).json();
      return { region, data: response };
    })
  );
  res.json(results);
});
```

**✅ チェックポイント**:
- 複数リージョンにスケールアウトできる
- Anycastの動作を理解している
- リージョン間通信ができる

---

## 演習8: Fly Postgres

**目標**: Fly PostgresをセットアップしてアプリケーションからDB接続する

### ステップ8-1: Postgresクラスターの作成

```bash
# Postgresクラスターを作成
flyctl postgres create --name myapp-db --region nrt

# 設定：
# Initial cluster size: 2（HA構成）
# VM size: shared-cpu-1x
# Volume size: 10GB
```

### ステップ8-2: アプリケーションをDBに接続

```bash
# アプリケーションにDBをアタッチ
flyctl postgres attach myapp-db

# 環境変数 DATABASE_URL が自動設定される
```

### ステップ8-3: マイグレーションの実行

```bash
# Postgresクラスターに接続
flyctl postgres connect -a myapp-db

# テーブル作成
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW()
);
```

### ステップ8-4: アプリケーションからのDB操作

`index.js`を更新：

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

app.post('/users', async (req, res) => {
  const { name } = req.body;
  const result = await pool.query(
    'INSERT INTO users (name) VALUES ($1) RETURNING *',
    [name]
  );
  res.json(result.rows[0]);
});

app.get('/users', async (req, res) => {
  const result = await pool.query('SELECT * FROM users');
  res.json(result.rows);
});
```

再デプロイ：
```bash
flyctl deploy
```

**✅ チェックポイント**:
- Fly Postgresを作成できる
- アプリケーションにDBを接続できる
- データベース操作ができる

---

## プラットフォーム比較

### Kubernetes vs Render vs Fly.io

| 項目 | Kubernetes | Render | Fly.io |
|------|-----------|--------|--------|
| **複雑さ** | 高い | 低い | 中程度 |
| **学習コスト** | 高い | 低い | 中程度 |
| **柔軟性** | 最高 | 限定的 | 高い |
| **グローバル分散** | 手動設定 | ❌ | 自動 |
| **コスト** | 変動 | 明確 | 明確 |
| **ベンダーロックイン** | ❌ | あり | あり |
| **適用規模** | 小〜大 | 小〜中 | 小〜中 |

### 選択ガイド

**Kubernetesを選ぶべき場合**:
- 大規模で複雑なシステム
- 完全なコントロールが必要
- マルチクラウド戦略
- オンプレミス要件

**Renderを選ぶべき場合**:
- スタートアップのMVP
- シンプルなWebアプリ
- Kubernetes経験がない
- 運用コストを削減したい

**Fly.ioを選ぶべき場合**:
- グローバル展開が必要
- 低レイテンシが重要
- エッジコンピューティング
- Dockerに精通している

---

## 理解度チェック

### Render
- [ ] Renderの利点を説明できる
- [ ] GitHubから自動デプロイできる
- [ ] render.yamlでIaCを実践できる

### Fly.io
- [ ] Fly.ioの特徴を理解している
- [ ] グローバル分散アプリケーションをデプロイできる
- [ ] Fly Postgresを使える

### プラットフォーム選択
- [ ] 各プラットフォームの違いを理解している
- [ ] ユースケースに応じて選択できる

## 📝 まとめ

✅ Renderによるシンプルなデプロイ
✅ Fly.ioによるグローバル分散
✅ プラットフォームの使い分け
✅ Infrastructure as Code

## 🎓 カリキュラム完了おめでとうございます！

すべてのカリキュラムを完了しました！

あなたは以下をマスターしました：
- Kubernetes基礎から高度なトピック
- Helmによるパッケージ管理
- Lensによる可視化
- Linkerd と Istio のサービスメッシュ
- Karpenter と Crossplane の自動化
- Render と Fly.io の抽象化プラットフォーム

**次のステップ**:
1. 実際のプロジェクトに適用する
2. CKA（Certified Kubernetes Administrator）資格に挑戦
3. コミュニティに貢献する

## 🔗 参考リンク

- [Render公式ドキュメント](https://render.com/docs)
- [Fly.io公式ドキュメント](https://fly.io/docs/)
- [Render vs Heroku比較](https://render.com/render-vs-heroku)
- [Fly.io Architecture](https://fly.io/docs/reference/architecture/)
