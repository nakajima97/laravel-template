# 実装例集

## 完全な実装例

### ユーザー管理機能の完全実装

#### 1. Eloquentモデル

```php
<?php
// app/Models/User.php
namespace App\Models;

use App\Support\Enums\UserStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use HasFactory, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'status',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
            'status' => UserStatus::class,
        ];
    }
    
    // リレーション
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
    
    public function communities()
    {
        return $this->belongsToMany(Community::class)
                    ->withPivot('role', 'joined_at')
                    ->withTimestamps();
    }
    
    // ビジネスメソッド
    public function isActive(): bool
    {
        return $this->status === UserStatus::ACTIVE;
    }
    
    public function canJoinCommunity(Community $community): bool
    {
        return $this->isActive() && !$this->communities->contains($community);
    }
}
```

#### 2. UseCase実装

```php
<?php
// app/UseCases/User/CreateUserUseCase.php
namespace App\UseCases\User;

use App\Models\User;
use App\Support\Enums\UserStatus;
use App\Exceptions\User\UserAlreadyExistsException;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;

final readonly class CreateUserUseCase
{
    public function execute(string $name, string $email, string $password): User
    {
        return DB::transaction(function () use ($name, $email, $password) {
            // ビジネスルールバリデーション
            if (User::where('email', $email)->exists()) {
                throw new UserAlreadyExistsException($email);
            }
            
            // ユーザー作成
            $user = User::create([
                'name' => $name,
                'email' => $email,
                'password' => Hash::make($password),
                'status' => UserStatus::ACTIVE,
            ]);
            
            // 副作用処理
            $user->sendEmailVerificationNotification();
            
            return $user;
        });
    }
}
```

```php
<?php
// app/UseCases/User/UpdateUserUseCase.php
namespace App\UseCases\User;

use App\Models\User;
use App\Exceptions\User\UserNotFoundException;
use Illuminate\Support\Facades\DB;

final readonly class UpdateUserUseCase
{
    public function execute(int $userId, string $name, ?string $email = null): User
    {
        return DB::transaction(function () use ($userId, $name, $email) {
            $user = User::find($userId);
            
            if (!$user) {
                throw new UserNotFoundException($userId);
            }
            
            // 更新データの準備
            $updateData = ['name' => $name];
            
            if ($email && $email !== $user->email) {
                // メールアドレス変更時のビジネスルール
                if (User::where('email', $email)->where('id', '!=', $userId)->exists()) {
                    throw new UserAlreadyExistsException($email);
                }
                
                $updateData['email'] = $email;
                $updateData['email_verified_at'] = null; // 再認証が必要
            }
            
            $user->update($updateData);
            
            // メールアドレス変更時は再認証メール送信
            if (isset($updateData['email'])) {
                $user->sendEmailVerificationNotification();
            }
            
            return $user->fresh();
        });
    }
}
```

#### 3. Action実装

```php
<?php
// app/Actions/User/CreateUserAction.php
namespace App\Actions\User;

use App\Http\Requests\CreateUserRequest;
use App\UseCases\User\CreateUserUseCase;
use App\Responders\Api\UserResponder;
use Illuminate\Http\JsonResponse;

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

```php
<?php
// app/Actions/User/UpdateUserAction.php
namespace App\Actions\User;

use App\Http\Requests\UpdateUserRequest;
use App\UseCases\User\UpdateUserUseCase;
use App\Responders\Api\UserResponder;
use Illuminate\Http\JsonResponse;

final readonly class UpdateUserAction
{
    public function __construct(
        private UpdateUserUseCase $useCase,
        private UserResponder $responder,
    ) {}

    public function __invoke(UpdateUserRequest $request, int $userId): JsonResponse
    {
        $user = $this->useCase->execute(
            userId: $userId,
            name: $request->validated('name'),
            email: $request->validated('email'),
        );

        return $this->responder->updated($user);
    }
}
```

#### 4. Responder実装

```php
<?php
// app/Responders/Api/UserResponder.php
namespace App\Responders\Api;

use App\Models\User;
use App\Http\Resources\UserResource;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

final class UserResponder
{
    public function created(User $user): JsonResponse
    {
        return UserResource::make($user)
            ->response()
            ->setStatusCode(201)
            ->header('Location', route('users.show', $user));
    }
    
    public function show(User $user): JsonResponse
    {
        return UserResource::make($user)->response();
    }
    
    public function updated(User $user): JsonResponse
    {
        return UserResource::make($user)->response();
    }
    
    public function deleted(): JsonResponse
    {
        return response()->json([
            'message' => 'User deleted successfully'
        ], 204);
    }
    
    public function index(AnonymousResourceCollection $users): JsonResponse
    {
        return $users->response();
    }
}
```

#### 5. FormRequest

```php
<?php
// app/Http/Requests/CreateUserRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class CreateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // または適切な認可ロジック
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'string', 'min:8', 'confirmed'],
        ];
    }
    
    public function messages(): array
    {
        return [
            'name.required' => 'ユーザー名は必須です。',
            'email.required' => 'メールアドレスは必須です。',
            'email.unique' => 'このメールアドレスは既に使用されています。',
            'password.required' => 'パスワードは必須です。',
            'password.min' => 'パスワードは8文字以上である必要があります。',
        ];
    }
}
```

#### 6. APIResource

```php
<?php
// app/Http/Resources/UserResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'status' => [
                'value' => $this->status->value,
                'label' => $this->status->label(),
            ],
            'email_verified_at' => $this->email_verified_at?->toISOString(),
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
            
            // 必要に応じてリレーションを含める
            'posts_count' => $this->whenCounted('posts'),
            'communities' => CommunityResource::collection($this->whenLoaded('communities')),
        ];
    }
}
```

#### 7. カスタム例外

```php
<?php
// app/Exceptions/User/UserAlreadyExistsException.php
namespace App\Exceptions\User;

use App\Exceptions\BusinessLogicException;

class UserAlreadyExistsException extends BusinessLogicException
{
    public function __construct(string $email)
    {
        parent::__construct(
            "User with email '{$email}' already exists",
            409
        );
    }
}
```

```php
<?php
// app/Exceptions/User/UserNotFoundException.php
namespace App\Exceptions\User;

use App\Exceptions\BusinessLogicException;

class UserNotFoundException extends BusinessLogicException
{
    public function __construct(int $userId)
    {
        parent::__construct(
            "User with ID '{$userId}' not found",
            404
        );
    }
}
```

#### 8. ルート定義

```php
<?php
// routes/api.php
use App\Actions\User\{CreateUserAction, ShowUserAction, UpdateUserAction, DeleteUserAction};
use Illuminate\Support\Facades\Route;

Route::prefix('v1')->group(function () {
    Route::post('/users', CreateUserAction::class)->name('users.store');
    Route::get('/users/{user}', ShowUserAction::class)->name('users.show');
    Route::put('/users/{user}', UpdateUserAction::class)->name('users.update');
    Route::delete('/users/{user}', DeleteUserAction::class)->name('users.destroy');
});
```

## コミュニティ参加機能の実装例

### 複雑なビジネスロジックを含むUseCase

```php
<?php
// app/UseCases/Community/JoinCommunityUseCase.php
namespace App\UseCases\Community;

use App\Models\{User, Community};
use App\Support\Enums\{UserStatus, CommunityType};
use App\Exceptions\Community\{CommunityNotFoundException, UserAlreadyMemberException, CommunityFullException};
use Illuminate\Support\Facades\DB;

final readonly class JoinCommunityUseCase
{
    public function execute(int $userId, int $communityId): void
    {
        DB::transaction(function () use ($userId, $communityId) {
            // エンティティ取得
            $user = User::findOrFail($userId);
            $community = Community::findOrFail($communityId);
            
            // ビジネスルールバリデーション
            $this->validateJoinRequest($user, $community);
            
            // コミュニティ参加処理
            $community->users()->attach($userId, [
                'role' => 'member',
                'joined_at' => now(),
            ]);
            
            // 副作用処理
            $this->handleSideEffects($user, $community);
        });
    }
    
    private function validateJoinRequest(User $user, Community $community): void
    {
        // ユーザーステータス確認
        if (!$user->isActive()) {
            throw new \InvalidArgumentException('User is not active');
        }
        
        // 既にメンバーかチェック
        if ($community->users->contains($user)) {
            throw new UserAlreadyMemberException($user->id, $community->id);
        }
        
        // コミュニティの参加可能性チェック
        if ($community->type === CommunityType::PRIVATE) {
            throw new \InvalidArgumentException('Cannot join private community');
        }
        
        // 定員チェック
        if ($community->max_members && $community->users->count() >= $community->max_members) {
            throw new CommunityFullException($community->id);
        }
    }
    
    private function handleSideEffects(User $user, Community $community): void
    {
        // ウェルカム通知送信
        $user->notify(new \App\Notifications\CommunityJoinedNotification($community));
        
        // 既存メンバーへの通知
        $community->users->each(function ($member) use ($user, $community) {
            if ($member->id !== $user->id) {
                $member->notify(new \App\Notifications\NewMemberJoinedNotification($user, $community));
            }
        });
        
        // ログ記録
        \Log::info('User joined community', [
            'user_id' => $user->id,
            'community_id' => $community->id,
        ]);
    }
}
```

## テストケース例

### UseCaseのテスト

```php
<?php
// tests/Unit/UseCases/User/CreateUserUseCaseTest.php
namespace Tests\Unit\UseCases\User;

use App\UseCases\User\CreateUserUseCase;
use App\Models\User;
use App\Support\Enums\UserStatus;
use App\Exceptions\User\UserAlreadyExistsException;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Notification;

class CreateUserUseCaseTest extends TestCase
{
    use RefreshDatabase;
    
    private CreateUserUseCase $useCase;
    
    protected function setUp(): void
    {
        parent::setUp();
        $this->useCase = new CreateUserUseCase();
        Notification::fake();
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
        $this->assertEquals(UserStatus::ACTIVE, $user->status);
        $this->assertTrue(Hash::check($password, $user->password));
        
        $this->assertDatabaseHas('users', [
            'name' => $name,
            'email' => $email,
            'status' => UserStatus::ACTIVE->value,
        ]);
        
        // 通知が送信されたことを確認
        Notification::assertSentTo($user, \Illuminate\Auth\Notifications\VerifyEmail::class);
    }
    
    public function test_throws_exception_when_email_already_exists(): void
    {
        // Arrange
        User::factory()->create(['email' => 'test@example.com']);
        
        // Act & Assert
        $this->expectException(UserAlreadyExistsException::class);
        $this->expectExceptionMessage("User with email 'test@example.com' already exists");
        
        $this->useCase->execute('Test User', 'test@example.com', 'password123');
    }
}
```

### Actionのテスト

```php
<?php
// tests/Feature/Actions/User/CreateUserActionTest.php
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
                         'status' => ['value', 'label'],
                         'created_at',
                         'updated_at',
                     ]
                 ])
                 ->assertJson([
                     'data' => [
                         'name' => 'Test User',
                         'email' => 'test@example.com',
                         'status' => [
                             'value' => 'active',
                             'label' => 'アクティブ',
                         ],
                     ]
                 ]);
                 
        $this->assertDatabaseHas('users', [
            'name' => 'Test User',
            'email' => 'test@example.com',
        ]);
    }
    
    public function test_returns_validation_error_for_invalid_data(): void
    {
        // Arrange
        $invalidData = [
            'name' => '',
            'email' => 'invalid-email',
            'password' => '123',
        ];
        
        // Act
        $response = $this->postJson('/api/v1/users', $invalidData);
        
        // Assert
        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['name', 'email', 'password']);
    }
}
```

---

*これらの実装例を参考に、プロジェクトに適した形で実装してください。* 