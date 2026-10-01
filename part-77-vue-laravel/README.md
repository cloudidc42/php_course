# Part 77: Vue 3 with Laravel Backend
## ขั้นตอนที่ 2171-2200: Vue 3 Composition API + Laravel API

การพัฒนา SPA ด้วย Vue 3 Composition API ที่เชื่อมต่อกับ Laravel REST API
รวมถึง Pinia state management, Vue Router, และ real-time updates

---

## ขั้นตอนที่ 2171: ตั้งค่า Laravel API Backend

```php
<?php
declare(strict_types=1);

// routes/api.php
use App\Http\Controllers\Api\V1\AuthController;
use App\Http\Controllers\Api\V1\PostController;
use App\Http\Controllers\Api\V1\UserController;
use Illuminate\Support\Facades\Route;

Route::prefix('v1')->group(function () {
    // Public routes
    Route::post('/register', [AuthController::class, 'register']);
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/forgot-password', [AuthController::class, 'forgotPassword']);
    Route::post('/reset-password', [AuthController::class, 'resetPassword']);

    Route::get('/posts', [PostController::class, 'index']);
    Route::get('/posts/{post:slug}', [PostController::class, 'show']);

    // Protected routes
    Route::middleware('auth:sanctum')->group(function () {
        Route::post('/logout', [AuthController::class, 'logout']);
        Route::get('/user', [UserController::class, 'profile']);
        Route::put('/user', [UserController::class, 'updateProfile']);

        Route::apiResource('posts', PostController::class)
             ->except(['index', 'show']);
        Route::post('/posts/{post}/like', [PostController::class, 'like']);
        Route::apiResource('comments', \App\Http\Controllers\Api\V1\CommentController::class);
    });
});
```

---

## ขั้นตอนที่ 2172: API Resource Controller

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Http\Resources\PostResource;
use App\Http\Resources\PostCollection;
use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class PostController extends Controller
{
    public function index(Request $request): PostCollection
    {
        $posts = Post::query()
            ->with(['author:id,name,avatar', 'categories:id,name,slug'])
            ->withCount(['comments', 'likes'])
            ->when($request->search, fn ($q, $s) => $q->search($s))
            ->when($request->category, fn ($q, $c) => $q->inCategory($c))
            ->when($request->sort, function ($q, $sort) {
                match ($sort) {
                    'popular' => $q->orderByDesc('likes_count'),
                    'oldest' => $q->oldest(),
                    default => $q->latest(),
                };
            }, fn ($q) => $q->latest())
            ->published()
            ->paginate($request->per_page ?? 15);

        return new PostCollection($posts);
    }

    public function store(StorePostRequest $request): JsonResponse
    {
        $post = $request->user()->posts()->create($request->validated());

        if ($request->hasFile('featured_image')) {
            $post->addMediaFromRequest('featured_image')
                 ->toMediaCollection('featured');
        }

        $post->categories()->sync($request->categories ?? []);
        $post->load(['author:id,name,avatar', 'categories:id,name,slug']);

        return response()->json([
            'message' => 'Post created successfully',
            'data' => new PostResource($post),
        ], Response::HTTP_CREATED);
    }

    public function show(Post $post): PostResource
    {
        abort_if(! $post->isPublished(), 404);

        $post->load([
            'author:id,name,avatar,bio',
            'categories:id,name,slug',
            'comments.user:id,name,avatar',
        ]);

        $post->increment('views');

        return new PostResource($post);
    }

    public function update(UpdatePostRequest $request, Post $post): JsonResponse
    {
        $this->authorize('update', $post);

        $post->update($request->validated());
        $post->categories()->sync($request->categories ?? $post->categories->pluck('id'));

        return response()->json([
            'message' => 'Post updated successfully',
            'data' => new PostResource($post->fresh(['author', 'categories'])),
        ]);
    }

    public function destroy(Post $post): Response
    {
        $this->authorize('delete', $post);
        $post->delete();

        return response()->noContent();
    }

    public function like(Post $post, Request $request): JsonResponse
    {
        $user = $request->user();
        $liked = $post->toggleLike($user);

        return response()->json([
            'liked' => $liked,
            'likes_count' => $post->fresh()->likes_count,
        ]);
    }
}
```

---

## ขั้นตอนที่ 2173: API Resource

```php
<?php
declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'excerpt' => $this->excerpt,
            'content' => $this->when(
                $request->routeIs('api.posts.show'),
                $this->content
            ),
            'status' => $this->status,
            'featured_image' => $this->getFirstMediaUrl('featured'),
            'views' => $this->views,
            'reading_time' => $this->reading_time,
            'author' => new UserResource($this->whenLoaded('author')),
            'categories' => CategoryResource::collection(
                $this->whenLoaded('categories')
            ),
            'comments_count' => $this->whenCounted('comments'),
            'likes_count' => $this->whenCounted('likes'),
            'is_liked' => $this->when(
                $request->user(),
                fn () => $this->isLikedBy($request->user())
            ),
            'can' => [
                'update' => $request->user()?->can('update', $this->resource),
                'delete' => $request->user()?->can('delete', $this->resource),
            ],
            'created_at' => $this->created_at?->toISOString(),
            'updated_at' => $this->updated_at?->toISOString(),
        ];
    }
}
```

---

## ขั้นตอนที่ 2174: Axios Setup กับ Interceptors

```javascript
// src/lib/axios.js
import axios from 'axios'
import { useAuthStore } from '@/stores/auth'
import router from '@/router'

const api = axios.create({
    baseURL: import.meta.env.VITE_API_URL || '/api/v1',
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    },
    withCredentials: true,
})

// Request interceptor - เพิ่ม auth token
api.interceptors.request.use(
    (config) => {
        const authStore = useAuthStore()
        if (authStore.token) {
            config.headers.Authorization = `Bearer ${authStore.token}`
        }
        return config
    },
    (error) => Promise.reject(error)
)

// Response interceptor - จัดการ errors
api.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config

        // ถ้า token หมดอายุ ให้ refresh
        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true
            const authStore = useAuthStore()

            try {
                await authStore.refreshToken()
                originalRequest.headers.Authorization = `Bearer ${authStore.token}`
                return api(originalRequest)
            } catch {
                authStore.logout()
                router.push('/login')
                return Promise.reject(error)
            }
        }

        // จัดการ validation errors
        if (error.response?.status === 422) {
            return Promise.reject({
                ...error,
                validationErrors: error.response.data.errors,
            })
        }

        // จัดการ server errors
        if (error.response?.status >= 500) {
            console.error('Server error:', error.response.data)
        }

        return Promise.reject(error)
    }
)

export default api
```

---

## ขั้นตอนที่ 2175: Pinia Store

```javascript
// src/stores/auth.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import api from '@/lib/axios'

export const useAuthStore = defineStore('auth', () => {
    const user = ref(null)
    const token = ref(localStorage.getItem('token'))
    const loading = ref(false)

    const isAuthenticated = computed(() => !!token.value && !!user.value)
    const isAdmin = computed(() => user.value?.role === 'admin')

    const login = async (credentials) => {
        loading.value = true
        try {
            const { data } = await api.post('/login', credentials)
            token.value = data.token
            user.value = data.user
            localStorage.setItem('token', data.token)
            return data
        } finally {
            loading.value = false
        }
    }

    const register = async (userData) => {
        loading.value = true
        try {
            const { data } = await api.post('/register', userData)
            token.value = data.token
            user.value = data.user
            localStorage.setItem('token', data.token)
            return data
        } finally {
            loading.value = false
        }
    }

    const logout = async () => {
        try {
            await api.post('/logout')
        } finally {
            token.value = null
            user.value = null
            localStorage.removeItem('token')
        }
    }

    const fetchUser = async () => {
        if (!token.value) return
        try {
            const { data } = await api.get('/user')
            user.value = data.data
        } catch {
            token.value = null
            localStorage.removeItem('token')
        }
    }

    const refreshToken = async () => {
        const { data } = await api.post('/auth/refresh')
        token.value = data.token
        localStorage.setItem('token', data.token)
    }

    return {
        user, token, loading,
        isAuthenticated, isAdmin,
        login, register, logout, fetchUser, refreshToken,
    }
}, {
    persist: {
        paths: ['token'],
    },
})
```

```javascript
// src/stores/posts.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import api from '@/lib/axios'

export const usePostStore = defineStore('posts', () => {
    const posts = ref([])
    const currentPost = ref(null)
    const pagination = ref(null)
    const loading = ref(false)
    const error = ref(null)

    const fetchPosts = async (params = {}) => {
        loading.value = true
        error.value = null
        try {
            const { data } = await api.get('/posts', { params })
            posts.value = data.data
            pagination.value = data.meta
        } catch (e) {
            error.value = e.message
        } finally {
            loading.value = false
        }
    }

    const fetchPost = async (slug) => {
        loading.value = true
        try {
            const { data } = await api.get(`/posts/${slug}`)
            currentPost.value = data.data
            return data.data
        } finally {
            loading.value = false
        }
    }

    const createPost = async (postData) => {
        const formData = new FormData()
        Object.entries(postData).forEach(([key, value]) => {
            if (Array.isArray(value)) {
                value.forEach(v => formData.append(`${key}[]`, v))
            } else if (value !== null && value !== undefined) {
                formData.append(key, value)
            }
        })

        const { data } = await api.post('/posts', formData, {
            headers: { 'Content-Type': 'multipart/form-data' },
        })
        posts.value.unshift(data.data)
        return data.data
    }

    const updatePost = async (id, postData) => {
        const { data } = await api.put(`/posts/${id}`, postData)
        const index = posts.value.findIndex(p => p.id === id)
        if (index !== -1) posts.value[index] = data.data
        return data.data
    }

    const deletePost = async (id) => {
        await api.delete(`/posts/${id}`)
        posts.value = posts.value.filter(p => p.id !== id)
    }

    const likePost = async (id) => {
        const { data } = await api.post(`/posts/${id}/like`)
        const post = posts.value.find(p => p.id === id)
        if (post) {
            post.is_liked = data.liked
            post.likes_count = data.likes_count
        }
        if (currentPost.value?.id === id) {
            currentPost.value.is_liked = data.liked
            currentPost.value.likes_count = data.likes_count
        }
    }

    return {
        posts, currentPost, pagination, loading, error,
        fetchPosts, fetchPost, createPost, updatePost, deletePost, likePost,
    }
})
```

---

## ขั้นตอนที่ 2176: Vue Router

```javascript
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const routes = [
    {
        path: '/',
        component: () => import('@/layouts/DefaultLayout.vue'),
        children: [
            { path: '', name: 'home', component: () => import('@/pages/HomePage.vue') },
            { path: 'posts', name: 'posts.index', component: () => import('@/pages/posts/PostsPage.vue') },
            { path: 'posts/:slug', name: 'posts.show', component: () => import('@/pages/posts/PostShowPage.vue') },
        ],
    },
    {
        path: '/auth',
        component: () => import('@/layouts/AuthLayout.vue'),
        meta: { requiresGuest: true },
        children: [
            { path: 'login', name: 'login', component: () => import('@/pages/auth/LoginPage.vue') },
            { path: 'register', name: 'register', component: () => import('@/pages/auth/RegisterPage.vue') },
        ],
    },
    {
        path: '/dashboard',
        component: () => import('@/layouts/DashboardLayout.vue'),
        meta: { requiresAuth: true },
        children: [
            { path: '', name: 'dashboard', component: () => import('@/pages/dashboard/DashboardPage.vue') },
            { path: 'posts/create', name: 'posts.create', component: () => import('@/pages/dashboard/CreatePostPage.vue') },
            { path: 'posts/:id/edit', name: 'posts.edit', component: () => import('@/pages/dashboard/EditPostPage.vue') },
        ],
    },
]

const router = createRouter({
    history: createWebHistory(import.meta.env.BASE_URL),
    routes,
    scrollBehavior(to, from, savedPosition) {
        if (savedPosition) return savedPosition
        return { top: 0 }
    },
})

router.beforeEach(async (to) => {
    const authStore = useAuthStore()

    if (authStore.token && !authStore.user) {
        await authStore.fetchUser()
    }

    if (to.meta.requiresAuth && !authStore.isAuthenticated) {
        return { name: 'login', query: { redirect: to.fullPath } }
    }

    if (to.meta.requiresGuest && authStore.isAuthenticated) {
        return { name: 'dashboard' }
    }
})

export default router
```

---

## ขั้นตอนที่ 2177: Real-time กับ Laravel Echo

```php
<?php
declare(strict_types=1);

// app/Events/PostCreated.php
namespace App\Events;

use App\Models\Post;
use App\Http\Resources\PostResource;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class PostCreated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Post $post
    ) {}

    public function broadcastOn(): array
    {
        return [
            new Channel('posts'),
        ];
    }

    public function broadcastWith(): array
    {
        return [
            'post' => new PostResource($this->post->load(['author:id,name,avatar', 'categories:id,name'])),
        ];
    }

    public function broadcastAs(): string
    {
        return 'post.created';
    }
}
```

```javascript
// src/composables/useEcho.js
import { onMounted, onUnmounted } from 'vue'
import Echo from 'laravel-echo'
import Pusher from 'pusher-js'

window.Pusher = Pusher

const echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    forceTLS: true,
    authEndpoint: '/api/broadcasting/auth',
})

export function useEcho() {
    return { echo }
}

export function usePostsChannel(onNewPost) {
    let channel

    onMounted(() => {
        channel = echo.channel('posts')
        channel.listen('.post.created', (e) => {
            onNewPost(e.post)
        })
    })

    onUnmounted(() => {
        echo.leaveChannel('posts')
    })
}

export function usePrivateChannel(userId, callbacks = {}) {
    let channel

    onMounted(() => {
        channel = echo.private(`user.${userId}`)

        if (callbacks.onNotification) {
            channel.notification(callbacks.onNotification)
        }
    })

    onUnmounted(() => {
        echo.leaveChannel(`private-user.${userId}`)
    })
}
```

---

## ขั้นตอนที่ 2178: Composables

```javascript
// src/composables/usePosts.js
import { ref, computed } from 'vue'
import { usePostStore } from '@/stores/posts'
import { storeToRefs } from 'pinia'

export function usePosts() {
    const store = usePostStore()
    const { posts, loading, pagination, error } = storeToRefs(store)
    const filters = ref({ search: '', category: '', sort: 'latest', page: 1 })

    const totalPages = computed(() => pagination.value?.last_page ?? 1)

    const search = (query) => {
        filters.value.search = query
        filters.value.page = 1
        store.fetchPosts(filters.value)
    }

    const changePage = (page) => {
        filters.value.page = page
        store.fetchPosts(filters.value)
        window.scrollTo({ top: 0, behavior: 'smooth' })
    }

    return {
        posts, loading, pagination, error, filters, totalPages,
        fetchPosts: () => store.fetchPosts(filters.value),
        search, changePage,
    }
}

// src/composables/useForm.js
import { ref, reactive } from 'vue'

export function useForm(initialValues = {}) {
    const values = reactive({ ...initialValues })
    const errors = ref({})
    const submitting = ref(false)

    const setErrors = (newErrors) => {
        errors.value = newErrors
    }

    const clearError = (field) => {
        delete errors.value[field]
    }

    const reset = () => {
        Object.assign(values, initialValues)
        errors.value = {}
    }

    const handleSubmit = (submitFn) => async (event) => {
        event?.preventDefault()
        submitting.value = true
        errors.value = {}
        try {
            await submitFn(values)
        } catch (e) {
            if (e.validationErrors) {
                errors.value = e.validationErrors
            }
        } finally {
            submitting.value = false
        }
    }

    return { values, errors, submitting, setErrors, clearError, reset, handleSubmit }
}
```

---

## สรุปบทที่ 77

| หัวข้อ | เทคโนโลยี | ประโยชน์ |
|--------|----------|---------|
| API Routes | Laravel Sanctum | Auth token |
| Axios Interceptors | axios | Auto refresh token |
| Pinia Store | defineStore + composables | State management |
| Vue Router | Guards + lazy loading | Navigation control |
| Laravel Echo | Pusher/Soketi | Real-time updates |
| Composables | useForm, usePosts | Code reuse |

ถัดไป → Part 78: React with Laravel API Backend
