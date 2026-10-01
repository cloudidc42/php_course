# Part 20: Laravel Eloquent ORM
## ขั้นตอนที่ 511-540: Eloquent ทั้งหมด

---

## ขั้นตอนที่ 511: Eloquent Model พื้นฐาน

```php
<?php
// app/Models/User.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model {
    use HasFactory, SoftDeletes;
    
    // กำหนดตาราง (ถ้าไม่ใช่ Default)
    protected $table = 'users';
    
    // Primary Key
    protected $primaryKey = 'id';
    
    // Timestamps
    public $timestamps = true;
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at';
    
    // Fillable (Mass Assignment)
    protected $fillable = [
        'name', 'email', 'password', 'role', 'bio', 'avatar'
    ];
    
    // Guarded (ไม่ให้ Mass Assign)
    // protected $guarded = ['id', 'admin'];
    
    // Hidden (JSON/Array)
    protected $hidden = ['password', 'remember_token'];
    
    // Append (Custom Attributes)
    protected $appends = ['full_name', 'avatar_url'];
    
    // Casts
    protected function casts(): array {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
            'preferences' => 'array',
            'metadata' => 'json',
            'active' => 'boolean',
            'score' => 'float',
        ];
    }
    
    // Custom Accessor (PHP 8.0+ way)
    protected function fullName(): Attribute {
        return Attribute::make(
            get: fn() => "{$this->name} ({$this->role})"
        );
    }
    
    protected function avatarUrl(): Attribute {
        return Attribute::make(
            get: fn() => $this->avatar
                ? asset('storage/' . $this->avatar)
                : 'https://ui-avatars.com/api/?name=' . urlencode($this->name)
        );
    }
    
    // Mutator
    protected function email(): Attribute {
        return Attribute::make(
            get: fn($value) => strtolower($value),
            set: fn($value) => strtolower(trim($value))
        );
    }
    
    // Scopes
    public function scopeActive($query) {
        return $query->where('active', true);
    }
    
    public function scopeRole($query, string $role) {
        return $query->where('role', $role);
    }
    
    public function scopeSearch($query, string $search) {
        return $query->where(function ($q) use ($search) {
            $q->where('name', 'LIKE', "%{$search}%")
              ->orWhere('email', 'LIKE', "%{$search}%");
        });
    }
    
    // Relationships
    public function posts() {
        return $this->hasMany(Post::class);
    }
    
    public function profile() {
        return $this->hasOne(Profile::class);
    }
    
    public function roles() {
        return $this->belongsToMany(Role::class)->withTimestamps();
    }
    
    // Methods
    public function isAdmin(): bool {
        return $this->role === 'admin';
    }
}
```

---

## ขั้นตอนที่ 512: Eloquent CRUD

```php
<?php
use App\Models\User;

// ========================
// CREATE
// ========================

// Create with save()
$user = new User();
$user->name = 'Alice';
$user->email = 'alice@example.com';
$user->password = 'secret';
$user->save();

// Create with create() (Mass Assignment - ต้อง fillable)
$user = User::create([
    'name' => 'Bob',
    'email' => 'bob@example.com',
    'password' => 'secret',
]);

// firstOrCreate - ถ้าไม่มีสร้างใหม่
$user = User::firstOrCreate(
    ['email' => 'charlie@example.com'],
    ['name' => 'Charlie', 'password' => 'secret']
);

// firstOrNew - เหมือน firstOrCreate แต่ไม่บันทึก
$user = User::firstOrNew(['email' => 'dave@example.com']);

// updateOrCreate - อัพเดทหรือสร้างใหม่
$user = User::updateOrCreate(
    ['email' => 'eve@example.com'],
    ['name' => 'Eve', 'role' => 'editor']
);

// ========================
// READ
// ========================

// All records
$users = User::all();
$users = User::all(['id', 'name', 'email']); // เฉพาะบางคอลัมน์

// Find by Primary Key
$user = User::find(1);
$user = User::find([1, 2, 3]);  // Multiple IDs
$user = User::findOrFail(1);    // Throw ModelNotFoundException

// First/Last
$user = User::first();
$user = User::latest()->first();
$user = User::oldest('created_at')->first();
$user = User::firstWhere('email', 'alice@example.com');

// Where Clauses
$users = User::where('active', true)->get();
$users = User::where('age', '>', 18)->where('role', 'admin')->get();
$users = User::whereIn('role', ['admin', 'editor'])->get();
$users = User::whereNotIn('role', ['banned'])->get();
$users = User::whereBetween('age', [18, 65])->get();
$users = User::whereNull('deleted_at')->get();
$users = User::whereNotNull('email_verified_at')->get();

// Scopes
$users = User::active()->role('admin')->get();
$results = User::search('alice')->paginate(15);

// Select specific columns
$users = User::select('id', 'name', 'email')->get();
$users = User::select('role', DB::raw('COUNT(*) as count'))
            ->groupBy('role')
            ->get();

// Ordering
$users = User::orderBy('name')->get();
$users = User::orderBy('name', 'desc')->get();
$users = User::latest()->get();   // orderBy created_at desc
$users = User::oldest()->get();   // orderBy created_at asc

// Pagination
$users = User::paginate(15);
$users = User::simplePaginate(15);  // Previous/Next only
$users = User::cursorPaginate(15);  // Cursor-based (efficient)

// Chunks (สำหรับข้อมูลจำนวนมาก)
User::chunk(100, function ($users) {
    foreach ($users as $user) {
        // Process each user
    }
});

// Lazy Collections (ประหยัด Memory)
User::lazy()->each(fn($user) => processUser($user));

// Count, Sum, Avg, Min, Max
$count = User::count();
$count = User::where('active', true)->count();
$avg = User::avg('age');
$max = User::max('salary');
$sum = User::where('active', true)->sum('salary');

// Exists
$exists = User::where('email', 'alice@example.com')->exists();
$doesNotExist = User::where('email', 'alien@mars.com')->doesntExist();

// ========================
// UPDATE
// ========================

// Update single model
$user = User::findOrFail(1);
$user->name = 'Alice Updated';
$user->save();

// update() on query
User::where('active', false)->update(['role' => 'inactive']);

// Increment/Decrement
User::where('id', 1)->increment('login_count');
User::where('id', 1)->decrement('credits', 10);
$user->increment('views');

// Fill and Save
$user->fill(['name' => 'New Name', 'bio' => 'New Bio'])->save();

// ========================
// DELETE
// ========================

// Delete single
$user = User::findOrFail(1);
$user->delete();

// Delete by query
User::where('active', false)->delete();

// Soft Delete (ต้อง use SoftDeletes)
$user->delete();           // Soft delete (sets deleted_at)
$user->restore();          // Restore
$user->forceDelete();      // Hard delete

// Include soft-deleted
$users = User::withTrashed()->get();
$users = User::onlyTrashed()->get();
```

---

## ขั้นตอนที่ 513: Relationships ทุกประเภท

```php
<?php
// ========================
// 1. One to One
// ========================
class User extends Model {
    public function profile() {
        return $this->hasOne(Profile::class, 'user_id', 'id');
    }
}

class Profile extends Model {
    public function user() {
        return $this->belongsTo(User::class, 'user_id', 'id');
    }
}

$user = User::find(1);
$profile = $user->profile;           // Access relationship
$userName = $user->profile->bio;     // Access nested

// Eager Loading
$users = User::with('profile')->get();
$user = User::with('profile:id,user_id,bio')->find(1);

// ========================
// 2. One to Many
// ========================
class User extends Model {
    public function posts() {
        return $this->hasMany(Post::class);
    }
    
    public function publishedPosts() {
        return $this->hasMany(Post::class)
            ->where('status', 'published')
            ->orderBy('published_at', 'desc');
    }
}

class Post extends Model {
    public function author() {
        return $this->belongsTo(User::class, 'user_id');
    }
}

$posts = $user->posts;               // All posts
$posts = $user->posts()->latest()->take(5)->get();
$post = $user->posts()->create(['title' => 'New Post', ...]);

// ========================
// 3. Many to Many
// ========================
class Post extends Model {
    public function tags() {
        return $this->belongsToMany(Tag::class, 'post_tags', 'post_id', 'tag_id')
                    ->withTimestamps()
                    ->withPivot('order');
    }
}

class Tag extends Model {
    public function posts() {
        return $this->belongsToMany(Post::class, 'post_tags');
    }
}

// Attach/Detach/Sync
$post->tags()->attach([1, 2, 3]);
$post->tags()->detach([1]);
$post->tags()->sync([1, 2, 3]);      // Replace all
$post->tags()->syncWithoutDetaching([4]); // Add without removing
$post->tags()->toggle([1, 2]);       // Toggle

// Pivot
$post->tags()->attach(1, ['order' => 1]);
foreach ($post->tags as $tag) {
    echo $tag->pivot->order;
}

// ========================
// 4. Has Many Through
// ========================
class Country extends Model {
    // Country → Users → Posts
    public function posts() {
        return $this->hasManyThrough(Post::class, User::class);
    }
}

// ========================
// 5. Polymorphic
// ========================
class Comment extends Model {
    public function commentable() {
        return $this->morphTo();
    }
}

class Post extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

class Video extends Model {
    public function comments() {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Usage
$post->comments()->create(['body' => 'Nice post!']);
$video->comments()->create(['body' => 'Great video!']);

// ========================
// 6. Many to Many Polymorphic
// ========================
class Tag extends Model {
    public function posts() {
        return $this->morphedByMany(Post::class, 'taggable');
    }
    
    public function videos() {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}

class Post extends Model {
    public function tags() {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

// ========================
// Eager Loading Techniques
// ========================

// Basic
$users = User::with(['posts', 'profile'])->get();

// Nested
$users = User::with('posts.comments.author')->get();

// Conditional
$users = User::with(['posts' => function($q) {
    $q->where('status', 'published')->latest();
}])->get();

// Lazy Eager Loading
$users = User::all();
$users->load('posts');
$users->loadMissing('profile');

// Prevent N+1 Problem
User::preventLazyLoading(!app()->isProduction());

// Count Relationships
$users = User::withCount('posts')->get();
$users = User::withCount(['posts', 'comments'])->get();
$users = User::withCount([
    'posts',
    'posts as published_count' => fn($q) => $q->where('status', 'published')
])->get();

foreach ($users as $user) {
    echo "{$user->name}: {$user->posts_count} posts\n";
}

// Exists/Doesnt Exist
$usersWithPosts = User::whereHas('posts')->get();
$usersWithoutPosts = User::whereDoesntHave('posts')->get();
$activePosters = User::whereHas('posts', fn($q) => 
    $q->where('status', 'published')->where('views', '>', 100)
)->get();
```

---

## ขั้นตอนที่ 514: Eloquent Collections

```php
<?php
// Eloquent returns Collection (extends LazyCollection)
$users = User::all();  // Returns Illuminate\Database\Eloquent\Collection

// All Laravel Collection methods work
$names = $users->pluck('name');
$emails = $users->pluck('email', 'id');
$admins = $users->where('role', 'admin');
$grouped = $users->groupBy('role');
$sorted = $users->sortBy('name');

// Map to array
$data = $users->map(fn($user) => [
    'id' => $user->id,
    'name' => $user->name,
    'post_count' => $user->posts->count()
]);

// Lazy Collections สำหรับข้อมูลมาก
$result = User::lazy()
    ->filter(fn($user) => $user->active)
    ->map(fn($user) => $user->only(['id', 'name', 'email']))
    ->values();

// Custom Collections
class UserCollection extends \Illuminate\Database\Eloquent\Collection {
    public function admins(): static {
        return $this->where('role', 'admin');
    }
    
    public function active(): static {
        return $this->where('active', true);
    }
    
    public function averageAge(): float {
        return $this->avg('age') ?? 0;
    }
}

// ใน User Model
public function newCollection(array $models = []): UserCollection {
    return new UserCollection($models);
}
```

---

## ขั้นตอนที่ 515: Advanced Eloquent

```php
<?php
// ========================
// Observer Pattern
// ========================
namespace App\Observers;

class UserObserver {
    public function creating(User $user): void {
        $user->slug = Str::slug($user->name) . '-' . Str::random(5);
    }
    
    public function created(User $user): void {
        // Send welcome email
        Mail::to($user)->send(new WelcomeEmail($user));
    }
    
    public function updating(User $user): void {
        if ($user->isDirty('email')) {
            $user->email_verified_at = null;
        }
    }
    
    public function deleting(User $user): void {
        // Clean up related data
        $user->posts()->delete();
    }
}

// Register in AppServiceProvider
User::observe(UserObserver::class);

// ========================
// Events in Model
// ========================
class Post extends Model {
    protected $dispatchesEvents = [
        'creating' => PostCreating::class,
        'created'  => PostCreated::class,
        'updated'  => PostUpdated::class,
        'deleted'  => PostDeleted::class,
    ];
}

// ========================
// Global Scopes
// ========================
class ActiveScope implements Scope {
    public function apply(Builder $builder, Model $model): void {
        $builder->where('active', true);
    }
}

class User extends Model {
    protected static function booted(): void {
        static::addGlobalScope(new ActiveScope);
        
        // Or inline:
        static::addGlobalScope('active', fn($q) => $q->where('active', true));
    }
}

// Remove global scope
User::withoutGlobalScope(ActiveScope::class)->get();
User::withoutGlobalScopes()->get();

// ========================
// Query Macros
// ========================
use Illuminate\Database\Eloquent\Builder;

Builder::macro('published', function () {
    return $this->where('status', 'published')
                ->whereNotNull('published_at')
                ->where('published_at', '<=', now());
});

// Usage
Post::published()->latest('published_at')->get();

// ========================
// Casts
// ========================
use Illuminate\Database\Eloquent\Casts\AsCollection;
use Illuminate\Database\Eloquent\Casts\AsEncryptedCollection;

class User extends Model {
    protected function casts(): array {
        return [
            'metadata'    => 'array',
            'settings'    => AsCollection::class,
            'secret_data' => AsEncryptedCollection::class,
            'birth_date'  => 'date:Y-m-d',
            'last_login'  => 'datetime:Y-m-d H:i:s',
            'score'       => 'decimal:2',
        ];
    }
}

// Custom Cast
class Money implements CastsAttributes {
    public function get($model, $key, $value, $attributes): array {
        return [
            'amount' => $value,
            'currency' => $attributes['currency'] ?? 'THB',
            'formatted' => number_format($value, 2) . ' ฿',
        ];
    }
    
    public function set($model, $key, $value, $attributes): int {
        return is_array($value) ? $value['amount'] : $value;
    }
}

// ========================
// Factories
// ========================
class UserFactory extends Factory {
    public function definition(): array {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'password' => 'password',
            'role' => fake()->randomElement(['admin', 'editor', 'user']),
            'active' => true,
        ];
    }
    
    // States
    public function admin(): static {
        return $this->state(['role' => 'admin']);
    }
    
    public function inactive(): static {
        return $this->state(['active' => false]);
    }
    
    public function withPosts(int $count = 3): static {
        return $this->has(Post::factory($count));
    }
}

// Usage in Tests/Seeders
User::factory()->create();
User::factory(10)->create();
User::factory()->admin()->create();
User::factory()->withPosts(5)->create();
User::factory(10)->each(fn($user) => $user->posts()->create([...]));
```

---

## 📝 แบบฝึกหัด Part 20

### แบบฝึกหัดที่ 1: Blog System Models
สร้าง Models สำหรับ Blog:
- User, Post, Category, Tag, Comment
- Relationships ทุกประเภท
- Scopes, Observers

### แบบฝึกหัดที่ 2: E-Commerce Models  
สร้าง Models สำหรับ Shop:
- Product, Category, Order, OrderItem, Customer
- Cart functionality
- Stock management

### แบบฝึกหัดที่ 3: Analytics Query
เขียน Complex Queries:
- Top 10 users by post count
- Monthly revenue report
- Category popularity

---

## 🎯 สรุป Part 20

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Model Setup | fillable, casts, hidden, timestamps |
| CRUD | create, find, update, delete |
| Relationships | hasOne/Many, belongsTo, M2M, Polymorphic |
| Eager Loading | with, load, withCount |
| Scopes | Local scopes, Global scopes |
| Collections | pluck, groupBy, map, filter |
| Observers | Model lifecycle events |
| Factories | Test data generation |

**ถัดไป → Part 21: Laravel Middleware**
