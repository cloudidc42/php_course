# Part 76: Laravel Inertia.js
## ขั้นตอนที่ 2141-2170: Server-side Routing with Client-side Rendering

Inertia.js เป็น "glue" ที่เชื่อม Laravel กับ Vue 3/React โดยไม่ต้องสร้าง API แยก
ช่วยให้เขียน SPA ด้วย Laravel routing และ Eloquent ได้โดยตรง

---

## ขั้นตอนที่ 2141: การติดตั้ง Inertia.js

```bash
# ติดตั้ง server-side adapter
composer require inertiajs/inertia-laravel

# ติดตั้ง client-side adapter
npm install @inertiajs/vue3 vue@3

# สร้าง middleware
php artisan inertia:middleware
```

เพิ่ม middleware ใน `bootstrap/app.php`:

```php
<?php
declare(strict_types=1);

use App\Http\Middleware\HandleInertiaRequests;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            HandleInertiaRequests::class,
        ]);
    })
    ->create();
```

---

## ขั้นตอนที่ 2142: Inertia Middleware

```php
<?php
declare(strict_types=1);

namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    /**
     * The root template that's loaded on the first page visit.
     */
    protected $rootView = 'app';

    /**
     * Determines the current asset version.
     */
    public function version(Request $request): ?string
    {
        return parent::version($request);
    }

    /**
     * Defines the props that are shared by default.
     * ข้อมูลที่แชร์ให้ทุกหน้า
     */
    public function share(Request $request): array
    {
        return array_merge(parent::share($request), [
            'auth' => [
                'user' => $request->user() ? [
                    'id' => $request->user()->id,
                    'name' => $request->user()->name,
                    'email' => $request->user()->email,
                    'avatar' => $request->user()->avatar_url,
                ] : null,
            ],
            'flash' => [
                'success' => $request->session()->get('success'),
                'error' => $request->session()->get('error'),
                'warning' => $request->session()->get('warning'),
            ],
            'permissions' => $request->user()
                ? $request->user()->getAllPermissions()->pluck('name')
                : [],
        ]);
    }
}
```

---

## ขั้นตอนที่ 2143: Root Template (Blade)

สร้าง `resources/views/app.blade.php`:

```html
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ config('app.name', 'Laravel') }}</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
    @inertiaHead
</head>
<body class="font-sans antialiased">
    @inertia
</body>
</html>
```

---

## ขั้นตอนที่ 2144: Vue 3 App Entry Point

สร้าง `resources/js/app.js`:

```javascript
import { createApp, h } from 'vue'
import { createInertiaApp } from '@inertiajs/vue3'
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers'
import '../css/app.css'

createInertiaApp({
    title: (title) => `${title} - My App`,
    resolve: (name) => resolvePageComponent(
        `./Pages/${name}.vue`,
        import.meta.glob('./Pages/**/*.vue')
    ),
    setup({ el, App, props, plugin }) {
        return createApp({ render: () => h(App, props) })
            .use(plugin)
            .mount(el)
    },
    progress: {
        color: '#4B5563',
        showSpinner: true,
    },
})
```

---

## ขั้นตอนที่ 2145: Controller พื้นฐาน

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\User;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class PostController extends Controller
{
    /**
     * แสดงรายการโพสต์
     */
    public function index(Request $request): Response
    {
        $posts = Post::query()
            ->with(['author:id,name,avatar'])
            ->withCount('comments')
            ->when($request->search, function ($query, $search) {
                $query->where('title', 'like', "%{$search}%")
                      ->orWhere('content', 'like', "%{$search}%");
            })
            ->when($request->category, function ($query, $category) {
                $query->whereHas('categories', fn ($q) => $q->where('slug', $category));
            })
            ->latest()
            ->paginate(15)
            ->withQueryString();

        return Inertia::render('Posts/Index', [
            'posts' => $posts,
            'filters' => $request->only(['search', 'category']),
            'categories' => \App\Models\Category::select('id', 'name', 'slug')->get(),
        ]);
    }

    /**
     * แสดงโพสต์เดียว
     */
    public function show(Post $post): Response
    {
        $post->load([
            'author:id,name,avatar,bio',
            'categories:id,name,slug',
            'comments' => function ($query) {
                $query->with('user:id,name,avatar')
                      ->latest()
                      ->limit(20);
            },
        ]);

        return Inertia::render('Posts/Show', [
            'post' => $post,
            'relatedPosts' => Post::relatedTo($post)->limit(4)->get(),
        ]);
    }

    /**
     * บันทึกโพสต์ใหม่
     */
    public function store(Request $request): \Illuminate\Http\RedirectResponse
    {
        $validated = $request->validate([
            'title' => ['required', 'string', 'max:255'],
            'content' => ['required', 'string', 'min:100'],
            'excerpt' => ['nullable', 'string', 'max:500'],
            'categories' => ['required', 'array', 'min:1'],
            'categories.*' => ['exists:categories,id'],
            'status' => ['required', 'in:draft,published'],
            'featured_image' => ['nullable', 'image', 'max:2048'],
        ]);

        $post = $request->user()->posts()->create([
            'title' => $validated['title'],
            'slug' => \Str::slug($validated['title']),
            'content' => $validated['content'],
            'excerpt' => $validated['excerpt'] ?? \Str::limit($validated['content'], 200),
            'status' => $validated['status'],
        ]);

        if ($request->hasFile('featured_image')) {
            $post->addMediaFromRequest('featured_image')
                 ->toMediaCollection('featured');
        }

        $post->categories()->sync($validated['categories']);

        return redirect()->route('posts.show', $post)
            ->with('success', 'โพสต์ถูกสร้างเรียบร้อยแล้ว');
    }
}
```

---

## ขั้นตอนที่ 2146: Vue Page Component

สร้าง `resources/js/Pages/Posts/Index.vue`:

```vue
<template>
  <AppLayout title="โพสต์ทั้งหมด">
    <!-- Filters -->
    <div class="mb-6 flex gap-4">
      <TextInput
        v-model="filters.search"
        placeholder="ค้นหาโพสต์..."
        class="w-64"
        @input="debouncedSearch"
      />
      <SelectInput v-model="filters.category" @change="applyFilters">
        <option value="">ทุกหมวดหมู่</option>
        <option
          v-for="cat in categories"
          :key="cat.id"
          :value="cat.slug"
        >{{ cat.name }}</option>
      </SelectInput>
    </div>

    <!-- Post Grid -->
    <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
      <PostCard
        v-for="post in posts.data"
        :key="post.id"
        :post="post"
      />
    </div>

    <!-- Pagination -->
    <Pagination :links="posts.links" class="mt-8" />
  </AppLayout>
</template>

<script setup>
import { ref, watch } from 'vue'
import { router } from '@inertiajs/vue3'
import { debounce } from 'lodash'
import AppLayout from '@/Layouts/AppLayout.vue'
import PostCard from '@/Components/PostCard.vue'
import Pagination from '@/Components/Pagination.vue'
import TextInput from '@/Components/TextInput.vue'
import SelectInput from '@/Components/SelectInput.vue'

const props = defineProps({
  posts: Object,
  filters: Object,
  categories: Array,
})

const filters = ref({ ...props.filters })

const applyFilters = () => {
  router.get(route('posts.index'), filters.value, {
    preserveState: true,
    replace: true,
  })
}

const debouncedSearch = debounce(applyFilters, 300)
</script>
```

---

## ขั้นตอนที่ 2147: Form Handling กับ Inertia

```vue
<template>
  <AppLayout title="สร้างโพสต์">
    <form @submit.prevent="submit" class="max-w-3xl mx-auto space-y-6">
      <!-- Title -->
      <div>
        <InputLabel for="title" value="หัวข้อ" />
        <TextInput
          id="title"
          v-model="form.title"
          type="text"
          class="mt-1 block w-full"
          autofocus
        />
        <InputError :message="form.errors.title" class="mt-2" />
      </div>

      <!-- Content -->
      <div>
        <InputLabel for="content" value="เนื้อหา" />
        <RichTextEditor
          id="content"
          v-model="form.content"
          class="mt-1"
        />
        <InputError :message="form.errors.content" class="mt-2" />
      </div>

      <!-- Categories -->
      <div>
        <InputLabel value="หมวดหมู่" />
        <div class="mt-2 flex flex-wrap gap-2">
          <label
            v-for="cat in categories"
            :key="cat.id"
            class="inline-flex items-center gap-2 cursor-pointer"
          >
            <input
              type="checkbox"
              :value="cat.id"
              v-model="form.categories"
              class="rounded border-gray-300"
            />
            <span>{{ cat.name }}</span>
          </label>
        </div>
        <InputError :message="form.errors.categories" class="mt-2" />
      </div>

      <!-- Featured Image -->
      <div>
        <InputLabel for="image" value="รูปภาพหลัก" />
        <input
          id="image"
          type="file"
          accept="image/*"
          @change="handleImageChange"
          class="mt-1"
        />
        <img
          v-if="imagePreview"
          :src="imagePreview"
          class="mt-2 h-40 rounded object-cover"
        />
      </div>

      <!-- Status -->
      <div class="flex items-center gap-4">
        <PrimaryButton
          type="submit"
          :disabled="form.processing"
        >
          {{ form.status === 'published' ? 'เผยแพร่' : 'บันทึกร่าง' }}
        </PrimaryButton>

        <button
          type="button"
          @click="form.status = form.status === 'draft' ? 'published' : 'draft'"
          class="text-sm text-gray-600"
        >
          {{ form.status === 'draft' ? 'เผยแพร่ทันที' : 'บันทึกเป็นร่าง' }}
        </button>

        <span v-if="form.processing" class="text-sm text-gray-500">
          กำลังบันทึก...
        </span>
      </div>
    </form>
  </AppLayout>
</template>

<script setup>
import { ref } from 'vue'
import { useForm } from '@inertiajs/vue3'
import AppLayout from '@/Layouts/AppLayout.vue'

const props = defineProps({
  categories: Array,
})

const form = useForm({
  title: '',
  content: '',
  excerpt: '',
  categories: [],
  status: 'draft',
  featured_image: null,
})

const imagePreview = ref(null)

const handleImageChange = (e) => {
  const file = e.target.files[0]
  if (!file) return

  form.featured_image = file
  const reader = new FileReader()
  reader.onload = (e) => { imagePreview.value = e.target.result }
  reader.readAsDataURL(file)
}

const submit = () => {
  form.post(route('posts.store'), {
    forceFormData: true,
    onSuccess: () => form.reset(),
  })
}
</script>
```

---

## ขั้นตอนที่ 2148: Authentication Flow

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Auth\Events\Registered;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules;
use Inertia\Inertia;
use Inertia\Response;

class RegisteredUserController extends Controller
{
    public function create(): Response
    {
        return Inertia::render('Auth/Register');
    }

    public function store(Request $request): RedirectResponse
    {
        $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'lowercase', 'email', 'max:255', 'unique:'.User::class],
            'password' => ['required', 'confirmed', Rules\Password::defaults()],
        ]);

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        event(new Registered($user));

        Auth::login($user);

        return redirect(route('dashboard', absolute: false));
    }
}
```

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Route;
use Inertia\Inertia;
use Inertia\Response;

class AuthenticatedSessionController extends Controller
{
    public function create(): Response
    {
        return Inertia::render('Auth/Login', [
            'canResetPassword' => Route::has('password.request'),
            'status' => session('status'),
        ]);
    }

    public function store(LoginRequest $request): RedirectResponse
    {
        $request->authenticate();
        $request->session()->regenerate();

        return redirect()->intended(route('dashboard', absolute: false));
    }

    public function destroy(Request $request): RedirectResponse
    {
        Auth::guard('web')->logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

---

## ขั้นตอนที่ 2149: Infinite Scroll

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class FeedController extends Controller
{
    public function index(): Response
    {
        $posts = Post::with('author:id,name,avatar')
            ->published()
            ->latest()
            ->cursorPaginate(10);

        return Inertia::render('Feed/Index', [
            'initialPosts' => $posts,
        ]);
    }

    public function loadMore(Request $request): JsonResponse
    {
        $posts = Post::with('author:id,name,avatar')
            ->published()
            ->latest()
            ->cursorPaginate(10, ['*'], 'cursor', $request->cursor);

        return response()->json($posts);
    }
}
```

```vue
<template>
  <AppLayout title="ฟีด">
    <div class="max-w-2xl mx-auto space-y-6">
      <PostCard
        v-for="post in allPosts"
        :key="post.id"
        :post="post"
      />

      <!-- Infinite Scroll Trigger -->
      <div ref="loadMoreTrigger" class="h-10 flex items-center justify-center">
        <LoadingSpinner v-if="loading" />
        <p v-else-if="!hasMore" class="text-gray-400 text-sm">
          โหลดครบทุกโพสต์แล้ว
        </p>
      </div>
    </div>
  </AppLayout>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import axios from 'axios'
import AppLayout from '@/Layouts/AppLayout.vue'
import PostCard from '@/Components/PostCard.vue'

const props = defineProps({
  initialPosts: Object,
})

const posts = ref(props.initialPosts.data)
const nextCursor = ref(props.initialPosts.next_cursor)
const loading = ref(false)
const loadMoreTrigger = ref(null)
const hasMore = computed(() => !!nextCursor.value)

const loadMore = async () => {
  if (loading.value || !hasMore.value) return

  loading.value = true
  try {
    const { data } = await axios.get(route('feed.load-more'), {
      params: { cursor: nextCursor.value }
    })
    posts.value.push(...data.data)
    nextCursor.value = data.next_cursor
  } finally {
    loading.value = false
  }
}

const allPosts = computed(() => posts.value)

let observer
onMounted(() => {
  observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) loadMore()
  }, { threshold: 0.5 })

  if (loadMoreTrigger.value) {
    observer.observe(loadMoreTrigger.value)
  }
})

onUnmounted(() => observer?.disconnect())
</script>
```

---

## ขั้นตอนที่ 2150: Shared Data และ Server-side Rendering

```php
<?php
declare(strict_types=1);

// routes/web.php
use App\Http\Controllers\PostController;
use App\Http\Controllers\DashboardController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return inertia('Welcome', [
        'featuredPosts' => \App\Models\Post::featured()->limit(6)->get(),
    ]);
})->name('home');

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::resource('posts', PostController::class);
    Route::resource('comments', \App\Http\Controllers\CommentController::class)
         ->only(['store', 'update', 'destroy']);
});

Route::middleware('auth')->group(function () {
    Route::get('/profile', [\App\Http\Controllers\ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [\App\Http\Controllers\ProfileController::class, 'update'])->name('profile.update');
    Route::delete('/profile', [\App\Http\Controllers\ProfileController::class, 'destroy'])->name('profile.destroy');
});
```

---

## สรุปบทที่ 76

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| Inertia Middleware | HandleInertiaRequests | แชร์ข้อมูล global |
| Page Component | Vue SFC | Render ฝั่ง client |
| useForm() | Inertia Helper | จัดการ form + errors |
| Cursor Pagination | Laravel + Axios | Infinite scroll |
| Shared Props | middleware::share() | Auth, flash messages |
| SSR Support | @inertiajs/vue3 | SEO friendly |

ถัดไป → Part 77: Vue 3 Composition API with Laravel Backend
