# 実装ガイドライン

## 実装手順

### 1. 新機能開発の流れ

新機能を開発する際は、以下の順序で実装してください：

```
1. Models → 2. UseCases → 3. Actions → 4. Responders → 5. Routes
```

#### 詳細手順

1. **Eloquentモデルの準備**
   - マイグレーション作成
   - モデルクラス作成（リレーション定義）
   - Factory・Seeder作成

2. **UseCaseの実装**
   - ビジネスロジックの実装
   - テストケース作成

3. **Actionの実装**
   - リクエスト処理の実装
   - FormRequestの作成

4. **Responderの実装**
   - レスポンス整形ロジック
   - Resourceクラスの活用

5. **ルート定義**
   - api.php または web.php への追加

### 2. 実装テンプレート

#### UseCase実装テンプレート

```php
<?php
namespace App\UseCases\{Domain};

use App\Models\{Model};
use Illuminate\Support\Facades\DB;

final readonly class {Action}{Domain}UseCase
{
    public function execute({parameters}): {ReturnType}
    {
        return DB::transaction(function () use ({parameters}) {
            // ビジネスロジックの実装
            
            // バリデーション（ビジネスルール）
            $this->validateBusinessRules({parameters});
            
            // メイン処理
            $result = $this->performMainLogic({parameters});
            
            // 副作用処理（通知、ログ等）
            $this->handleSideEffects($result);
            
            return $result;
        });
    }
    
    private function validateBusinessRules({parameters}): void
    {
        // ビジネスルールのバリデーション
    }
    
    private function performMainLogic({parameters}): {ReturnType}
    {
        // メインのビジネスロジック
    }
    
    private function handleSideEffects({ReturnType} $result): void
    {
        // 副作用の処理（オプション）
    }
}
```

#### Action実装テンプレート

```php
<?php
namespace App\Actions\{Domain};

use App\Http\Requests\{Request};
use App\UseCases\{Domain}\{UseCase};
use App\Responders\Api\{Responder};
use Illuminate\Http\JsonResponse;

final readonly class {Action}{Domain}Action
{
    public function __construct(
        private {UseCase} $useCase,
        private {Responder} $responder,
    ) {}

    public function __invoke({Request} $request): JsonResponse
    {
        $result = $this->useCase->execute(
            // バリデーション済みデータを渡す
            ...$request->validated()
        );

        return $this->responder->{method}($result);
    }
}
```

#### Responder実装テンプレート

```php
<?php
namespace App\Responders\Api;

use App\Models\{Model};
use App\Http\Resources\{Resource};
use Illuminate\Http\JsonResponse;

final class {Domain}Responder
{
    public function created({Model} $model): JsonResponse
    {
        return {Resource}::make($model)
            ->response()
            ->setStatusCode(201);
    }
    
    public function show({Model} $model): JsonResponse
    {
        return {Resource}::make($model)->response();
    }
    
    public function updated({Model} $model): JsonResponse
    {
        return {Resource}::make($model)->response();
    }
    
    public function deleted(): JsonResponse
    {
        return response()->json(['message' => 'Deleted successfully'], 204);
    }
    
    public function index($collection): JsonResponse
    {
        return {Resource}::collection($collection)->response();
    }
}
```

## コーディング規約

### 1. 命名規則

#### クラス名
- **UseCase**: `{動詞}{ドメイン}UseCase`
  ```php
  CreateUserUseCase
  UpdatePostUseCase
  FindCommunityMembersUseCase
  ```

- **Action**: `{動詞}{ドメイン}Action`
  ```php
  CreateUserAction
  ShowPostAction
  DeleteCommunityAction
  ```

- **Responder**: `{ドメイン}Responder`
  ```php
  UserResponder
  PostResponder
  CommunityResponder
  ```

#### メソッド名
- **UseCase**: `execute()` で統一
- **Action**: `__invoke()` で統一（Single Action Controller）
- **Responder**: HTTP動詞に対応
  ```php
  created()    // POST (201)
  show()       // GET (200)
  updated()    // PUT/PATCH (200)
  deleted()    // DELETE (204)
  index()      // GET Collection (200)
  ```

### 2. 型指定ルール

#### 厳密な型指定を必須とする

```php
// ✅ Good
public function execute(string $name, int $age): User
{
    // implementation
}

// ❌ Bad
public function execute($name, $age)
{
    // implementation
}
```

#### readonly classの活用

```php
// ✅ Good
final readonly class CreateUserUseCase
{
    public function __construct(
        private EmailService $emailService,
    ) {}
}
```

### 3. 例外処理

#### カスタム例外の使用

```php
<?php
namespace App\Exceptions\User;

use App\Exceptions\BusinessLogicException;

class UserAlreadyExistsException extends BusinessLogicException
{
    public function __construct(string $email)
    {
        parent::__construct("User with email '{$email}' already exists", 409);
    }
}
```

#### UseCase内での例外スロー

```php
public function execute(string $email, string $password): User
{
    if (User::where('email', $email)->exists()) {
        throw new UserAlreadyExistsException($email);
    }
    
    // 正常処理
}
```

### 4. トランザクション管理

#### UseCaseでのDB::transaction使用

```php
public function execute(array $userData, array $profileData): User
{
    return DB::transaction(function () use ($userData, $profileData) {
        $user = User::create($userData);
        $user->profile()->create($profileData);
        
        // 他の関連処理
        
        return $user;
    });
}
```

### 5. テスト実装規約

#### UseCaseのテスト

```php
<?php
namespace Tests\Unit\UseCases\User;

use App\UseCases\User\CreateUserUseCase;
use App\Models\User;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class CreateUserUseCaseTest extends TestCase
{
    use RefreshDatabase;
    
    private CreateUserUseCase $useCase;
    
    protected function setUp(): void
    {
        parent::setUp();
        $this->useCase = new CreateUserUseCase();
    }
    
    public function test_creates_user_successfully(): void
    {
        // Arrange
        $name = 'Test User';
        $email = 'test@example.com';
        $password = 'password123';
        
        // Act
        $user = $this->useCase->execute($name, $email, $password);
        
        // Assert
        $this->assertInstanceOf(User::class, $user);
        $this->assertEquals($name, $user->name);
        $this->assertEquals($email, $user->email);
        $this->assertDatabaseHas('users', [
            'name' => $name,
            'email' => $email,
        ]);
    }
    
    public function test_throws_exception_when_email_already_exists(): void
    {
        // Arrange
        User::factory()->create(['email' => 'test@example.com']);
        
        // Act & Assert
        $this->expectException(UserAlreadyExistsException::class);
        $this->useCase->execute('Test User', 'test@example.com', 'password');
    }
}
```

#### Actionのテスト

```php
<?php
namespace Tests\Feature\Actions\User;

use App\Models\User;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class CreateUserActionTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_creates_user_via_api(): void
    {
        // Arrange
        $userData = [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password123',
            'password_confirmation' => 'password123',
        ];
        
        // Act
        $response = $this->postJson('/api/v1/users', $userData);
        
        // Assert
        $response->assertStatus(201)
                 ->assertJsonStructure([
                     'data' => [
                         'id',
                         'name',
                         'email',
                         'created_at',
                         'updated_at',
                     ]
                 ]);
                 
        $this->assertDatabaseHas('users', [
            'name' => 'Test User',
            'email' => 'test@example.com',
        ]);
    }
}
```

## ベストプラクティス

### 1. UseCaseの単一責務原則

```php
// ✅ Good: 単一責務
class CreateUserUseCase
{
    public function execute(string $name, string $email, string $password): User
    {
        // ユーザー作成のみに集中
    }
}

class SendWelcomeEmailUseCase
{
    public function execute(User $user): void
    {
        // ウェルカムメール送信のみに集中
    }
}

// ❌ Bad: 複数責務
class CreateUserAndSendEmailUseCase
{
    public function execute(string $name, string $email, string $password): User
    {
        // ユーザー作成 + メール送信が混在
    }
}
```

### 2. Eloquentの効果的な活用

```php
// ✅ Good: Eloquentの機能をフル活用
public function execute(int $communityId, int $userId): void
{
    $community = Community::findOrFail($communityId);
    $user = User::findOrFail($userId);
    
    // リレーション使用
    $community->users()->attach($userId, [
        'joined_at' => now(),
        'role' => 'member',
    ]);
    
    // モデルイベント活用
    $user->notify(new CommunityJoinedNotification($community));
}
```

### 3. 適切な抽象化レベル

```php
// ✅ Good: 適切な抽象化
public function execute(int $postId, string $content): Comment
{
    $post = Post::findOrFail($postId);
    
    $this->authorizeCommentCreation($post);
    
    return $this->createComment($post, $content);
}

// ❌ Bad: 抽象化不足
public function execute(int $postId, string $content): Comment
{
    $post = Post::where('id', $postId)->first();
    if (!$post) {
        throw new ModelNotFoundException();
    }
    if ($post->status !== 'published') {
        throw new UnauthorizedException();
    }
    if (auth()->user()->cannot('comment', $post)) {
        throw new ForbiddenException();
    }
    $comment = new Comment();
    $comment->post_id = $postId;
    $comment->user_id = auth()->id();
    $comment->content = $content;
    $comment->save();
    return $comment;
}
```

## パフォーマンス考慮事項

### 1. N+1問題の対策

```php
// ✅ Good: Eager Loading
public function execute(): Collection
{
    return Post::with(['user', 'community', 'comments.user'])
        ->published()
        ->latest()
        ->get();
}

// ❌ Bad: N+1 Problem
public function execute(): Collection
{
    return Post::published()->latest()->get();
    // テンプレートで $post->user, $post->community などを参照すると N+1 発生
}
```

### 2. 大量データの処理

```php
// ✅ Good: Chunk処理
public function execute(): void
{
    User::where('last_login_at', '<', now()->subDays(30))
        ->chunk(100, function ($users) {
            foreach ($users as $user) {
                $user->sendInactiveWarning();
            }
        });
}
```

---

*このガイドラインに従って実装することで、一貫性のあるコードベースを維持できます。* 