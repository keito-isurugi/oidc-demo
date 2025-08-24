# ローカル開発環境セットアップガイド

## 必要なツール

### 必須
- **Go**: 1.24+ (最新版: Go 1.24.6)
- **Docker**: 28.0+ (最新版)
- **Docker Compose**: 2.39+ (最新版)
- **Make**: GNU Make 3.81以上

### 推奨
- **golangci-lint**: 最新版（コード品質チェック）
- **migrate**: 最新版（DBマイグレーション）
- **air**: 最新版（ホットリロード）
- **jq**: 1.6以上（JSON処理）
- **openssl**: 1.1.1以上（鍵生成）

## インストール手順

### 1. macOS (Homebrew使用)

```bash
# Go
brew install go

# Docker Desktop
brew install --cask docker

# 開発ツール
brew install make jq openssl

# Go開発ツール
go install github.com/cosmtrek/air@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

### 2. Ubuntu/Debian

```bash
# Go (snap使用)
sudo snap install go --classic

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 開発ツール
sudo apt-get update
sudo apt-get install -y make jq openssl

# Go開発ツール
go install github.com/cosmtrek/air@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

## プロジェクトセットアップ

### 1. リポジトリのクローン

```bash
git clone https://github.com/yourusername/oidc-demo.git
cd oidc-demo/idp
```

### 2. 環境変数の設定

```bash
# .env.localファイルを作成
cp .env.example .env.local

# 必要に応じて編集
vim .env.local
```

#### .env.local の内容

```env
# Server Configuration
SERVER_HOST=localhost
SERVER_PORT=8080
SERVER_ENV=development

# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=oidc_demo
DB_USER=oidc_user
DB_PASSWORD=oidc_password
DB_SSLMODE=disable

# Redis Configuration (Session Store)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# JWT Configuration
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PUBLIC_KEY_PATH=./keys/public.pem
JWT_ISSUER=http://localhost:8080

# Token Expiration (seconds)
ACCESS_TOKEN_TTL=3600
REFRESH_TOKEN_TTL=2592000
AUTH_CODE_TTL=600

# Admin Configuration
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin123

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001
CORS_ALLOWED_METHODS=GET,POST,PUT,DELETE,OPTIONS
CORS_ALLOWED_HEADERS=Authorization,Content-Type

# Logging
LOG_LEVEL=debug
LOG_FORMAT=json
```

### 3. JWT鍵の生成

```bash
# 鍵ディレクトリの作成
mkdir -p keys

# RSA秘密鍵の生成
openssl genrsa -out keys/private.pem 2048

# RSA公開鍵の生成
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# 権限設定
chmod 600 keys/private.pem
chmod 644 keys/public.pem
```

### 4. Dockerコンテナの起動

```bash
# PostgreSQLとRedisを起動
docker-compose up -d postgres redis

# ログ確認
docker-compose logs -f postgres redis
```

#### docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:17-alpine
    container_name: oidc_postgres
    environment:
      POSTGRES_DB: oidc_demo
      POSTGRES_USER: oidc_user
      POSTGRES_PASSWORD: oidc_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U oidc_user -d oidc_demo"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:8-alpine
    container_name: oidc_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # 開発用のmailhog（メール確認用）
  mailhog:
    image: mailhog/mailhog
    container_name: oidc_mailhog
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI
    profiles:
      - dev

volumes:
  postgres_data:
  redis_data:
```

### 5. データベースマイグレーション

```bash
# マイグレーション実行
make migrate-up

# または直接実行
migrate -path ./migrations -database "postgresql://oidc_user:oidc_password@localhost:5432/oidc_demo?sslmode=disable" up
```

### 6. アプリケーションの起動

```bash
# 依存パッケージのインストール
go mod download

# 通常起動
make run

# ホットリロード付き起動（air使用）
make dev

# または直接実行
air -c .air.toml
```

#### .air.toml

```toml
root = "."
tmp_dir = "tmp"

[build]
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ./cmd/server"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "keys", "migrations"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  kill_delay = "0s"
  log = "build-errors.log"
  send_interrupt = false
  stop_on_error = true

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  time = false

[misc]
  clean_on_exit = false
```

## Makefile

```makefile
.PHONY: help
help:
	@echo "Available commands:"
	@echo "  make setup       - Initial setup"
	@echo "  make dev        - Start development server with hot reload"
	@echo "  make run        - Start server"
	@echo "  make build      - Build binary"
	@echo "  make test       - Run tests"
	@echo "  make lint       - Run linter"
	@echo "  make migrate-up - Run migrations"
	@echo "  make migrate-down - Rollback migrations"
	@echo "  make docker-up  - Start Docker containers"
	@echo "  make docker-down - Stop Docker containers"
	@echo "  make clean      - Clean build artifacts"

.PHONY: setup
setup:
	@echo "Setting up development environment..."
	@mkdir -p keys
	@test -f keys/private.pem || openssl genrsa -out keys/private.pem 2048
	@test -f keys/public.pem || openssl rsa -in keys/private.pem -pubout -out keys/public.pem
	@test -f .env.local || cp .env.example .env.local
	@go mod download
	@echo "Setup complete!"

.PHONY: dev
dev:
	@air -c .air.toml

.PHONY: run
run:
	@go run ./cmd/server

.PHONY: build
build:
	@go build -o bin/idp ./cmd/server

.PHONY: test
test:
	@go test -v -cover ./...

.PHONY: test-integration
test-integration:
	@go test -v -tags=integration ./test/integration/...

.PHONY: lint
lint:
	@golangci-lint run

.PHONY: migrate-up
migrate-up:
	@migrate -path ./migrations -database "postgresql://oidc_user:oidc_password@localhost:5432/oidc_demo?sslmode=disable" up

.PHONY: migrate-down
migrate-down:
	@migrate -path ./migrations -database "postgresql://oidc_user:oidc_password@localhost:5432/oidc_demo?sslmode=disable" down

.PHONY: migrate-create
migrate-create:
	@read -p "Enter migration name: " name; \
	migrate create -ext sql -dir ./migrations -seq $$name

.PHONY: docker-up
docker-up:
	@docker-compose up -d postgres redis

.PHONY: docker-down
docker-down:
	@docker-compose down

.PHONY: docker-logs
docker-logs:
	@docker-compose logs -f

.PHONY: clean
clean:
	@rm -rf bin/ tmp/
	@go clean -cache
```

## 動作確認

### 1. ヘルスチェック

```bash
curl http://localhost:8080/health
# Expected: {"status":"ok"}
```

### 2. OpenID Configuration

```bash
curl http://localhost:8080/.well-known/openid-configuration | jq
```

### 3. テストクライアントの登録

```bash
# Admin APIでクライアント登録
curl -X POST http://localhost:8080/admin/clients \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d '{
    "client_name": "Test Client",
    "redirect_uris": ["http://localhost:3000/callback"],
    "grant_types": ["authorization_code"],
    "response_types": ["code"],
    "scope": "openid profile email"
  }' | jq
```

### 4. テストユーザーの作成

```bash
curl -X POST http://localhost:8080/admin/users \
  -H "Content-Type: application/json" \
  -u admin:admin123 \
  -d '{
    "username": "testuser",
    "email": "test@example.com",
    "password": "password123",
    "name": "Test User"
  }' | jq
```

## トラブルシューティング

### ポート競合

```bash
# 使用中のポートを確認
lsof -i :8080
lsof -i :5432
lsof -i :6379

# プロセスを終了
kill -9 <PID>
```

### データベース接続エラー

```bash
# PostgreSQLコンテナの状態確認
docker-compose ps postgres
docker-compose logs postgres

# 接続テスト
psql -h localhost -U oidc_user -d oidc_demo
```

### 権限エラー

```bash
# 鍵ファイルの権限確認
ls -la keys/

# 権限修正
chmod 600 keys/private.pem
chmod 644 keys/public.pem
```

## VSCode設定

### .vscode/launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Server",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/cmd/server",
      "envFile": "${workspaceFolder}/.env.local",
      "args": []
    },
    {
      "name": "Debug Tests",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${workspaceFolder}/...",
      "args": ["-test.v"]
    }
  ]
}
```

### .vscode/settings.json

```json
{
  "go.useLanguageServer": true,
  "go.lintTool": "golangci-lint",
  "go.lintOnSave": "package",
  "go.testOnSave": false,
  "go.coverOnSave": false,
  "go.testFlags": ["-v"],
  "go.testTimeout": "30s",
  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    }
  },
  "files.exclude": {
    "**/.git": true,
    "**/.DS_Store": true,
    "tmp": true,
    "bin": true
  }
}
```

## 便利なコマンド

```bash
# ログをリアルタイムで確認
docker-compose logs -f postgres redis

# データベースに接続
docker-compose exec postgres psql -U oidc_user -d oidc_demo

# Redisに接続
docker-compose exec redis redis-cli

# テストカバレッジ確認
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# 依存関係の更新
go mod tidy
go mod vendor

# セキュリティチェック
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```