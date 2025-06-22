# 移行戦略ガイド

## 段階的移行アプローチ

既存のLaravelプロジェクトから新しいADR + UseCaseアーキテクチャへの移行は、段階的に実施することを強く推奨します。

## フェーズ1: 基盤準備

### 1.1 ディレクトリ構造の作成

```bash
# 新しいディレクトリ構造を作成
mkdir -p app/Actions
mkdir -p app/UseCases
mkdir -p app/Responders/Api
mkdir -p app/Responders/Web
mkdir -p app/Support/Enums
mkdir -p app/Support/Traits
mkdir -p app/Exceptions
# app/Providers は Laravel デフォルトで存在
```

### 1.2 基底クラスの作成

#### 共通例外クラス
```php
<?php
// app/Exceptions/BusinessLogicException.php
namespace App\Exceptions;

use Exception;

abstract class BusinessLogicException extends Exception
{
    protected int $statusCode = 400;
    
    public function __construct(string $message, int $statusCode = null)
    {
        parent::__construct($message);
        $this->statusCode = $statusCode ?? $this->statusCode;
    }
    
    public function getStatusCode(): int
    {
        return $this->statusCode;
    }
}
```

#### 基底Responderクラス
```php
<?php
// app/Responders/BaseResponder.php
namespace App\Responders;

use Illuminate\Http\JsonResponse;

abstract class BaseResponder
{
    protected function successResponse(mixed $data, int $status = 200): JsonResponse
    {
        return response()->json([
            'success' => true,
            'data' => $data,
        ], $status);
    }
    
    protected function errorResponse(string $message, int $status = 400): JsonResponse
    {
        return response()->json([
            'success' => false,
            'message' => $message,
        ], $status);
    }
}
```

### 1.3 Enum活用の準備

```php
<?php
// app/Support/Enums/UserStatus.php
namespace App\Support\Enums;

enum UserStatus: string
{
    case ACTIVE = 'active';
    case INACTIVE = 'inactive';
    case SUSPENDED = 'suspended';
    
    public function label(): string
    {
        return match($this) {
            self::ACTIVE => 'アクティブ',
            self::INACTIVE => '非アクティブ',
            self::SUSPENDED => '停止中',
        };
    }
}
```

## フェーズ2: 新機能の新アーキテクチャ適用

### 2.1 新機能開発ルール

新しく開発する機能は、必ず新アーキテクチャで実装してください。

#### 実装例: 新しいコメント機能

1. **UseCase作成**
```php
<?php
// app/UseCases/Comment/CreateCommentUseCase.php
namespace App\UseCases\Comment;

use App\Models\{Post, Comment};
use App\Exceptions\Comment\PostNotCommentableException;
use Illuminate\Support\Facades\DB;

final readonly class CreateCommentUseCase
{
    public function execute(int $postId, int $userId, string $content): Comment
    {
        return DB::transaction(function () use ($postId, $userId, $content) {
            $post = Post::findOrFail($postId);
            
            if (!$post->isCommentable()) {
                throw new PostNotCommentableException();
            }
            
            return Comment::create([
                'post_id' => $postId,
                'user_id' => $userId,
                'content' => $content,
            ]);
        });
    }
}
```

2. **Action作成**
```php
<?php
// app/Actions/Comment/CreateCommentAction.php
namespace App\Actions\Comment;

use App\Http\Requests\CreateCommentRequest;
use App\UseCases\Comment\CreateCommentUseCase;
use App\Responders\Api\CommentResponder;
use Illuminate\Http\JsonResponse;

final readonly class CreateCommentAction
{
    public function __construct(
        private CreateCommentUseCase $useCase,
        private CommentResponder $responder,
    ) {}

    public function __invoke(CreateCommentRequest $request): JsonResponse
    {
        $comment = $this->useCase->execute(
            postId: $request->validated('post_id'),
            userId: $request->user()->id,
            content: $request->validated('content'),
        );

        return $this->responder->created($comment);
    }
}
```

3. **Responder作成**
```php
<?php
// app/Responders/Api/CommentResponder.php
namespace App\Responders\Api;

use App\Models\Comment;
use App\Http\Resources\CommentResource;
use App\Responders\BaseResponder;
use Illuminate\Http\JsonResponse;

final class CommentResponder extends BaseResponder
{
    public function created(Comment $comment): JsonResponse
    {
        return CommentResource::make($comment)
            ->response()
            ->setStatusCode(201);
    }
}
```

### 2.2 既存機能との共存

新アーキテクチャと既存コントローラーが共存する期間は以下に注意：

#### ルーティングの整理
```php
<?php
// routes/api.php

// 新アーキテクチャ
Route::prefix('v1')->group(function () {
    Route::post('/comments', \App\Actions\Comment\CreateCommentAction::class);
});

// 既存アーキテクチャ（段階的に削除予定）
Route::apiResource('posts', PostController::class);
Route::apiResource('users', UserController::class);
```

## フェーズ3: 既存機能の段階的移行

### 3.1 移行優先度の決定

以下の基準で移行優先度を決定してください：

1. **高優先度**
   - 頻繁に変更される機能
   - ビジネスロジックが複雑な機能
   - バグが多発している機能

2. **中優先度**
   - 安定しているが重要な機能
   - 今後拡張予定の機能

3. **低優先度**
   - レガシーで変更頻度が低い機能
   - 近い将来削除予定の機能

### 3.2 移行手順

#### ステップ1: UseCaseの抽出

既存コントローラーからビジネスロジックを抽出してUseCaseを作成：

```php
// Before: app/Http/Controllers/UserController.php
class UserController extends Controller
{
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8',
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        $user->sendEmailVerificationNotification();

        return new UserResource($user);
    }
}

// After: app/UseCases/User/CreateUserUseCase.php
final readonly class CreateUserUseCase
{
    public function execute(string $name, string $email, string $password): User
    {
        return DB::transaction(function () use ($name, $email, $password) {
            $user = User::create([
                'name' => $name,
                'email' => $email,
                'password' => Hash::make($password),
            ]);

            $user->sendEmailVerificationNotification();

            return $user;
        });
    }
}
```

#### ステップ2: コントローラーのUseCaseへの移行

```php
// 移行中: app/Http/Controllers/UserController.php
class UserController extends Controller
{
    public function __construct(
        private CreateUserUseCase $createUserUseCase,
    ) {}

    public function store(CreateUserRequest $request)
    {
        $user = $this->createUserUseCase->execute(
            name: $request->validated('name'),
            email: $request->validated('email'),
            password: $request->validated('password'),
        );

        return new UserResource($user);
    }
}
```

#### ステップ3: Action + Responderへの完全移行

```php
// 最終形: app/Actions/User/CreateUserAction.php
final readonly class CreateUserAction
{
    public function __construct(
        private CreateUserUseCase $useCase,
        private UserResponder $responder,
    ) {}

    public function __invoke(CreateUserRequest $request): JsonResponse
    {
        $user = $this->useCase->execute(
            name: $request->validated('name'),
            email: $request->validated('email'),
            password: $request->validated('password'),
        );

        return $this->responder->created($user);
    }
}
```

## フェーズ4: レガシーコードの削除

### 4.1 削除チェックリスト

移行完了後、以下を確認してから削除：

- [ ] 新アーキテクチャでの動作確認
- [ ] テストケースの移行完了
- [ ] API仕様の互換性確認
- [ ] 関連ドキュメントの更新

### 4.2 段階的削除

```php
// 1. コントローラーメソッドを非推奨マーク
/**
 * @deprecated Use CreateUserAction instead
 */
public function store(Request $request)
{
    // implementation
}

// 2. ルーティングにコメント追加
// TODO: Remove after migration to CreateUserAction
Route::post('/users', [UserController::class, 'store']);

// 3. 完全削除
// ファイル削除 + ルート削除
```

## 移行時の注意点

### 1. API互換性の維持

```php
// 既存APIの互換性を保つためのResponder
class LegacyCompatibleUserResponder extends UserResponder
{
    public function created(User $user): JsonResponse
    {
        // 既存のレスポンス形式を維持
        return response()->json([
            'user' => new UserResource($user),
            'message' => 'User created successfully'
        ], 201);
    }
}
```

### 2. テストケースの移行

```php
// 既存のテストケースを段階的に移行
class UserControllerTest extends TestCase
{
    /** @test */
    public function it_creates_user_via_legacy_endpoint()
    {
        // 既存のテスト（削除予定）
    }
    
    /** @test */
    public function it_creates_user_via_new_action()
    {
        // 新しいテスト
    }
}
```

### 3. モニタリングの強化

移行期間中は以下を監視：

- エラー率の変化
- レスポンス時間の変化
- API使用パターンの変化

### 4. ロールバック計画

問題発生時のロールバック手順を準備：

```bash
# Git tagを使った巻き戻し
git tag -a "pre-migration-v1.0" -m "Before architecture migration"

# 問題発生時
git checkout pre-migration-v1.0
```

## 移行スケジュール例

| フェーズ | 期間 | 成果物 |
|---------|------|--------|
| フェーズ1 | 1週間 | 基盤構築完了 |
| フェーズ2 | 2-4週間 | 新機能の新アーキテクチャ適用 |
| フェーズ3 | 4-8週間 | 既存機能の段階的移行 |
| フェーズ4 | 1-2週間 | レガシーコード削除・整理 |

## 成功指標

移行成功の指標：

- ✅ 新機能開発速度の向上
- ✅ バグ発生率の低下
- ✅ テストカバレッジの向上
- ✅ コードレビュー時間の短縮
- ✅ 開発者の満足度向上

---

*段階的移行により、サービス停止なしでアーキテクチャを改善できます。チーム全体で進捗を共有し、着実に進めましょう。* 