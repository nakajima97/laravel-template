# Laravel ADR + UseCase アーキテクチャ ドキュメント

## 概要

このドキュメント群は、Laravel 12を基盤としたADRパターン（Action-Domain-Responder）とUseCaseパターンを組み合わせた「なんちゃってクリーンアーキテクチャ」の設計・実装・移行について解説しています。

参考記事: [5年間 Laravel を使って辿り着いた，全然頑張らない「なんちゃってクリーンアーキテクチャ」という落としどころ](https://zenn.dev/mpyw/articles/ce7d09eb6d8117)

## 特徴

- ✅ **Laravel 12 対応**: 最新機能の活用
- ✅ **実用性重視**: 理論より開発効率・保守性
- ✅ **段階的導入**: 既存プロジェクトとの共存
- ✅ **Eloquent フル活用**: Repository パターンを避けた軽量設計
- ✅ **型安全**: PHP 8.3+ の機能を最大限活用

## ドキュメント構成

### 📋 [architecture.md](./architecture.md)
**メインの設計書**
- アーキテクチャ全体の設計思想
- ディレクトリ構造の詳細
- 各層の責務定義
- データフローの解説

### 🛠️ [implementation-guide.md](./implementation-guide.md)
**実装ガイドライン**
- 新機能開発の手順
- 実装テンプレート集
- コーディング規約
- テスト戦略

### 🔄 [migration-strategy.md](./migration-strategy.md)
**移行戦略ガイド**
- 段階的移行アプローチ
- フェーズ別実装手順
- 既存コードとの共存方法
- リスク管理

### 💻 [examples.md](./examples.md)
**実装例集**
- 完全な機能実装例
- 複雑なビジネスロジックの実装
- テストケースの書き方
- ベストプラクティス

## クイックスタート

### 1. 新規プロジェクトの場合

```bash
# Laravel 12 プロジェクト作成
composer create-project laravel/laravel your-project

# ディレクトリ構造作成
mkdir -p app/{Actions,UseCases,Responders/{Api,Web},Support/{Enums,Traits},Exceptions}

# 基底クラス作成後、新機能は新アーキテクチャで実装
```

### 2. 既存プロジェクトの場合

1. **フェーズ1**: 基盤準備（1週間）
2. **フェーズ2**: 新機能の新アーキテクチャ適用（2-4週間）
3. **フェーズ3**: 既存機能の段階的移行（4-8週間）
4. **フェーズ4**: レガシーコード削除（1-2週間）

詳細は [migration-strategy.md](./migration-strategy.md) を参照してください。

## アーキテクチャ概要

### ディレクトリ構造
```
app/
├── Actions/        # ADRのAction層（エントリポイント）
├── UseCases/       # ビジネスロジックの中核
├── Responders/     # レスポンス整形
├── Http/           # Laravel標準（最小限）
├── Models/         # Eloquentモデル
├── Support/        # ヘルパー・ユーティリティ
└── Exceptions/     # カスタム例外
```

### データフロー
```
HTTP Request → Action → UseCase ↔ Models → Responder → HTTP Response
```

### 実装例
```php
// Action（エントリポイント）
final readonly class CreateUserAction
{
    public function __invoke(CreateUserRequest $request): JsonResponse
    {
        $user = $this->useCase->execute(...$request->validated());
        return $this->responder->created($user);
    }
}

// UseCase（ビジネスロジック）
final readonly class CreateUserUseCase
{
    public function execute(string $name, string $email, string $password): User
    {
        return DB::transaction(function () use ($name, $email, $password) {
            // Eloquentを直接使用
            return User::create([
                'name' => $name,
                'email' => $email,
                'password' => Hash::make($password),
            ]);
        });
    }
}
```

## なぜこのアーキテクチャなのか？

### 従来の問題点
- **Fat Controller**: コントローラーにビジネスロジックが集中
- **Fat Model**: Eloquentモデルが肥大化
- **テストしにくい**: 責務が混在してテストが困難

### 解決策
- **単一責務**: 各クラスが明確な責務を持つ
- **軽量設計**: Repository パターンを避けてEloquentの恩恵を活用
- **段階的導入**: 既存コードと共存しながら改善

## 適用範囲

### 適している場合
- 中小規模のWebアプリケーション
- Eloquentの機能をフル活用したい
- 段階的な改善を求める既存プロジェクト
- チーム全体で理解しやすい設計を求める

### 適していない場合
- 超大規模・複雑なドメインロジック
- データベース非依存な設計が必要
- マイクロサービス間の複雑な連携

## 次のステップ

1. **設計理解**: [architecture.md](./architecture.md) でアーキテクチャ全体を理解
2. **実装準備**: [implementation-guide.md](./implementation-guide.md) で実装方法を学習
3. **実装開始**: [examples.md](./examples.md) の例を参考に実装
4. **移行計画**: [migration-strategy.md](./migration-strategy.md) で移行戦略を策定

## 追加リソース

### 関連記事
- [元記事: なんちゃってクリーンアーキテクチャ](https://zenn.dev/mpyw/articles/ce7d09eb6d8117)
- [Laravel 12 公式ドキュメント](https://laravel.com/docs)
- [ADRパターンについて](https://github.com/pmjones/adr)

### コミュニティ
- Laravel Japan User Group
- PHP コミュニティ
- GitHub Issues（このプロジェクト）

---

*このアーキテクチャにより、保守性と開発効率の両立を実現し、チーム全体で持続可能な開発を行えます。* 