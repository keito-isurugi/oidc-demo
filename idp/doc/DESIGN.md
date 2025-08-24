# OIDC OP (OpenID Provider) 設計書

## 1. 概要

### 1.1 目的
OpenID Connect仕様に準拠した認証プロバイダー(OP)をGoで実装する。

### 1.2 スコープ
- OpenID Connect Core 1.0の基本機能
- Authorization Code Flow
- JWT形式のIDトークン発行
- Discovery機能
- 基本的なユーザー管理

### 1.3 技術スタック
- 言語: Go 1.24+ (最新版: Go 1.24.6)
- Webフレームワーク: Gin (高性能) または Echo v4 (最新版: v4.13.4)
- JWT: github.com/golang-jwt/jwt/v5 (最新メジャーバージョン)
- データベース: PostgreSQL 17+ (最新安定版: 17.6)
- ORM: GORM v2 (最新版、開発者フレンドリー)

## 2. アーキテクチャ

### 2.1 クリーンアーキテクチャ構成

```
┌──────────────────────────────────────────────────────┐
│                  External Systems                     │
│         (Browsers, OAuth Clients, Admin Tools)        │
└────────────────────┬─────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────┐
│              Frameworks & Drivers Layer              │
│        (Web Framework, Database, External APIs)       │
├───────────────────────────────────────────────────────┤
│              Interface Adapters Layer                 │
│   (Controllers, Presenters, Gateways, Repositories)   │
├───────────────────────────────────────────────────────┤
│              Application Business Rules               │
│                    (Use Cases)                        │
├───────────────────────────────────────────────────────┤
│              Enterprise Business Rules                │
│                    (Entities)                         │
└───────────────────────────────────────────────────────┘

依存関係の方向: 外側 → 内側のみ (依存性逆転の原則)
```

### 2.2 プロジェクト構造
```
oidc-demo/
├── cmd/
│   └── server/
│       └── main.go                    # エントリーポイント・DI設定
├── internal/
│   ├── entity/                        # Enterprise Business Rules
│   │   ├── user.go                    # ユーザーエンティティ
│   │   ├── client.go                  # クライアントエンティティ  
│   │   ├── authorization_code.go      # 認可コードエンティティ
│   │   ├── token.go                   # トークンエンティティ
│   │   └── errors.go                  # ドメインエラー
│   ├── usecase/                       # Application Business Rules
│   │   ├── auth/
│   │   │   ├── authorize.go           # 認可ユースケース
│   │   │   ├── authenticate.go        # 認証ユースケース
│   │   │   └── interface.go           # 必要なインターフェース定義
│   │   ├── token/
│   │   │   ├── issue_token.go         # トークン発行ユースケース
│   │   │   ├── refresh_token.go       # トークン更新ユースケース
│   │   │   ├── validate_token.go      # トークン検証ユースケース
│   │   │   └── interface.go           # 必要なインターフェース定義
│   │   ├── user/
│   │   │   ├── get_userinfo.go        # ユーザー情報取得ユースケース
│   │   │   ├── create_user.go         # ユーザー作成ユースケース
│   │   │   ├── update_user.go         # ユーザー更新ユースケース
│   │   │   └── interface.go           # 必要なインターフェース定義
│   │   └── client/
│   │       ├── register_client.go     # クライアント登録ユースケース
│   │       ├── validate_client.go     # クライアント検証ユースケース
│   │       └── interface.go           # 必要なインターフェース定義
│   ├── interface/                      # Interface Adapters Layer
│   │   ├── controller/                 # Webコントローラー
│   │   │   ├── auth_controller.go     # 認証コントローラー
│   │   │   ├── token_controller.go    # トークンコントローラー
│   │   │   ├── userinfo_controller.go # ユーザー情報コントローラー
│   │   │   ├── discovery_controller.go # Discoveryコントローラー
│   │   │   ├── admin_controller.go    # 管理コントローラー
│   │   │   └── dto/                   # データ転送オブジェクト
│   │   │       ├── request.go         # リクエストDTO
│   │   │       └── response.go        # レスポンスDTO
│   │   ├── presenter/                  # プレゼンター
│   │   │   ├── auth_presenter.go      # 認証プレゼンター
│   │   │   ├── token_presenter.go     # トークンプレゼンター
│   │   │   └── error_presenter.go     # エラープレゼンター
│   │   └── repository/                 # リポジトリ実装
│   │       ├── user_repository.go     # ユーザーリポジトリ
│   │       ├── client_repository.go   # クライアントリポジトリ
│   │       ├── code_repository.go     # 認可コードリポジトリ
│   │       └── token_repository.go    # トークンリポジトリ
│   └── infrastructure/                 # Frameworks & Drivers Layer
│       ├── config/
│       │   └── config.go              # 設定管理
│       ├── web/
│       │   ├── router.go              # ルーティング設定
│       │   └── middleware/
│       │       ├── auth.go            # 認証ミドルウェア
│       │       ├── cors.go            # CORSミドルウェア
│       │       └── logging.go         # ロギングミドルウェア
│       ├── persistence/
│       │   ├── postgres/              # PostgreSQL実装
│       │   │   ├── connection.go      # DB接続
│       │   │   ├── user_store.go      # ユーザーストア
│       │   │   ├── client_store.go    # クライアントストア
│       │   │   ├── code_store.go      # 認可コードストア
│       │   │   └── token_store.go     # トークンストア
│       │   ├── memory/                # インメモリ実装（開発・テスト用）
│       │   │   ├── user_store.go
│       │   │   ├── client_store.go
│       │   │   ├── code_store.go
│       │   │   └── token_store.go
│       │   └── models/                # DBモデル
│       │       ├── user_model.go
│       │       ├── client_model.go
│       │       ├── code_model.go
│       │       └── token_model.go
│       ├── crypto/
│       │   ├── jwt_service.go         # JWT処理実装
│       │   ├── hash_service.go        # ハッシュ処理実装
│       │   └── key_manager.go         # 鍵管理
│       └── logger/
│           └── logger.go              # ロガー実装
├── pkg/                                # 外部パッケージとして切り出し可能な汎用コード
│   ├── oidc/
│   │   ├── constants.go              # OIDC定数
│   │   ├── claims.go                 # クレーム定義
│   │   └── scopes.go                 # スコープ定義
│   └── utils/
│       ├── random.go                  # ランダム文字列生成
│       └── validator.go              # バリデーション
├── migrations/
│   └── *.sql                         # DBマイグレーション
├── keys/                              # JWT署名用の鍵
│   ├── private.pem
│   └── public.pem
├── test/
│   ├── integration/                  # 統合テスト
│   └── e2e/                         # E2Eテスト
├── docker/
│   └── Dockerfile
├── docker-compose.yml
├── Makefile                          # ビルド・実行タスク
├── go.mod
├── go.sum
└── README.md
```

### 2.3 依存性注入とインターフェース

クリーンアーキテクチャの原則に従い、各層は抽象（インターフェース）に依存し、具象実装には依存しません。

#### Use Case層のインターフェース例
```go
// internal/usecase/auth/interface.go
package auth

type UserRepository interface {
    FindByID(ctx context.Context, id string) (*entity.User, error)
    FindByUsername(ctx context.Context, username string) (*entity.User, error)
}

type ClientRepository interface {
    FindByID(ctx context.Context, id string) (*entity.Client, error)
    ValidateRedirectURI(ctx context.Context, clientID, redirectURI string) error
}

type CodeRepository interface {
    Save(ctx context.Context, code *entity.AuthorizationCode) error
    FindByCode(ctx context.Context, code string) (*entity.AuthorizationCode, error)
    Delete(ctx context.Context, code string) error
}

type TokenGenerator interface {
    GenerateAccessToken(user *entity.User, client *entity.Client, scopes []string) (string, error)
    GenerateIDToken(user *entity.User, client *entity.Client, nonce string) (string, error)
}

type PasswordHasher interface {
    Hash(password string) (string, error)
    Verify(password, hash string) error
}
```

#### 依存性注入の実装例（main.go）
```go
// cmd/server/main.go
func main() {
    // Infrastructure層の初期化
    db := initDatabase()
    jwtService := crypto.NewJWTService(privateKey, publicKey)
    hashService := crypto.NewHashService()
    
    // Repository実装の初期化
    userRepo := postgres.NewUserRepository(db)
    clientRepo := postgres.NewClientRepository(db)
    codeRepo := postgres.NewCodeRepository(db)
    tokenRepo := postgres.NewTokenRepository(db)
    
    // Use Case層の初期化（依存性注入）
    authUseCase := auth.NewAuthorizeUseCase(
        userRepo, 
        clientRepo, 
        codeRepo,
        hashService,
    )
    tokenUseCase := token.NewIssueTokenUseCase(
        codeRepo,
        tokenRepo,
        jwtService,
    )
    
    // Controller層の初期化
    authController := controller.NewAuthController(authUseCase)
    tokenController := controller.NewTokenController(tokenUseCase)
    
    // ルーター設定
    router := web.NewRouter(
        authController,
        tokenController,
        // ...
    )
    
    // サーバー起動
    router.Run(":8080")
}
```

## 3. エンドポイント設計

### 3.1 必須エンドポイント

#### 3.1.1 Discovery
- **GET** `/.well-known/openid-configuration`
  - OpenID Provider設定情報を返す

#### 3.1.2 Authorization
- **GET/POST** `/authorize`
  - パラメータ:
    - response_type (required): "code"
    - client_id (required)
    - redirect_uri (required)
    - scope (required): "openid"を含む
    - state (recommended)
    - nonce (optional)
    - code_challenge (PKCE, optional)
    - code_challenge_method (PKCE, optional)

#### 3.1.3 Token
- **POST** `/token`
  - Content-Type: application/x-www-form-urlencoded
  - パラメータ:
    - grant_type (required): "authorization_code"
    - code (required)
    - redirect_uri (required if used in /authorize)
    - client_id (required)
    - client_secret (confidential client)
    - code_verifier (PKCE, optional)

#### 3.1.4 UserInfo
- **GET/POST** `/userinfo`
  - Authorization: Bearer {access_token}
  - 返却: ユーザー情報のJSON

#### 3.1.5 JWKS
- **GET** `/jwks`
  - JWKセットを返す（公開鍵情報）

### 3.2 管理用エンドポイント

#### 3.2.1 Client管理
- **POST** `/admin/clients` - クライアント登録
- **GET** `/admin/clients` - クライアント一覧
- **GET** `/admin/clients/{id}` - クライアント詳細
- **PUT** `/admin/clients/{id}` - クライアント更新
- **DELETE** `/admin/clients/{id}` - クライアント削除

#### 3.2.2 User管理
- **POST** `/admin/users` - ユーザー登録
- **GET** `/admin/users` - ユーザー一覧
- **GET** `/admin/users/{id}` - ユーザー詳細
- **PUT** `/admin/users/{id}` - ユーザー更新
- **DELETE** `/admin/users/{id}` - ユーザー削除

## 4. データモデル

### 4.1 User
```go
type User struct {
    ID           string    
    Username     string    
    Email        string    
    PasswordHash string    
    Name         string    
    GivenName    string    
    FamilyName   string    
    Picture      string    
    CreatedAt    time.Time 
    UpdatedAt    time.Time 
}
```

### 4.2 Client
```go
type Client struct {
    ID                string   
    Secret            string   
    Name              string   
    RedirectURIs      []string 
    GrantTypes        []string 
    ResponseTypes     []string 
    Scopes            []string 
    TokenEndpointAuth string   // client_secret_basic, client_secret_post, none
    CreatedAt         time.Time
    UpdatedAt         time.Time
}
```

### 4.3 AuthorizationCode
```go
type AuthorizationCode struct {
    Code              string
    ClientID          string
    UserID            string
    RedirectURI       string
    Scopes            []string
    Nonce             string
    CodeChallenge     string
    CodeChallengeMethod string
    ExpiresAt         time.Time
    CreatedAt         time.Time
}
```

### 4.4 Token
```go
type Token struct {
    ID           string
    Type         string    // access_token, refresh_token
    Value        string
    ClientID     string
    UserID       string
    Scopes       []string
    ExpiresAt    time.Time
    CreatedAt    time.Time
}
```

## 5. セキュリティ要件

### 5.1 認証・認可
- パスワードはbcryptでハッシュ化
- クライアントシークレットは安全に管理
- PKCE (Proof Key for Code Exchange) サポート
- state パラメータによるCSRF対策

### 5.2 トークン
- JWTはRS256で署名
- アクセストークンの有効期限: 1時間
- リフレッシュトークンの有効期限: 30日
- 認可コードの有効期限: 10分

### 5.3 通信
- HTTPS必須（開発環境除く）
- 適切なCORS設定

### 5.4 監査
- 認証試行のログ記録
- トークン発行のログ記録
- 不正なアクセスの検知

## 6. 設定項目

### 6.1 環境変数
```env
# Server
PORT=8080
HOST=localhost

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=oidc_demo
DB_USER=oidc_user
DB_PASSWORD=secret

# JWT
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PUBLIC_KEY_PATH=./keys/public.pem

# Token
ACCESS_TOKEN_TTL=3600
REFRESH_TOKEN_TTL=2592000
AUTH_CODE_TTL=600

# Admin
ADMIN_USERNAME=admin
ADMIN_PASSWORD=admin_password

# Environment
ENV=development
```

## 7. 実装フェーズ

### Phase 1: 基盤構築
1. プロジェクトセットアップ
2. 基本的なHTTPサーバー
3. データベース接続
4. 設定管理

### Phase 2: コア機能
1. ユーザー管理機能
2. クライアント管理機能
3. JWT生成・検証

### Phase 3: OIDC実装
1. Authorization エンドポイント
2. Token エンドポイント
3. UserInfo エンドポイント
4. Discovery エンドポイント

### Phase 4: セキュリティ強化
1. PKCE実装
2. レート制限
3. 監査ログ

### Phase 5: テスト・最適化
1. ユニットテスト
2. 統合テスト
3. パフォーマンス最適化

## 8. テスト戦略

### 8.1 ユニットテスト
- 各サービスレイヤーのテスト
- JWT処理のテスト
- バリデーションのテスト

### 8.2 統合テスト
- 認証フロー全体のテスト
- エラーケースのテスト
- セキュリティテスト

### 8.3 E2Eテスト
- 実際のクライアントを使用したテスト
- 複数のグラントタイプのテスト

## 9. 参考仕様

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [OAuth 2.0 PKCE RFC 7636](https://tools.ietf.org/html/rfc7636)
- [JSON Web Token RFC 7519](https://tools.ietf.org/html/rfc7519)