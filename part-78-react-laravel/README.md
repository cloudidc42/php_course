# Part 78: React with Laravel API Backend
## ขั้นตอนที่ 2201-2230: React + TypeScript + Laravel

การพัฒนา React application ที่เชื่อมต่อกับ Laravel API
ใช้ TypeScript, React Query, React Hook Form, และ React Router

---

## ขั้นตอนที่ 2201: ตั้งค่า React + TypeScript + Vite

```bash
# สร้าง React app ด้วย Vite
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install

# ติดตั้ง dependencies
npm install @tanstack/react-query axios react-router-dom react-hook-form
npm install @hookform/resolvers zod zustand
npm install -D @types/node
```

---

## ขั้นตอนที่ 2202: TypeScript Types จาก PHP API

```typescript
// src/types/api.ts

export interface User {
  id: number
  name: string
  email: string
  avatar: string | null
  role: 'admin' | 'editor' | 'user'
  bio: string | null
  created_at: string
}

export interface Category {
  id: number
  name: string
  slug: string
}

export interface Post {
  id: number
  title: string
  slug: string
  excerpt: string
  content?: string
  status: 'draft' | 'published'
  featured_image: string | null
  views: number
  reading_time: number
  author: User
  categories: Category[]
  comments_count: number
  likes_count: number
  is_liked?: boolean
  can?: {
    update: boolean
    delete: boolean
  }
  created_at: string
  updated_at: string
}

export interface PaginationMeta {
  current_page: number
  from: number
  last_page: number
  per_page: number
  to: number
  total: number
  links: Array<{
    url: string | null
    label: string
    active: boolean
  }>
}

export interface PaginatedResponse<T> {
  data: T[]
  meta: PaginationMeta
  links: {
    first: string
    last: string
    prev: string | null
    next: string | null
  }
}

export interface ApiResponse<T> {
  data: T
  message?: string
}

export interface ApiError {
  message: string
  errors?: Record<string, string[]>
}

export interface LoginCredentials {
  email: string
  password: string
  remember?: boolean
}

export interface RegisterData {
  name: string
  email: string
  password: string
  password_confirmation: string
}

export interface PostFilters {
  search?: string
  category?: string
  sort?: 'latest' | 'oldest' | 'popular'
  page?: number
  per_page?: number
}
```

---

## ขั้นตอนที่ 2203: API Client

```typescript
// src/lib/api.ts
import axios, { AxiosError, AxiosInstance } from 'axios'
import { ApiError } from '@/types/api'

const createApiClient = (): AxiosInstance => {
  const client = axios.create({
    baseURL: import.meta.env.VITE_API_URL || '/api/v1',
    headers: {
      'Content-Type': 'application/json',
      Accept: 'application/json',
    },
  })

  client.interceptors.request.use((config) => {
    const token = localStorage.getItem('auth_token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  })

  client.interceptors.response.use(
    (response) => response,
    async (error: AxiosError<ApiError>) => {
      if (error.response?.status === 401) {
        localStorage.removeItem('auth_token')
        window.location.href = '/login'
      }
      return Promise.reject(error)
    }
  )

  return client
}

export const api = createApiClient()

// API service functions
export const authApi = {
  login: (credentials: { email: string; password: string }) =>
    api.post<{ token: string; user: import('@/types/api').User }>('/login', credentials),

  register: (data: import('@/types/api').RegisterData) =>
    api.post<{ token: string; user: import('@/types/api').User }>('/register', data),

  logout: () => api.post('/logout'),

  getUser: () => api.get<{ data: import('@/types/api').User }>('/user'),
}

export const postsApi = {
  list: (params?: import('@/types/api').PostFilters) =>
    api.get<import('@/types/api').PaginatedResponse<import('@/types/api').Post>>('/posts', { params }),

  get: (slug: string) =>
    api.get<{ data: import('@/types/api').Post }>(`/posts/${slug}`),

  create: (data: FormData) =>
    api.post<{ data: import('@/types/api').Post }>('/posts', data, {
      headers: { 'Content-Type': 'multipart/form-data' },
    }),

  update: (id: number, data: Partial<import('@/types/api').Post>) =>
    api.put<{ data: import('@/types/api').Post }>(`/posts/${id}`, data),

  delete: (id: number) => api.delete(`/posts/${id}`),

  like: (id: number) =>
    api.post<{ liked: boolean; likes_count: number }>(`/posts/${id}/like`),
}
```

---

## ขั้นตอนที่ 2204: Authentication Context

```typescript
// src/contexts/AuthContext.tsx
import React, { createContext, useContext, useEffect, useState, useCallback } from 'react'
import { User } from '@/types/api'
import { authApi } from '@/lib/api'

interface AuthContextValue {
  user: User | null
  token: string | null
  isAuthenticated: boolean
  isLoading: boolean
  login: (email: string, password: string) => Promise<void>
  register: (name: string, email: string, password: string, confirmation: string) => Promise<void>
  logout: () => Promise<void>
}

const AuthContext = createContext<AuthContextValue | null>(null)

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null)
  const [token, setToken] = useState<string | null>(
    () => localStorage.getItem('auth_token')
  )
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    if (token) {
      authApi.getUser()
        .then(({ data }) => setUser(data.data))
        .catch(() => {
          setToken(null)
          localStorage.removeItem('auth_token')
        })
        .finally(() => setIsLoading(false))
    } else {
      setIsLoading(false)
    }
  }, [token])

  const login = useCallback(async (email: string, password: string) => {
    const { data } = await authApi.login({ email, password })
    setToken(data.token)
    setUser(data.user)
    localStorage.setItem('auth_token', data.token)
  }, [])

  const register = useCallback(async (
    name: string, email: string, password: string, password_confirmation: string
  ) => {
    const { data } = await authApi.register({ name, email, password, password_confirmation })
    setToken(data.token)
    setUser(data.user)
    localStorage.setItem('auth_token', data.token)
  }, [])

  const logout = useCallback(async () => {
    try {
      await authApi.logout()
    } finally {
      setToken(null)
      setUser(null)
      localStorage.removeItem('auth_token')
    }
  }, [])

  return (
    <AuthContext.Provider value={{
      user, token,
      isAuthenticated: !!token && !!user,
      isLoading,
      login, register, logout,
    }}>
      {children}
    </AuthContext.Provider>
  )
}

export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext)
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider')
  }
  return context
}
```

---

## ขั้นตอนที่ 2205: React Query Hooks

```typescript
// src/hooks/usePosts.ts
import { useQuery, useMutation, useQueryClient, useInfiniteQuery } from '@tanstack/react-query'
import { postsApi } from '@/lib/api'
import { PostFilters } from '@/types/api'

export const postKeys = {
  all: ['posts'] as const,
  lists: () => [...postKeys.all, 'list'] as const,
  list: (filters: PostFilters) => [...postKeys.lists(), { filters }] as const,
  details: () => [...postKeys.all, 'detail'] as const,
  detail: (slug: string) => [...postKeys.details(), slug] as const,
}

export function usePosts(filters: PostFilters = {}) {
  return useQuery({
    queryKey: postKeys.list(filters),
    queryFn: () => postsApi.list(filters).then(r => r.data),
    staleTime: 1000 * 60 * 5, // 5 minutes
  })
}

export function usePost(slug: string) {
  return useQuery({
    queryKey: postKeys.detail(slug),
    queryFn: () => postsApi.get(slug).then(r => r.data.data),
    enabled: !!slug,
  })
}

export function useInfinitePosts(filters: Omit<PostFilters, 'page'> = {}) {
  return useInfiniteQuery({
    queryKey: postKeys.list(filters),
    queryFn: ({ pageParam = 1 }) =>
      postsApi.list({ ...filters, page: pageParam as number }).then(r => r.data),
    getNextPageParam: (lastPage) =>
      lastPage.meta.current_page < lastPage.meta.last_page
        ? lastPage.meta.current_page + 1
        : undefined,
    initialPageParam: 1,
  })
}

export function useCreatePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (data: FormData) => postsApi.create(data).then(r => r.data.data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() })
    },
  })
}

export function useUpdatePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<import('@/types/api').Post> }) =>
      postsApi.update(id, data).then(r => r.data.data),
    onSuccess: (post) => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() })
      queryClient.setQueryData(postKeys.detail(post.slug), post)
    },
  })
}

export function useDeletePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: number) => postsApi.delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() })
    },
  })
}

export function useLikePost() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (id: number) => postsApi.like(id).then(r => r.data),
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: postKeys.lists() })
      // Optimistic update
      const snapshot = queryClient.getQueryData(postKeys.lists())
      return { snapshot }
    },
    onError: (_, __, context) => {
      if (context?.snapshot) {
        queryClient.setQueryData(postKeys.lists(), context.snapshot)
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() })
    },
  })
}
```

---

## ขั้นตอนที่ 2206: React Hook Form + Zod

```typescript
// src/components/PostForm.tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { useCreatePost } from '@/hooks/usePosts'
import { useNavigate } from 'react-router-dom'

const postSchema = z.object({
  title: z.string().min(5, 'หัวข้อต้องมีอย่างน้อย 5 ตัวอักษร').max(255),
  content: z.string().min(100, 'เนื้อหาต้องมีอย่างน้อย 100 ตัวอักษร'),
  excerpt: z.string().max(500).optional(),
  status: z.enum(['draft', 'published']),
  categories: z.array(z.number()).min(1, 'เลือกหมวดหมู่อย่างน้อย 1 หมวด'),
  featured_image: z.instanceof(File).optional().nullable(),
})

type PostFormValues = z.infer<typeof postSchema>

interface PostFormProps {
  categories: Array<{ id: number; name: string }>
}

export function PostForm({ categories }: PostFormProps) {
  const navigate = useNavigate()
  const createPost = useCreatePost()

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setValue,
    watch,
  } = useForm<PostFormValues>({
    resolver: zodResolver(postSchema),
    defaultValues: {
      status: 'draft',
      categories: [],
    },
  })

  const onSubmit = async (values: PostFormValues) => {
    const formData = new FormData()
    formData.append('title', values.title)
    formData.append('content', values.content)
    if (values.excerpt) formData.append('excerpt', values.excerpt)
    formData.append('status', values.status)
    values.categories.forEach(id => formData.append('categories[]', String(id)))
    if (values.featured_image) {
      formData.append('featured_image', values.featured_image)
    }

    const post = await createPost.mutateAsync(formData)
    navigate(`/posts/${post.slug}`)
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      <div>
        <label className="block text-sm font-medium text-gray-700">หัวข้อ</label>
        <input
          {...register('title')}
          type="text"
          className="mt-1 block w-full rounded-md border-gray-300 shadow-sm"
          placeholder="หัวข้อบทความ"
        />
        {errors.title && (
          <p className="mt-1 text-sm text-red-600">{errors.title.message}</p>
        )}
      </div>

      <div>
        <label className="block text-sm font-medium text-gray-700">หมวดหมู่</label>
        <div className="mt-2 space-y-2">
          {categories.map(cat => (
            <label key={cat.id} className="inline-flex items-center mr-4">
              <input
                type="checkbox"
                value={cat.id}
                onChange={(e) => {
                  const current = watch('categories')
                  if (e.target.checked) {
                    setValue('categories', [...current, cat.id])
                  } else {
                    setValue('categories', current.filter(id => id !== cat.id))
                  }
                }}
                className="rounded border-gray-300"
              />
              <span className="ml-2 text-sm">{cat.name}</span>
            </label>
          ))}
        </div>
        {errors.categories && (
          <p className="mt-1 text-sm text-red-600">{errors.categories.message}</p>
        )}
      </div>

      <div className="flex gap-4">
        <button
          type="submit"
          disabled={isSubmitting || createPost.isPending}
          className="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:opacity-50"
        >
          {isSubmitting ? 'กำลังบันทึก...' : 'บันทึก'}
        </button>
      </div>
    </form>
  )
}
```

---

## ขั้นตอนที่ 2207: Protected Routes

```typescript
// src/router/index.tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom'
import { useAuth } from '@/contexts/AuthContext'
import DefaultLayout from '@/layouts/DefaultLayout'
import DashboardLayout from '@/layouts/DashboardLayout'
import HomePage from '@/pages/HomePage'
import PostsPage from '@/pages/posts/PostsPage'
import PostShowPage from '@/pages/posts/PostShowPage'
import LoginPage from '@/pages/auth/LoginPage'
import RegisterPage from '@/pages/auth/RegisterPage'
import DashboardPage from '@/pages/dashboard/DashboardPage'
import CreatePostPage from '@/pages/dashboard/CreatePostPage'

function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth()

  if (isLoading) return <div>Loading...</div>
  if (!isAuthenticated) return <Navigate to="/login" replace />
  return <>{children}</>
}

function GuestRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth()

  if (isLoading) return <div>Loading...</div>
  if (isAuthenticated) return <Navigate to="/dashboard" replace />
  return <>{children}</>
}

export function AppRouter() {
  return (
    <BrowserRouter>
      <Routes>
        <Route element={<DefaultLayout />}>
          <Route path="/" element={<HomePage />} />
          <Route path="/posts" element={<PostsPage />} />
          <Route path="/posts/:slug" element={<PostShowPage />} />
        </Route>

        <Route element={<GuestRoute><DefaultLayout /></GuestRoute>}>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/register" element={<RegisterPage />} />
        </Route>

        <Route element={<ProtectedRoute><DashboardLayout /></ProtectedRoute>}>
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/dashboard/posts/create" element={<CreatePostPage />} />
        </Route>
      </Routes>
    </BrowserRouter>
  )
}
```

---

## ขั้นตอนที่ 2208: Laravel CORS สำหรับ React

```php
<?php
declare(strict_types=1);

// config/cors.php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],
    'allowed_methods' => ['*'],
    'allowed_origins' => [
        env('FRONTEND_URL', 'http://localhost:3000'),
    ],
    'allowed_origins_patterns' => [],
    'allowed_headers' => ['*'],
    'exposed_headers' => [],
    'max_age' => 0,
    'supports_credentials' => true,
];
```

```php
<?php
declare(strict_types=1);

// app/Http/Controllers/Api/V1/AuthController.php
namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function register(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'confirmed', Password::defaults()],
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        $token = $user->createToken('auth-token')->plainTextToken;

        return response()->json([
            'token' => $token,
            'user' => $user,
        ], 201);
    }

    public function login(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);

        $user = User::where('email', $validated['email'])->first();

        if (! $user || ! Hash::check($validated['password'], $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['อีเมลหรือรหัสผ่านไม่ถูกต้อง'],
            ]);
        }

        $token = $user->createToken(
            'auth-token',
            ['*'],
            $request->remember ? now()->addMonth() : now()->addDay()
        )->plainTextToken;

        return response()->json([
            'token' => $token,
            'user' => $user,
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out successfully']);
    }
}
```

---

## สรุปบทที่ 78

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|----------|---------|
| TypeScript Types | Interface + Type | Type-safe API calls |
| React Query | useQuery/useMutation | Cache + sync state |
| Auth Context | createContext | Global auth state |
| React Hook Form | register + zodResolver | Form validation |
| Protected Routes | Navigate component | Route guarding |
| Optimistic Updates | onMutate | Better UX |

ถัดไป → Part 79: SPA Authentication Strategies
