# Part 44: Laravel Livewire

## ขั้นตอนที่ 1181-1210: Livewire 3 - Dynamic Components without JavaScript

Livewire ช่วยให้สร้าง dynamic UI ด้วย PHP อย่างเดียว โดยไม่ต้องเขียน JavaScript มาก

---

## ขั้นตอนที่ 1181: Livewire 3 Components พื้นฐาน

```php
<?php

declare(strict_types=1);

namespace App\Livewire;

use App\Models\Post;
use Livewire\Component;
use Livewire\Attributes\Title;
use Livewire\Attributes\Computed;

#[Title('Posts List')]
class PostsList extends Component
{
    public string $search   = '';
    public string $sortBy   = 'created_at';
    public string $sortDir  = 'desc';
    public int $perPage     = 10;

    // Two-way binding ด้วย wire:model
    public function updatedSearch(): void
    {
        $this->resetPage();
    }

    // Computed property - cache ผลลัพธ์
    #[Computed]
    public function posts()
    {
        return Post::query()
            ->when($this->search, fn ($q) => $q->where('title', 'like', "%{$this->search}%"))
            ->orderBy($this->sortBy, $this->sortDir)
            ->paginate($this->perPage);
    }

    public function sortBy(string $column): void
    {
        if ($this->sortBy === $column) {
            $this->sortDir = $this->sortDir === 'asc' ? 'desc' : 'asc';
        } else {
            $this->sortBy  = $column;
            $this->sortDir = 'asc';
        }
    }

    public function deletePost(int $postId): void
    {
        $this->authorize('delete', Post::find($postId));

        Post::destroy($postId);

        $this->dispatch('post-deleted', postId: $postId);
        session()->flash('success', 'Post deleted successfully');
    }

    public function render()
    {
        return view('livewire.posts-list');
    }
}
```

```html
<!-- resources/views/livewire/posts-list.blade.php -->
<div>
    <!-- Flash message -->
    @if (session()->has('success'))
        <div class="alert alert-success" x-data x-init="setTimeout(() => $el.remove(), 3000)">
            {{ session('success') }}
        </div>
    @endif

    <!-- Search -->
    <div class="mb-4">
        <input
            type="text"
            wire:model.live.debounce.300ms="search"
            placeholder="Search posts..."
            class="form-control"
        />
    </div>

    <!-- Loading indicator -->
    <div wire:loading class="spinner"></div>

    <!-- Table -->
    <table class="table" wire:loading.class="opacity-50">
        <thead>
            <tr>
                <th wire:click="sortBy('title')" style="cursor:pointer">
                    Title
                    @if ($sortBy === 'title')
                        {{ $sortDir === 'asc' ? '▲' : '▼' }}
                    @endif
                </th>
                <th wire:click="sortBy('created_at')" style="cursor:pointer">
                    Date
                </th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($this->posts as $post)
                <tr wire:key="post-{{ $post->id }}">
                    <td>{{ $post->title }}</td>
                    <td>{{ $post->created_at->diffForHumans() }}</td>
                    <td>
                        <button
                            wire:click="deletePost({{ $post->id }})"
                            wire:confirm="Are you sure?"
                            class="btn btn-danger btn-sm"
                        >
                            Delete
                        </button>
                    </td>
                </tr>
            @endforeach
        </tbody>
    </table>

    {{ $this->posts->links() }}
</div>
```

---

## ขั้นตอนที่ 1182: Real-time Validation

```php
<?php

declare(strict_types=1);

namespace App\Livewire;

use App\Models\User;
use Livewire\Component;
use Livewire\Attributes\Rule;
use Livewire\Attributes\Validate;

class RegisterForm extends Component
{
    #[Validate('required|min:2|max:50')]
    public string $name = '';

    #[Validate('required|email|unique:users,email')]
    public string $email = '';

    #[Validate('required|min:8|regex:/^(?=.*[A-Z])(?=.*[0-9]).+$/')]
    public string $password = '';

    #[Validate('required|same:password')]
    public string $passwordConfirmation = '';

    public bool $agreeToTerms = false;

    public function register(): void
    {
        $this->validate([
            'agreeToTerms' => 'accepted',
        ]);

        $user = User::create([
            'name'     => $this->name,
            'email'    => $this->email,
            'password' => bcrypt($this->password),
        ]);

        auth()->login($user);

        $this->redirect('/dashboard', navigate: true);
    }

    // Validate แต่ละ field เมื่อ blur
    public function updated(string $property): void
    {
        $this->validateOnly($property);
    }

    public function render()
    {
        return view('livewire.register-form');
    }
}
```

```html
<!-- resources/views/livewire/register-form.blade.php -->
<form wire:submit="register">
    <div class="mb-3">
        <label>Name</label>
        <input
            type="text"
            wire:model.blur="name"
            class="form-control @error('name') is-invalid @enderror"
        />
        @error('name')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>

    <div class="mb-3">
        <label>Email</label>
        <input
            type="email"
            wire:model.blur="email"
            class="form-control @error('email') is-invalid @enderror"
        />
        @error('email')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>

    <div class="mb-3">
        <label>Password</label>
        <input
            type="password"
            wire:model.blur="password"
            class="form-control @error('password') is-invalid @enderror"
        />
        @error('password')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
        <div class="form-text">
            At least 8 characters, 1 uppercase, 1 number
        </div>
    </div>

    <div class="mb-3">
        <input type="checkbox" wire:model="agreeToTerms" id="terms" />
        <label for="terms">I agree to terms</label>
    </div>

    <button type="submit" wire:loading.attr="disabled" class="btn btn-primary">
        <span wire:loading>Registering...</span>
        <span wire:loading.remove>Register</span>
    </button>
</form>
```

---

## ขั้นตอนที่ 1183: Alpine.js Integration

```php
<?php

declare(strict_types=1);

namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\On;

class Modal extends Component
{
    public bool $show = false;
    public string $title = '';
    public string $content = '';

    #[On('open-modal')]
    public function openModal(string $title, string $content): void
    {
        $this->title   = $title;
        $this->content = $content;
        $this->show    = true;
    }

    public function close(): void
    {
        $this->show    = false;
        $this->title   = '';
        $this->content = '';
    }

    public function render()
    {
        return view('livewire.modal');
    }
}
```

```html
<!-- Modal ที่ใช้ Alpine.js ร่วมกัน -->
<div
    x-data="{ show: @entangle('show') }"
    x-show="show"
    x-on:keydown.escape.window="show = false"
    x-transition:enter="transition ease-out duration-300"
    x-transition:enter-start="opacity-0 scale-90"
    x-transition:enter-end="opacity-100 scale-100"
    x-transition:leave="transition ease-in duration-200"
    x-transition:leave-start="opacity-100 scale-100"
    x-transition:leave-end="opacity-0 scale-90"
    class="fixed inset-0 z-50 flex items-center justify-center"
    style="display: none;"
>
    <div class="bg-white rounded-lg shadow-xl p-6 max-w-md w-full mx-4">
        <div class="flex justify-between items-start mb-4">
            <h3 class="text-lg font-semibold">{{ $title }}</h3>
            <button
                x-on:click="show = false"
                class="text-gray-400 hover:text-gray-600"
            >&times;</button>
        </div>

        <div>{{ $content }}</div>

        <div class="mt-4 flex justify-end">
            <button wire:click="close" class="btn btn-secondary">Close</button>
        </div>
    </div>
</div>
```

---

## ขั้นตอนที่ 1184: File Uploads ด้วย Livewire

```php
<?php

declare(strict_types=1);

namespace App\Livewire;

use App\Models\Post;
use Livewire\Component;
use Livewire\Attributes\Validate;
use Livewire\WithFileUploads;

class CreatePost extends Component
{
    use WithFileUploads;

    #[Validate('required|min:5|max:200')]
    public string $title = '';

    #[Validate('required|min:50')]
    public string $body = '';

    #[Validate('nullable|image|max:2048|dimensions:min_width=200,min_height=200')]
    public $thumbnail = null;

    #[Validate('nullable|array|max:5')]
    public array $attachments = [];

    #[Validate('nullable|.*|file|max:5120|mimes:pdf,doc,docx,zip')]
    public $attachment = null;

    public int $uploadProgress = 0;

    public function save(): void
    {
        $this->validate();

        $post = Post::create([
            'user_id' => auth()->id(),
            'title'   => $this->title,
            'body'    => $this->body,
        ]);

        if ($this->thumbnail) {
            $path = $this->thumbnail->store('thumbnails', 's3');
            $post->update(['thumbnail' => $path]);
        }

        foreach ($this->attachments as $attachment) {
            $path = $attachment->store('attachments', 's3');
            $post->attachments()->create([
                'path'           => $path,
                'original_name'  => $attachment->getClientOriginalName(),
                'size'           => $attachment->getSize(),
                'mime_type'      => $attachment->getMimeType(),
            ]);
        }

        session()->flash('success', 'Post created successfully!');
        $this->reset(['title', 'body', 'thumbnail', 'attachments']);
    }

    public function render()
    {
        return view('livewire.create-post');
    }
}
```

```html
<!-- resources/views/livewire/create-post.blade.php -->
<form wire:submit="save" enctype="multipart/form-data">
    <div class="mb-4">
        <input type="text" wire:model="title" placeholder="Post title" class="form-control" />
        @error('title') <span class="text-danger">{{ $message }}</span> @enderror
    </div>

    <div class="mb-4">
        <textarea wire:model="body" rows="10" class="form-control"></textarea>
        @error('body') <span class="text-danger">{{ $message }}</span> @enderror
    </div>

    <!-- Single file upload with preview -->
    <div class="mb-4">
        <label>Thumbnail</label>
        <input type="file" wire:model="thumbnail" accept="image/*" />

        @if ($thumbnail)
            <div class="mt-2">
                <img src="{{ $thumbnail->temporaryUrl() }}" class="img-thumbnail" style="max-height: 200px;" />
            </div>
        @endif

        <div wire:loading wire:target="thumbnail">Uploading thumbnail...</div>
        @error('thumbnail') <span class="text-danger">{{ $message }}</span> @enderror
    </div>

    <!-- Multiple file uploads -->
    <div class="mb-4">
        <label>Attachments (max 5)</label>
        <input type="file" wire:model="attachments" multiple accept=".pdf,.doc,.docx,.zip" />

        @if ($attachments)
            <ul class="mt-2">
                @foreach ($attachments as $attachment)
                    <li>{{ $attachment->getClientOriginalName() }} ({{ round($attachment->getSize() / 1024) }} KB)</li>
                @endforeach
            </ul>
        @endif

        @error('attachments') <span class="text-danger">{{ $message }}</span> @enderror
    </div>

    <button type="submit" wire:loading.attr="disabled" class="btn btn-primary">
        <span wire:loading>Saving...</span>
        <span wire:loading.remove>Save Post</span>
    </button>
</form>
```

---

## ขั้นตอนที่ 1185: Livewire Events และ Lifecycle

```php
<?php

declare(strict_types=1);

namespace App\Livewire;

use Livewire\Component;
use Livewire\Attributes\On;

class ShoppingCart extends Component
{
    public array $items = [];
    public float $total = 0.0;

    // Lifecycle hooks
    public function mount(): void
    {
        // เรียกครั้งแรกเมื่อ component ถูก render
        $this->items = session('cart', []);
        $this->calculateTotal();
    }

    public function hydrate(): void
    {
        // เรียกทุกครั้งที่ request ใหม่ (หลัง mount)
        $this->calculateTotal();
    }

    public function dehydrate(): void
    {
        // เรียกก่อน response ถูกส่ง
        session(['cart' => $this->items]);
    }

    // รับ event จาก component อื่น
    #[On('add-to-cart')]
    public function addItem(int $productId, int $quantity, string $name, float $price): void
    {
        if (isset($this->items[$productId])) {
            $this->items[$productId]['quantity'] += $quantity;
        } else {
            $this->items[$productId] = [
                'product_id' => $productId,
                'name'       => $name,
                'price'      => $price,
                'quantity'   => $quantity,
            ];
        }

        $this->calculateTotal();

        // ส่ง event กลับ
        $this->dispatch('cart-updated', count: count($this->items), total: $this->total);
    }

    public function removeItem(int $productId): void
    {
        unset($this->items[$productId]);
        $this->calculateTotal();
    }

    public function updateQuantity(int $productId, int $quantity): void
    {
        if ($quantity <= 0) {
            $this->removeItem($productId);
            return;
        }

        $this->items[$productId]['quantity'] = $quantity;
        $this->calculateTotal();
    }

    private function calculateTotal(): void
    {
        $this->total = array_sum(
            array_map(
                fn ($item) => $item['price'] * $item['quantity'],
                $this->items
            )
        );
    }

    public function checkout(): void
    {
        if (empty($this->items)) {
            $this->dispatch('show-toast', message: 'Cart is empty!', type: 'warning');
            return;
        }

        $this->dispatch('checkout-started');
        $this->redirect('/checkout');
    }

    public function render()
    {
        return view('livewire.shopping-cart');
    }
}
```

---

## ขั้นตอนที่ 1186: Livewire Polling และ Real-time Updates

```php
<?php

declare(strict_types=1);

namespace App\Livewire;

use App\Models\Notification;
use Livewire\Component;
use Livewire\Attributes\Polling;

class NotificationBell extends Component
{
    public int $count = 0;
    public array $recent = [];

    public function mount(): void
    {
        $this->loadNotifications();
    }

    #[Polling('5s')]  // refresh ทุก 5 วินาที
    public function loadNotifications(): void
    {
        $notifications = Notification::where('user_id', auth()->id())
            ->where('read_at', null)
            ->latest()
            ->take(5)
            ->get();

        $this->count  = $notifications->count();
        $this->recent = $notifications->toArray();
    }

    public function markAllRead(): void
    {
        Notification::where('user_id', auth()->id())
            ->whereNull('read_at')
            ->update(['read_at' => now()]);

        $this->count  = 0;
        $this->recent = [];
    }

    public function render()
    {
        return view('livewire.notification-bell');
    }
}
```

---

## สรุปบทที่ 44

| Feature | วิธีใช้ | ประโยชน์ |
|---------|--------|---------|
| Two-way binding | wire:model | ไม่ต้องเขียน JS |
| Real-time validation | wire:model.blur | UX ดีขึ้น |
| Alpine.js | x-data, @entangle | Animation, local state |
| File uploads | WithFileUploads | Upload พร้อม preview |
| Events | dispatch, #[On] | Component communication |
| Polling | #[Polling] | Real-time data |
| Lifecycle hooks | mount, hydrate | Component initialization |

**ต่อไป**: Part 45 - Database Advanced

---
