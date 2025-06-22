# Laravel ADR + UseCase アーキテクチャ設計書

## 概要

本プロジェクトでは、Laravel 12を基盤として、ADRパターン（Action-Domain-Responder）とUseCaseパターンを組み合わせた「なんちゃってクリーンアーキテクチャ」を採用しています。

この設計により、以下を実現します：
- ビジネスロジックの明確な分離
- Eloquentモデルの機能をフル活用
- テスタビリティの向上
- 段階的な導入が可能

## 設計思想

### 基本原則

1. **実用性重視**: 理論的な完璧さよりも、開発効率と保守性を重視
2. **Laravel らしさの維持**: フレームワークの恩恵を最大限活用
3. **段階的導入**: 既存コードと共存しながら漸進的に改善
4. **学習コストの最小化**: チーム全体が理解しやすい構造

### アーキテクチャの特徴

- **ADRパターン**: HTTP リクエストの処理フローを明確化
- **UseCaseパターン**: ビジネスロジックを単一責務で分離
- **Direct Eloquent**: Repository パターンを避け、Eloquent を直接活用
- **Type Safety**: PHP 8.3+ の型安全機能をフル活用

## ディレクトリ構造

```
app/
├── Actions/                    # ADRのAction層（エントリポイント）
│   ├── User/
│   │   ├── CreateUserAction.php
│   │   ├── ShowUserAction.php
│   │   ├── UpdateUserAction.php
│   │   ├── DeleteUserAction.php
│   │   └── ListUsersAction.php
│   ├── Community/
│   │   ├── CreateCommunityAction.php
│   │   ├── JoinCommunityAction.php
│   │   └── LeaveCommunityAction.php
│   └── Post/
│       ├── CreatePostAction.php
│       ├── UpdatePostAction.php
│       └── DeletePostAction.php
├── UseCases/                   # ビジネスロジックの中核
│   ├── User/
│   │   ├── CreateUserUseCase.php
│   │   ├── FindUserUseCase.php
│   │   ├── UpdateUserUseCase.php
│   │   ├── DeleteUserUseCase.php
│   │   └── ListUsersUseCase.php
│   ├── Community/
│   │   ├── CreateCommunityUseCase.php
│   │   ├── JoinCommunityUseCase.php
│   │   ├── LeaveCommunityUseCase.php
│   │   └── FindCommunityMembersUseCase.php
│   └── Post/
│       ├── CreatePostUseCase.php
│       ├── UpdatePostUseCase.php
│       ├── DeletePostUseCase.php
│       └── FindPostsUseCase.php
├── Responders/                 # ADRのResponder層（レスポンス整形）
│   ├── Api/
│   │   ├── UserResponder.php
│   │   ├── CommunityResponder.php
│   │   └── PostResponder.php
│   └── Web/
│       ├── UserResponder.php
│       └── CommunityResponder.php
├── Http/                       # Laravel標準（最小限の役割）
│   ├── Controllers/
│   │   └── Controller.php      # 基底クラスのみ保持
│   ├── Requests/               # FormRequest（バリデーション）
│   │   ├── CreateUserRequest.php
│   │   ├── UpdateUserRequest.php
│   │   ├── CreateCommunityRequest.php
│   │   └── CreatePostRequest.php
│   ├── Resources/              # API Resource（シリアライゼーション）
│   │   ├── UserResource.php
│   │   ├── CommunityResource.php
│   │   └── PostResource.php
│   └── Middleware/             # 各種ミドルウェア
├── Models/                     # Eloquentモデル（UseCaseから直接使用）
│   ├── User.php
│   ├── Community.php
│   ├── Post.php
│   └── CommunityUser.php       # 中間テーブル
├── Providers/                  # Laravelサービスプロバイダー
│   ├── AppServiceProvider.php
│   ├── AuthServiceProvider.php
│   ├── EventServiceProvider.php
│   └── RouteServiceProvider.php
├── Policies/                   # 認可ロジック
│   ├── UserPolicy.php
│   ├── CommunityPolicy.php
│   └── PostPolicy.php
├── Services/                   # 外部サービス連携
│   ├── Email/
│   │   └── EmailService.php
│   ├── Storage/
│   │   └── FileUploadService.php
│   └── Notification/
│       └── NotificationService.php
├── Support/                    # ヘルパー・ユーティリティ
│   ├── Enums/
│   │   ├── UserStatus.php
│   │   ├── CommunityType.php
│   │   └── PostStatus.php
│   ├── Traits/
│   │   ├── HasUuid.php
│   │   └── HasTimestamps.php
│   └── Helpers/
│       └── StringHelper.php
└── Exceptions/                 # カスタム例外
    ├── BusinessLogicException.php
    ├── User/
    │   ├── UserNotFoundException.php
    │   └── UserAlreadyExistsException.php
    └── Community/
        ├── CommunityNotFoundException.php
        └── UserNotMemberException.php
```

## 各層の責務

### Action層
- HTTPリクエストの受け取り
- バリデーション済みデータの抽出
- 適切なUseCaseの呼び出し
- Responderによるレスポンス生成

### UseCase層
- ビジネスロジックの実装
- トランザクション管理
- Eloquentモデルの操作
- ドメインルールの実行

### Responder層
- レスポンスデータの整形
- HTTPステータスコードの設定
- API/Web向けの出力調整

### 既存Laravel層
- **Models**: Eloquentモデル（リレーション、アクセサ、ミューテータ）
- **Requests**: バリデーションルール
- **Resources**: APIシリアライゼーション
- **Policies**: 認可ロジック
- **Providers**: サービスプロバイダー（DI設定、アプリケーション初期化）

## データフロー

```
HTTP Request
      ↓
   Action
      ↓
  UseCase ←→ Models (Eloquent)
      ↓
  Responder
      ↓
 HTTP Response
```

1. **Request → Action**: HTTPリクエストを受信
2. **Action → UseCase**: バリデーション済みデータを渡してビジネスロジック実行
3. **UseCase ↔ Models**: EloquentモデルでDB操作・ビジネスルール実行
4. **UseCase → Responder**: 処理結果をResponderに渡す
5. **Responder → Response**: 適切な形式でHTTPレスポンス生成

## Laravel 12 対応機能

### PHP 8.3+ 機能活用
- `readonly class` による不変オブジェクト
- 厳密な型指定
- Enumによる定数管理

### Laravel 12 新機能
- 改善されたルーティング機能
- 強化されたバリデーション
- パフォーマンス向上したEloquent

## テスト戦略

### ユニットテスト
- **UseCase**: ビジネスロジックの単体テスト
- **Models**: Eloquentモデルのテスト
- **Services**: 外部サービス連携のテスト

### 機能テスト
- **Action**: HTTPエンドポイントのテスト
- **Integration**: 複数UseCaseの連携テスト

### テストディレクトリ構造
```
tests/
├── Unit/
│   ├── UseCases/
│   ├── Models/
│   └── Services/
├── Feature/
│   ├── Actions/
│   └── Integration/
└── TestCase.php
```

## 導入メリット

### 開発効率
- ✅ ビジネスロジックの所在が明確
- ✅ 単一責務による変更影響の局所化
- ✅ Eloquentの強力な機能をフル活用

### 保守性
- ✅ 層が明確で理解しやすい
- ✅ テストが書きやすい
- ✅ 段階的な機能追加・修正が容易

### 拡張性
- ✅ 新機能の追加パターンが統一
- ✅ 既存コードへの影響最小化
- ✅ チーム開発でのコード品質維持

## 注意事項

### 適用範囲
- 中小規模のWebアプリケーションに最適
- 超大規模・複雑なドメインには本格的DDDを推奨
- チームの技術レベルに応じた導入深度の調整が必要

### 制約事項
- Eloquentへの依存が前提
- Laravelフレームワークとの密結合
- データベース変更時の影響範囲が広い

---

*このドキュメントは Laravel 12 ベースで作成されています。バージョンアップ時は適宜更新してください。* 