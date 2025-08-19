# ディレクトリ構成の比較: レイヤー/ドメイン vs ドメイン/レイヤー

## 1. レイヤー/ドメイン構成（レイヤーファースト）

```
idp/
├── entity/
│   ├── user.go
│   ├── client.go
│   ├── token.go
│   └── authorization_code.go
├── usecase/
│   ├── auth/
│   │   ├── authorize.go
│   │   └── authenticate.go
│   ├── token/
│   │   ├── issue_token.go
│   │   └── validate_token.go
│   └── user/
│       └── get_userinfo.go
├── interface/
│   ├── controller/
│   │   ├── auth_controller.go
│   │   └── token_controller.go
│   └── repository/
│       ├── user_repository.go
│       └── client_repository.go
└── infrastructure/
    ├── persistence/
    │   └── postgres/
    └── web/
        └── router.go
```

### メリット
- ✅ クリーンアーキテクチャの層が明確に見える
- ✅ 依存関係の方向が視覚的に理解しやすい
- ✅ 新しい開発者がアーキテクチャを理解しやすい
- ✅ 層ごとの責務が明確

### デメリット
- ❌ 機能追加時に複数の層にファイルを追加する必要がある
- ❌ 関連するファイルが離れた場所に配置される
- ❌ ドメインの境界が曖昧になりやすい

## 2. ドメイン/レイヤー構成（ドメインファースト）

```
idp/
├── auth/
│   ├── entity/
│   │   └── authorization_code.go
│   ├── usecase/
│   │   ├── authorize.go
│   │   └── authenticate.go
│   ├── controller/
│   │   └── auth_controller.go
│   └── repository/
│       └── code_repository.go
├── token/
│   ├── entity/
│   │   └── token.go
│   ├── usecase/
│   │   ├── issue_token.go
│   │   └── validate_token.go
│   ├── controller/
│   │   └── token_controller.go
│   └── repository/
│       └── token_repository.go
├── user/
│   ├── entity/
│   │   └── user.go
│   ├── usecase/
│   │   └── get_userinfo.go
│   ├── controller/
│   │   └── userinfo_controller.go
│   └── repository/
│       └── user_repository.go
└── client/
    ├── entity/
    │   └── client.go
    ├── usecase/
    │   └── validate_client.go
    └── repository/
        └── client_repository.go
```

### メリット
- ✅ 機能ごとにコードがまとまっている
- ✅ ドメインの境界が明確
- ✅ 機能追加・削除が容易
- ✅ マイクロサービスへの移行が簡単

### デメリット
- ❌ クリーンアーキテクチャの層が見えにくい
- ❌ 共通コードの配置が難しい
- ❌ 小規模なドメインではディレクトリが深くなりすぎる

## 3. ハイブリッド構成（推奨）

```
idp/
├── internal/
│   ├── entity/                    # 共通エンティティ層
│   │   ├── user.go
│   │   ├── client.go
│   │   ├── token.go
│   │   └── authorization_code.go
│   ├── usecase/                   # ユースケース層（ドメイン別）
│   │   ├── auth/
│   │   │   ├── authorize.go
│   │   │   ├── authenticate.go
│   │   │   └── interface.go
│   │   ├── token/
│   │   │   ├── issue_token.go
│   │   │   ├── validate_token.go
│   │   │   └── interface.go
│   │   └── user/
│   │       ├── get_userinfo.go
│   │       └── interface.go
│   ├── interface/                 # インターフェース層
│   │   ├── controller/
│   │   │   ├── auth_controller.go
│   │   │   └── token_controller.go
│   │   └── repository/
│   │       ├── user_repository.go
│   │       └── client_repository.go
│   └── infrastructure/            # インフラ層
│       ├── persistence/
│       ├── crypto/
│       └── web/
├── cmd/
│   └── server/
│       └── main.go
└── pkg/                           # 共有パッケージ
    └── oidc/
```

### メリット
- ✅ エンティティは共通層で管理（DRY原則）
- ✅ ユースケースはドメイン別に整理
- ✅ インフラ層は技術別に整理
- ✅ クリーンアーキテクチャの原則を維持

## 推奨: ハイブリッド構成

OIDC OPの場合、以下の理由でハイブリッド構成を推奨します：

1. **エンティティの共有性**: User, Client, Tokenなどは複数のユースケースで使用される
2. **中規模の複雑さ**: 純粋なレイヤー構成では散らばりすぎ、純粋なドメイン構成では重複が多い
3. **段階的な成長**: 将来的にドメインが大きくなったら、その部分だけドメイン構成に移行可能
4. **チーム開発**: 層の責務とドメインの境界の両方が明確

この構成により、クリーンアーキテクチャの原則を守りながら、実用的で保守しやすいコードベースを実現できます。