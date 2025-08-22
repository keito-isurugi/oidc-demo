# OIDC Demo - OpenID Connect Provider Implementation

GoによるOpenID Connect Provider (OP) の実装デモプロジェクトです。クリーンアーキテクチャを採用し、OIDC Core 1.0仕様に準拠した認証プロバイダーを構築します。

## 特徴

- 🏛️ **クリーンアーキテクチャ**: 依存性逆転の原則に基づいた設計
- 🔐 **OpenID Connect Core 1.0準拠**: 標準的なOIDCフローの実装
- 🔑 **JWT (RS256)**: 安全なトークン発行
- 🛡️ **PKCE対応**: より安全な認証フロー
- 🐳 **Docker対応**: 簡単な環境構築
- ♻️ **ホットリロード**: 開発効率の向上

## クイックスタート

### 前提条件

- Go 1.21以上
- Docker & Docker Compose
- Make

### セットアップ

```bash
# リポジトリのクローン
git clone https://github.com/yourusername/oidc-demo.git
cd oidc-demo

# idpディレクトリへ移動
cd idp

# 環境変数ファイルの作成
cp .env.example .env.local

# JWT署名用の鍵を生成
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# Dockerコンテナの起動（PostgreSQL, Redis）
docker-compose up -d

# データベースマイグレーション
make migrate-up

# アプリケーションの起動
make dev  # ホットリロード付き
# または
make run  # 通常起動
```

### 動作確認

```bash
# ヘルスチェック
curl http://localhost:8080/health

# OpenID Configuration
curl http://localhost:8080/.well-known/openid-configuration
```

## プロジェクト構造

```
oidc-demo/
├── idp/                    # Identity Provider実装
│   ├── cmd/
│   │   └── server/        # アプリケーションエントリーポイント
│   ├── internal/
│   │   ├── entity/        # ビジネスエンティティ
│   │   ├── usecase/       # ビジネスロジック
│   │   ├── interface/     # インターフェースアダプター
│   │   └── infrastructure/ # 外部サービス実装
│   ├── keys/              # JWT署名鍵
│   ├── migrations/        # DBマイグレーション
│   └── pkg/              # 共有パッケージ
└── README.md
```

詳細な構造については [idp/DESIGN.md](idp/DESIGN.md) を参照してください。

## 主要なエンドポイント

### OIDC標準エンドポイント

| エンドポイント | メソッド | 説明 |
|------------|---------|------|
| `/.well-known/openid-configuration` | GET | Discovery情報 |
| `/authorize` | GET/POST | 認可エンドポイント |
| `/token` | POST | トークンエンドポイント |
| `/userinfo` | GET/POST | ユーザー情報エンドポイント |
| `/jwks` | GET | 公開鍵セット |

### 管理用エンドポイント

| エンドポイント | メソッド | 説明 |
|------------|---------|------|
| `/admin/clients` | POST | クライアント登録 |
| `/admin/clients/{id}` | GET/PUT/DELETE | クライアント管理 |
| `/admin/users` | POST | ユーザー登録 |
| `/admin/users/{id}` | GET/PUT/DELETE | ユーザー管理 |

## 開発

### 便利なMakeコマンド

```bash
make help         # ヘルプ表示
make setup        # 初期セットアップ
make dev          # 開発サーバー起動（ホットリロード）
make test         # テスト実行
make lint         # リンター実行
make build        # バイナリビルド
make docker-up    # Dockerコンテナ起動
make docker-down  # Dockerコンテナ停止
```

### テスト実行

```bash
# ユニットテスト
make test

# 統合テスト
make test-integration

# カバレッジレポート
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### 環境変数

主要な環境変数（`.env.local`）:

```env
# サーバー設定
SERVER_HOST=localhost
SERVER_PORT=8080

# データベース設定
DB_HOST=localhost
DB_PORT=5432
DB_NAME=oidc_demo
DB_USER=oidc_user
DB_PASSWORD=oidc_password

# JWT設定
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PUBLIC_KEY_PATH=./keys/public.pem

# トークン有効期限（秒）
ACCESS_TOKEN_TTL=3600
REFRESH_TOKEN_TTL=2592000
```

完全なリストは [.env.example](.env.example) を参照してください。

## Docker環境

### コンテナ構成

- **postgres**: PostgreSQL 15 (データベース)
- **redis**: Redis 7 (セッションストア)
- **mailhog**: MailHog (開発用メールサーバー) ※オプション

### Docker Composeコマンド

```bash
# コンテナ起動
docker-compose up -d

# ログ確認
docker-compose logs -f

# コンテナ停止
docker-compose down

# ボリューム含めて削除
docker-compose down -v
```

## トラブルシューティング

### ポート競合エラー

```bash
# ポート使用状況確認
lsof -i :8080

# プロセス終了
kill -9 <PID>
```

### データベース接続エラー

```bash
# PostgreSQL状態確認
docker-compose ps postgres
docker-compose logs postgres

# データベース接続テスト
psql -h localhost -U oidc_user -d oidc_demo
```

### 鍵ファイルの権限エラー

```bash
# 適切な権限に修正
chmod 600 keys/private.pem
chmod 644 keys/public.pem
```

詳細は [idp/LOCAL_SETUP.md](idp/LOCAL_SETUP.md) を参照してください。

## ドキュメント

- [設計書](idp/DESIGN.md) - アーキテクチャと設計の詳細
- [ローカル環境構築](idp/LOCAL_SETUP.md) - 詳細なセットアップ手順
- [ディレクトリ構成](idp/DIRECTORY_STRUCTURE_COMPARISON.md) - プロジェクト構造の説明

## セキュリティ

- パスワードはbcryptでハッシュ化
- JWT署名にはRS256を使用
- PKCE (Proof Key for Code Exchange) サポート
- CSRF対策のstateパラメータ
- HTTPS必須（本番環境）

## ライセンス

MIT

## 貢献

プルリクエストを歓迎します。大きな変更の場合は、まずissueを開いて変更内容を議論してください。

## サポート

問題が発生した場合は、[Issues](https://github.com/yourusername/oidc-demo/issues) で報告してください。