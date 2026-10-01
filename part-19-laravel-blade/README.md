# Part 19: Laravel Blade Templates
## ขั้นตอนที่ 491-510: Blade Template Engine

---

## ขั้นตอนที่ 491: Blade Basics

```blade
{{-- resources/views/welcome.blade.php --}}

{{-- Echo (auto-escaped) --}}
{{ $name }}
{{ $user->email }}
{{ now()->format('d/m/Y') }}

{{-- Unescaped (ระวัง XSS!) --}}
{!! $html !!}

{{-- Default value --}}
{{ $name ?? 'Guest' }}
{{ $user->name ?? 'Unknown' }}

{{-- Comments (ไม่ render ใน HTML) --}}
{{-- This is a comment --}}

{{-- PHP Code --}}
@php
    $greeting = match(true) {
        now()->hour < 12 => 'สวัสดีตอนเช้า',
        now()->hour < 17 => 'สวัสดีตอนบ่าย',
        default => 'สวัสดีตอนเย็น'
    };
@endphp

<h1>{{ $greeting }}, {{ $name }}!</h1>
```

---

## ขั้นตอนที่ 492: Control Structures

```blade
{{-- If/Else --}}
@if ($user->isAdmin())
    <div class="admin-badge">Admin</div>
@elseif ($user->isModerator())
    <div class="mod-badge">Moderator</div>
@else
    <div class="user-badge">User</div>
@endif

{{-- Unless --}}
@unless ($user->isVerified())
    <div class="alert">Please verify your email!</div>
@endunless

{{-- Isset / Empty --}}
@isset($variable)
    <p>{{ $variable }}</p>
@endisset

@empty($records)
    <p>No records found</p>
@endempty

{{-- Auth --}}
@auth
    Welcome, {{ auth()->user()->name }}!
    <a href="{{ route('logout') }}">Logout</a>
@endauth

@guest
    <a href="{{ route('login') }}">Login</a>
@endguest

@auth('api')
    {{-- For specific guard --}}
@endauth

{{-- Role / Can (with Gates/Policies) --}}
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@cannot('delete', $post)
    <p>ไม่มีสิทธิ์ลบ</p>
@endcannot

@canany(['update', 'delete'], $post)
    <div class="actions">...</div>
@endcanany

{{-- Environment --}}
@production
    {{-- Only in production --}}
    <script>/* analytics code */</script>
@endproduction

@env(['local', 'testing'])
    <div class="debug-bar">...</div>
@endenv

{{-- Switch --}}
@switch($user->role)
    @case('admin')
        <span class="badge-red">Admin</span>
        @break
    @case('editor')
        <span class="badge-blue">Editor</span>
        @break
    @default
        <span class="badge-gray">User</span>
@endswitch
```

---

## ขั้นตอนที่ 493: Loops

```blade
{{-- Foreach --}}
@foreach ($users as $user)
    <div class="user-card">
        <h3>{{ $user->name }}</h3>
        <p>{{ $user->email }}</p>
    </div>
@endforeach

{{-- Foreach with $loop variable --}}
@foreach ($items as $item)
    <div class="item 
        @if ($loop->first) first-item @endif
        @if ($loop->last) last-item @endif
        @if ($loop->odd) odd @else even @endif
    ">
        {{-- $loop->index      (0-based) --}}
        {{-- $loop->iteration  (1-based) --}}
        {{-- $loop->remaining  --}}
        {{-- $loop->count      --}}
        {{-- $loop->first      --}}
        {{-- $loop->last       --}}
        {{-- $loop->even/odd   --}}
        {{-- $loop->depth      (nested loops) --}}
        {{-- $loop->parent     (parent loop) --}}
        
        {{ $loop->iteration }}. {{ $item->name }}
        
        @if ($loop->iteration % 3 === 0)
            </div><div class="row"> {{-- Column break every 3 --}}
        @endif
    </div>
@endforeach

{{-- Forelse (empty fallback) --}}
@forelse ($posts as $post)
    <article>{{ $post->title }}</article>
@empty
    <p>ยังไม่มีบทความ</p>
@endforelse

{{-- For --}}
@for ($i = 1; $i <= 10; $i++)
    <span>{{ $i }}</span>
@endfor

{{-- While --}}
@while ($condition)
    ...
@endwhile

{{-- Continue / Break --}}
@foreach ($users as $user)
    @continue($user->isBanned())
    
    @if ($user->isAdmin())
        <div class="admin">{{ $user->name }}</div>
        @continue
    @endif
    
    {{ $user->name }}
    
    @break($loop->iteration >= 10)
@endforeach
```

---

## ขั้นตอนที่ 494: Layouts & Components

### Layout (Blade Inheritance)
```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@yield('title', 'My App') - {{ config('app.name') }}</title>
    
    @stack('styles')
    <link rel="stylesheet" href="{{ asset('css/app.css') }}">
</head>
<body class="@yield('body-class', 'bg-gray-100')">
    <nav>
        @include('partials.navbar')
    </nav>
    
    <main>
        {{-- Flash Messages --}}
        @if (session('success'))
            <div class="alert alert-success">{{ session('success') }}</div>
        @endif
        
        @if (session('error'))
            <div class="alert alert-danger">{{ session('error') }}</div>
        @endif
        
        @yield('content')
    </main>
    
    <footer>
        @include('partials.footer')
    </footer>
    
    <script src="{{ asset('js/app.js') }}"></script>
    @stack('scripts')
</body>
</html>
```

```blade
{{-- resources/views/users/index.blade.php --}}
@extends('layouts.app')

@section('title', 'รายชื่อผู้ใช้')

@section('body-class', 'users-page bg-white')

@section('content')
    <div class="container">
        <h1>Users</h1>
        
        @foreach ($users as $user)
            <div class="user">{{ $user->name }}</div>
        @endforeach
        
        {{ $users->links() }}
    </div>
@endsection

@push('scripts')
    <script>
        console.log('Users page loaded');
    </script>
@endpush
```

---

## ขั้นตอนที่ 495: Blade Components

### Class Component
```php
<?php
// app/View/Components/Alert.php
// สร้าง: php artisan make:component Alert

namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\View\View;

class Alert extends Component {
    public function __construct(
        public string $type = 'info',
        public ?string $title = null,
        public bool $dismissible = false,
    ) {}
    
    public function alertClass(): string {
        return match($this->type) {
            'success' => 'bg-green-100 border-green-400 text-green-700',
            'error', 'danger' => 'bg-red-100 border-red-400 text-red-700',
            'warning' => 'bg-yellow-100 border-yellow-400 text-yellow-700',
            default => 'bg-blue-100 border-blue-400 text-blue-700',
        };
    }
    
    public function render(): View {
        return view('components.alert');
    }
}
```

```blade
{{-- resources/views/components/alert.blade.php --}}
<div class="border px-4 py-3 rounded {{ $alertClass() }}" role="alert">
    @if ($title)
        <strong class="font-bold">{{ $title }}</strong>
    @endif
    
    <span class="block">{{ $slot }}</span>
    
    @if ($dismissible)
        <button type="button" class="close" data-dismiss="alert">
            <span>&times;</span>
        </button>
    @endif
</div>
```

```blade
{{-- Usage --}}
<x-alert type="success" title="สำเร็จ!">
    บันทึกข้อมูลเรียบร้อยแล้ว
</x-alert>

<x-alert type="error" :dismissible="true">
    เกิดข้อผิดพลาด กรุณาลองใหม่
</x-alert>
```

### Anonymous Component (ไม่ต้องมี Class)
```blade
{{-- resources/views/components/card.blade.php --}}
@props(['title' => null, 'class' => ''])

<div class="bg-white shadow rounded-lg overflow-hidden {{ $class }}">
    @if ($title)
        <div class="px-6 py-4 border-b">
            <h3 class="text-lg font-semibold">{{ $title }}</h3>
        </div>
    @endif
    
    <div class="p-6">
        {{ $slot }}
    </div>
    
    @isset($footer)
        <div class="px-6 py-4 border-t bg-gray-50">
            {{ $footer }}
        </div>
    @endisset
</div>
```

```blade
{{-- Usage --}}
<x-card title="ข้อมูลผู้ใช้">
    <p>{{ $user->name }}</p>
    <p>{{ $user->email }}</p>
    
    <x-slot name="footer">
        <a href="{{ route('users.edit', $user) }}">แก้ไข</a>
    </x-slot>
</x-card>
```

---

## ขั้นตอนที่ 496: Forms with Blade

```blade
{{-- Create Form --}}
<form action="{{ route('users.store') }}" method="POST" enctype="multipart/form-data">
    @csrf
    
    <div class="mb-4">
        <label for="name">ชื่อ</label>
        <input type="text" 
               id="name" 
               name="name" 
               value="{{ old('name') }}"
               class="form-control @error('name') is-invalid @enderror">
        
        @error('name')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>
    
    <div class="mb-4">
        <label for="role">บทบาท</label>
        <select name="role" id="role" class="form-select">
            @foreach (['admin' => 'Admin', 'editor' => 'Editor', 'user' => 'User'] as $value => $label)
                <option value="{{ $value }}" @selected(old('role') == $value)>
                    {{ $label }}
                </option>
            @endforeach
        </select>
    </div>
    
    <button type="submit">บันทึก</button>
</form>

{{-- Edit Form --}}
<form action="{{ route('users.update', $user) }}" method="POST">
    @csrf
    @method('PUT')  {{-- Override method --}}
    
    <input type="text" name="name" value="{{ old('name', $user->name) }}">
    
    <button type="submit">อัปเดต</button>
</form>

{{-- Delete Form --}}
<form action="{{ route('users.destroy', $user) }}" method="POST"
      onsubmit="return confirm('แน่ใจหรือไม่?')">
    @csrf
    @method('DELETE')
    <button type="submit" class="btn btn-danger">ลบ</button>
</form>
```

---

## ขั้นตอนที่ 497: Blade Directives (Custom)

```php
<?php
// app/Providers/AppServiceProvider.php

use Illuminate\Support\Facades\Blade;

public function boot(): void {
    // Custom directive
    Blade::directive('datetime', function ($expression) {
        return "<?php echo e(($expression)->format('d/m/Y H:i:s')); ?>";
    });
    
    Blade::directive('money', function ($expression) {
        return "<?php echo '฿' . number_format($expression, 2); ?>";
    });
    
    Blade::directive('nl2br', function ($expression) {
        return "<?php echo nl2br(e($expression)); ?>";
    });
    
    // Conditional directive
    Blade::if('admin', function () {
        return auth()->check() && auth()->user()->isAdmin();
    });
    
    Blade::if('role', function (string $role) {
        return auth()->check() && auth()->user()->role === $role;
    });
}
```

```blade
{{-- Usage of custom directives --}}
@datetime($user->created_at)
@money($product->price)
@nl2br($comment->body)

@admin
    <a href="/admin">Admin Panel</a>
@endadmin

@role('editor')
    <a href="/editor">Editor Area</a>
@endrole
```

---

## ขั้นตอนที่ 498: Livewire Component (ขั้นสูง)

```php
<?php
// composer require livewire/livewire
// php artisan make:livewire SearchUsers

namespace App\Livewire;

use App\Models\User;
use Livewire\Component;
use Livewire\WithPagination;

class SearchUsers extends Component {
    use WithPagination;
    
    public string $search = '';
    public string $sortField = 'name';
    public string $sortDirection = 'asc';
    
    public function updatingSearch(): void {
        $this->resetPage();
    }
    
    public function sortBy(string $field): void {
        if ($this->sortField === $field) {
            $this->sortDirection = $this->sortDirection === 'asc' ? 'desc' : 'asc';
        } else {
            $this->sortField = $field;
            $this->sortDirection = 'asc';
        }
    }
    
    public function deleteUser(int $id): void {
        User::findOrFail($id)->delete();
        session()->flash('success', 'User deleted');
    }
    
    public function render(): \Illuminate\View\View {
        return view('livewire.search-users', [
            'users' => User::query()
                ->when($this->search, fn($q) => $q->where('name', 'like', "%{$this->search}%")
                    ->orWhere('email', 'like', "%{$this->search}%"))
                ->orderBy($this->sortField, $this->sortDirection)
                ->paginate(10),
        ]);
    }
}
```

```blade
{{-- resources/views/livewire/search-users.blade.php --}}
<div>
    <input type="text" 
           wire:model.live="search" 
           placeholder="ค้นหาผู้ใช้..."
           class="border rounded px-3 py-2 w-full">
    
    <table class="w-full mt-4">
        <thead>
            <tr>
                <th wire:click="sortBy('name')" class="cursor-pointer">
                    ชื่อ
                    @if ($sortField === 'name')
                        {{ $sortDirection === 'asc' ? '↑' : '↓' }}
                    @endif
                </th>
                <th wire:click="sortBy('email')" class="cursor-pointer">
                    Email
                </th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @forelse ($users as $user)
                <tr>
                    <td>{{ $user->name }}</td>
                    <td>{{ $user->email }}</td>
                    <td>
                        <button wire:click="deleteUser({{ $user->id }})"
                                wire:confirm="แน่ใจหรือไม่?"
                                class="text-red-500">
                            ลบ
                        </button>
                    </td>
                </tr>
            @empty
                <tr><td colspan="3">ไม่พบผู้ใช้</td></tr>
            @endforelse
        </tbody>
    </table>
    
    {{ $users->links() }}
</div>
```

---

## 🎯 สรุป Part 19

| หัวข้อ | Directive/Feature |
|--------|------------------|
| Echo | {{ }}, {!! !!} |
| Conditionals | @if, @unless, @auth, @can |
| Loops | @foreach, @forelse, $loop |
| Layout | @extends, @section, @yield, @stack |
| Components | x-component, class components |
| Forms | @csrf, @method, @error, old() |
| Custom Directives | Blade::directive, Blade::if |
| Livewire | Reactive components, wire:model |

**ถัดไป → Part 21: Laravel Middleware & Auth**
