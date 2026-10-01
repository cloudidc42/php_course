# Part 56: Laravel Filament

## ขั้นตอนที่ 1541-1570: Filament PHP Admin Panel

Filament เป็น Admin Panel framework สำหรับ Laravel ที่สร้าง UI สวยงามได้รวดเร็ว พร้อม CRUD, Charts, Forms และ Tables ครบครัน

---

## ขั้นตอนที่ 1541: ติดตั้ง Filament

```bash
# ติดตั้ง Filament v3
composer require filament/filament:"^3.0"

# สร้าง admin user
php artisan filament:install --panels

# สร้าง user สำหรับ admin
php artisan make:filament-user

# publish assets
php artisan vendor:publish --tag=filament-config
```

```php
<?php

declare(strict_types=1);

// app/Providers/Filament/AdminPanelProvider.php
namespace App\Providers\Filament;

use Filament\Http\Middleware\Authenticate;
use Filament\Http\Middleware\DisableBladeIconComponents;
use Filament\Http\Middleware\DispatchServingFilamentEvent;
use Filament\Pages;
use Filament\Panel;
use Filament\PanelProvider;
use Filament\Support\Colors\Color;
use Filament\Widgets;
use Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse;
use Illuminate\Cookie\Middleware\EncryptCookies;
use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken;
use Illuminate\Routing\Middleware\SubstituteBindings;
use Illuminate\Session\Middleware\AuthenticateSession;
use Illuminate\Session\Middleware\StartSession;
use Illuminate\View\Middleware\ShareErrorsFromSession;

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->default()
            ->id('admin')
            ->path('admin')
            ->login()
            ->colors([
                'primary' => Color::Amber,
            ])
            ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
            ->discoverPages(in: app_path('Filament/Pages'), for: 'App\\Filament\\Pages')
            ->pages([
                Pages\Dashboard::class,
            ])
            ->discoverWidgets(in: app_path('Filament/Widgets'), for: 'App\\Filament\\Widgets')
            ->widgets([
                Widgets\AccountWidget::class,
                Widgets\FilamentInfoWidget::class,
            ])
            ->middleware([
                EncryptCookies::class,
                AddQueuedCookiesToResponse::class,
                StartSession::class,
                AuthenticateSession::class,
                ShareErrorsFromSession::class,
                VerifyCsrfToken::class,
                SubstituteBindings::class,
                DisableBladeIconComponents::class,
                DispatchServingFilamentEvent::class,
            ])
            ->authMiddleware([
                Authenticate::class,
            ]);
    }
}
```

---

## ขั้นตอนที่ 1542: สร้าง Resource สำหรับ CRUD

```bash
# สร้าง Post resource
php artisan make:filament-resource Post --generate

# สร้างพร้อม soft delete
php artisan make:filament-resource Post --soft-deletes --generate
```

```php
<?php

declare(strict_types=1);

// app/Filament/Resources/PostResource.php
namespace App\Filament\Resources;

use App\Filament\Resources\PostResource\Pages;
use App\Filament\Resources\PostResource\RelationManagers;
use App\Models\Post;
use Filament\Forms;
use Filament\Forms\Form;
use Filament\Resources\Resource;
use Filament\Tables;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class PostResource extends Resource
{
    protected static ?string $model = Post::class;
    protected static ?string $navigationIcon = 'heroicon-o-document-text';
    protected static ?string $navigationGroup = 'Content Management';
    protected static ?int $navigationSort = 1;

    public static function form(Form $form): Form
    {
        return $form
            ->schema([
                Forms\Components\Section::make('Post Information')
                    ->schema([
                        Forms\Components\TextInput::make('title')
                            ->required()
                            ->maxLength(255)
                            ->live(onBlur: true)
                            ->afterStateUpdated(fn (string $operation, $state, Forms\Set $set) =>
                                $operation === 'create' ? $set('slug', \Str::slug($state)) : null
                            ),

                        Forms\Components\TextInput::make('slug')
                            ->required()
                            ->maxLength(255)
                            ->unique(Post::class, 'slug', ignoreRecord: true),

                        Forms\Components\Select::make('category_id')
                            ->relationship('category', 'name')
                            ->searchable()
                            ->preload()
                            ->createOptionForm([
                                Forms\Components\TextInput::make('name')
                                    ->required()
                                    ->maxLength(255),
                            ]),

                        Forms\Components\Select::make('status')
                            ->options([
                                'draft'     => 'Draft',
                                'published' => 'Published',
                                'archived'  => 'Archived',
                            ])
                            ->default('draft')
                            ->required(),
                    ])
                    ->columns(2),

                Forms\Components\Section::make('Content')
                    ->schema([
                        Forms\Components\RichEditor::make('content')
                            ->required()
                            ->columnSpanFull()
                            ->toolbarButtons([
                                'attachFiles',
                                'blockquote',
                                'bold',
                                'bulletList',
                                'codeBlock',
                                'h2',
                                'h3',
                                'italic',
                                'link',
                                'orderedList',
                                'redo',
                                'strike',
                                'underline',
                                'undo',
                            ]),

                        Forms\Components\TagsInput::make('tags')
                            ->separator(','),
                    ]),

                Forms\Components\Section::make('SEO')
                    ->schema([
                        Forms\Components\TextInput::make('meta_title')
                            ->maxLength(60),

                        Forms\Components\Textarea::make('meta_description')
                            ->maxLength(160)
                            ->rows(3),
                    ])
                    ->columns(1)
                    ->collapsed(),

                Forms\Components\Section::make('Media')
                    ->schema([
                        Forms\Components\FileUpload::make('featured_image')
                            ->image()
                            ->imageEditor()
                            ->directory('posts/images')
                            ->visibility('public'),
                    ]),
            ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\ImageColumn::make('featured_image')
                    ->label('Image')
                    ->circular(),

                Tables\Columns\TextColumn::make('title')
                    ->searchable()
                    ->sortable()
                    ->weight(\Filament\Support\Enums\FontWeight::Bold),

                Tables\Columns\TextColumn::make('category.name')
                    ->badge()
                    ->color('primary'),

                Tables\Columns\SelectColumn::make('status')
                    ->options([
                        'draft'     => 'Draft',
                        'published' => 'Published',
                        'archived'  => 'Archived',
                    ])
                    ->sortable(),

                Tables\Columns\TextColumn::make('created_at')
                    ->dateTime()
                    ->sortable()
                    ->toggleable(isToggledHiddenByDefault: true),
            ])
            ->filters([
                Tables\Filters\SelectFilter::make('status')
                    ->options([
                        'draft'     => 'Draft',
                        'published' => 'Published',
                        'archived'  => 'Archived',
                    ]),

                Tables\Filters\SelectFilter::make('category')
                    ->relationship('category', 'name'),

                Tables\Filters\TrashedFilter::make(),
            ])
            ->actions([
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make(),
                Tables\Actions\RestoreAction::make(),
            ])
            ->bulkActions([
                Tables\Actions\BulkActionGroup::make([
                    Tables\Actions\DeleteBulkAction::make(),
                    Tables\Actions\RestoreBulkAction::make(),
                    Tables\Actions\ForceDeleteBulkAction::make(),
                ]),
            ])
            ->defaultSort('created_at', 'desc');
    }

    public static function getRelations(): array
    {
        return [
            RelationManagers\CommentsRelationManager::class,
            RelationManagers\TagsRelationManager::class,
        ];
    }

    public static function getPages(): array
    {
        return [
            'index'  => Pages\ListPosts::route('/'),
            'create' => Pages\CreatePost::route('/create'),
            'edit'   => Pages\EditPost::route('/{record}/edit'),
        ];
    }

    public static function getEloquentQuery(): Builder
    {
        return parent::getEloquentQuery()
            ->withoutGlobalScopes([
                SoftDeletingScope::class,
            ]);
    }
}
```

---

## ขั้นตอนที่ 1543: Relation Managers

```php
<?php

declare(strict_types=1);

// app/Filament/Resources/PostResource/RelationManagers/CommentsRelationManager.php
namespace App\Filament\Resources\PostResource\RelationManagers;

use Filament\Forms;
use Filament\Forms\Form;
use Filament\Resources\RelationManagers\RelationManager;
use Filament\Tables;
use Filament\Tables\Table;

class CommentsRelationManager extends RelationManager
{
    protected static string $relationship = 'comments';
    protected static ?string $recordTitleAttribute = 'content';

    public function form(Form $form): Form
    {
        return $form
            ->schema([
                Forms\Components\TextInput::make('author_name')
                    ->required()
                    ->maxLength(255),

                Forms\Components\TextInput::make('author_email')
                    ->email()
                    ->required(),

                Forms\Components\Textarea::make('content')
                    ->required()
                    ->columnSpanFull(),

                Forms\Components\Toggle::make('is_approved')
                    ->label('Approved'),
            ]);
    }

    public function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('author_name'),
                Tables\Columns\TextColumn::make('content')->limit(50),
                Tables\Columns\IconColumn::make('is_approved')->boolean(),
                Tables\Columns\TextColumn::make('created_at')->dateTime(),
            ])
            ->headerActions([
                Tables\Actions\CreateAction::make(),
            ])
            ->actions([
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make(),
                Tables\Actions\Action::make('approve')
                    ->icon('heroicon-o-check')
                    ->action(fn ($record) => $record->update(['is_approved' => true]))
                    ->visible(fn ($record) => !$record->is_approved),
            ])
            ->bulkActions([
                Tables\Actions\BulkAction::make('approve')
                    ->action(fn ($records) => $records->each->update(['is_approved' => true]))
                    ->icon('heroicon-o-check'),
                Tables\Actions\DeleteBulkAction::make(),
            ]);
    }
}
```

---

## ขั้นตอนที่ 1544: Custom Widgets และ Charts

```php
<?php

declare(strict_types=1);

// app/Filament/Widgets/StatsOverviewWidget.php
namespace App\Filament\Widgets;

use App\Models\Order;
use App\Models\Post;
use App\Models\User;
use Filament\Widgets\StatsOverviewWidget as BaseWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

class StatsOverviewWidget extends BaseWidget
{
    protected static ?int $sort = 1;

    protected function getStats(): array
    {
        $totalUsers = User::count();
        $newUsersThisMonth = User::whereMonth('created_at', now()->month)->count();

        $totalRevenue = Order::where('status', 'completed')->sum('total');
        $revenueThisMonth = Order::where('status', 'completed')
            ->whereMonth('created_at', now()->month)
            ->sum('total');

        $publishedPosts = Post::where('status', 'published')->count();

        // สร้าง sparkline data
        $userTrend = User::selectRaw('DATE(created_at) as date, COUNT(*) as count')
            ->whereDate('created_at', '>=', now()->subDays(7))
            ->groupBy('date')
            ->pluck('count')
            ->toArray();

        return [
            Stat::make('Total Users', number_format($totalUsers))
                ->description("+{$newUsersThisMonth} this month")
                ->descriptionIcon('heroicon-m-arrow-trending-up')
                ->color('success')
                ->chart($userTrend),

            Stat::make('Total Revenue', '฿' . number_format($totalRevenue, 2))
                ->description('฿' . number_format($revenueThisMonth, 2) . ' this month')
                ->descriptionIcon('heroicon-m-banknotes')
                ->color('primary'),

            Stat::make('Published Posts', number_format($publishedPosts))
                ->description('Active content')
                ->descriptionIcon('heroicon-m-document-text')
                ->color('warning'),
        ];
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Filament/Widgets/RevenueChart.php
namespace App\Filament\Widgets;

use App\Models\Order;
use Filament\Widgets\ChartWidget;
use Illuminate\Support\Carbon;

class RevenueChart extends ChartWidget
{
    protected static ?string $heading = 'Revenue Overview';
    protected static ?int $sort = 2;

    protected function getData(): array
    {
        $months = collect(range(11, 0))->map(fn ($i) => now()->subMonths($i));

        $labels = $months->map(fn (Carbon $m) => $m->format('M Y'))->toArray();

        $revenues = $months->map(fn (Carbon $m) => Order::where('status', 'completed')
            ->whereYear('created_at', $m->year)
            ->whereMonth('created_at', $m->month)
            ->sum('total')
        )->toArray();

        return [
            'datasets' => [
                [
                    'label'           => 'Revenue (THB)',
                    'data'            => $revenues,
                    'backgroundColor' => 'rgba(59, 130, 246, 0.1)',
                    'borderColor'     => 'rgb(59, 130, 246)',
                    'fill'            => true,
                    'tension'         => 0.3,
                ],
            ],
            'labels' => $labels,
        ];
    }

    protected function getType(): string
    {
        return 'line';
    }
}
```

---

## ขั้นตอนที่ 1545: Custom Pages

```php
<?php

declare(strict_types=1);

// app/Filament/Pages/Settings.php
namespace App\Filament\Pages;

use App\Models\Setting;
use Filament\Actions\Action;
use Filament\Forms\Components\FileUpload;
use Filament\Forms\Components\Section;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\Toggle;
use Filament\Forms\Concerns\InteractsWithForms;
use Filament\Forms\Contracts\HasForms;
use Filament\Forms\Form;
use Filament\Notifications\Notification;
use Filament\Pages\Page;

class Settings extends Page implements HasForms
{
    use InteractsWithForms;

    protected static ?string $navigationIcon = 'heroicon-o-cog-6-tooth';
    protected static string $view = 'filament.pages.settings';
    protected static ?string $navigationGroup = 'System';
    protected static ?int $navigationSort = 100;

    public ?array $data = [];

    public function mount(): void
    {
        $settings = Setting::all()->pluck('value', 'key')->toArray();
        $this->form->fill($settings);
    }

    public function form(Form $form): Form
    {
        return $form
            ->schema([
                Section::make('Site Configuration')
                    ->schema([
                        TextInput::make('site_name')
                            ->required()
                            ->maxLength(255),

                        TextInput::make('site_url')
                            ->url()
                            ->required(),

                        Textarea::make('site_description')
                            ->rows(3),

                        FileUpload::make('site_logo')
                            ->image()
                            ->directory('settings'),
                    ])
                    ->columns(2),

                Section::make('Email Settings')
                    ->schema([
                        TextInput::make('mail_from_address')
                            ->email()
                            ->required(),

                        TextInput::make('mail_from_name')
                            ->required(),
                    ])
                    ->columns(2),

                Section::make('Features')
                    ->schema([
                        Toggle::make('maintenance_mode')
                            ->label('Maintenance Mode'),

                        Toggle::make('registration_enabled')
                            ->label('User Registration'),

                        Toggle::make('comments_enabled')
                            ->label('Comments'),
                    ])
                    ->columns(3),
            ])
            ->statePath('data');
    }

    protected function getFormActions(): array
    {
        return [
            Action::make('save')
                ->label('Save Settings')
                ->submit('save'),
        ];
    }

    public function save(): void
    {
        $data = $this->form->getState();

        foreach ($data as $key => $value) {
            Setting::updateOrCreate(
                ['key' => $key],
                ['value' => $value]
            );
        }

        Notification::make()
            ->title('Settings saved successfully')
            ->success()
            ->send();
    }
}
```

---

## ขั้นตอนที่ 1546: Custom Actions และ Bulk Actions

```php
<?php

declare(strict_types=1);

// app/Filament/Resources/UserResource.php (ส่วน Actions)
namespace App\Filament\Resources;

use App\Models\User;
use Filament\Forms;
use Filament\Resources\Resource;
use Filament\Tables;
use Filament\Tables\Table;
use Filament\Notifications\Notification;
use Illuminate\Support\Facades\Mail;

class UserResource extends Resource
{
    protected static ?string $model = User::class;
    protected static ?string $navigationIcon = 'heroicon-o-users';

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('name')->searchable(),
                Tables\Columns\TextColumn::make('email')->searchable(),
                Tables\Columns\BadgeColumn::make('role')
                    ->colors([
                        'danger'  => 'admin',
                        'warning' => 'editor',
                        'success' => 'user',
                    ]),
                Tables\Columns\IconColumn::make('email_verified_at')
                    ->label('Verified')
                    ->boolean()
                    ->trueIcon('heroicon-o-check-badge')
                    ->falseIcon('heroicon-o-x-circle'),
            ])
            ->actions([
                Tables\Actions\EditAction::make(),

                Tables\Actions\Action::make('sendVerification')
                    ->label('Send Verification')
                    ->icon('heroicon-o-envelope')
                    ->color('info')
                    ->action(function (User $record): void {
                        $record->sendEmailVerificationNotification();
                        Notification::make()
                            ->title('Verification email sent')
                            ->success()
                            ->send();
                    })
                    ->visible(fn (User $record) => !$record->hasVerifiedEmail()),

                Tables\Actions\Action::make('impersonate')
                    ->label('Login As')
                    ->icon('heroicon-o-arrow-right-on-rectangle')
                    ->color('warning')
                    ->requiresConfirmation()
                    ->action(function (User $record) {
                        auth()->login($record);
                        return redirect('/');
                    }),

                Tables\Actions\ActionGroup::make([
                    Tables\Actions\Action::make('ban')
                        ->label('Ban User')
                        ->icon('heroicon-o-no-symbol')
                        ->color('danger')
                        ->requiresConfirmation()
                        ->form([
                            Forms\Components\Textarea::make('reason')
                                ->label('Ban Reason')
                                ->required(),
                        ])
                        ->action(function (User $record, array $data): void {
                            $record->ban(['comment' => $data['reason']]);
                            Notification::make()
                                ->title('User has been banned')
                                ->danger()
                                ->send();
                        }),

                    Tables\Actions\DeleteAction::make(),
                ]),
            ])
            ->bulkActions([
                Tables\Actions\BulkActionGroup::make([
                    Tables\Actions\BulkAction::make('sendNewsletter')
                        ->label('Send Newsletter')
                        ->icon('heroicon-o-envelope')
                        ->form([
                            Forms\Components\TextInput::make('subject')->required(),
                            Forms\Components\RichEditor::make('body')->required(),
                        ])
                        ->action(function ($records, array $data): void {
                            foreach ($records as $record) {
                                Mail::to($record->email)->queue(
                                    new \App\Mail\Newsletter($data['subject'], $data['body'])
                                );
                            }
                            Notification::make()
                                ->title('Newsletter queued for ' . $records->count() . ' users')
                                ->success()
                                ->send();
                        }),

                    Tables\Actions\DeleteBulkAction::make(),
                ]),
            ]);
    }

    public static function getRelations(): array { return []; }
    public static function getPages(): array
    {
        return [
            'index'  => \App\Filament\Resources\UserResource\Pages\ListUsers::route('/'),
            'create' => \App\Filament\Resources\UserResource\Pages\CreateUser::route('/create'),
            'edit'   => \App\Filament\Resources\UserResource\Pages\EditUser::route('/{record}/edit'),
        ];
    }
}
```

---

## ขั้นตอนที่ 1547: Filament Notifications และ Livewire Integration

```php
<?php

declare(strict_types=1);

// app/Filament/Widgets/LatestOrdersWidget.php
namespace App\Filament\Widgets;

use App\Models\Order;
use Filament\Tables;
use Filament\Tables\Table;
use Filament\Widgets\TableWidget as BaseWidget;

class LatestOrdersWidget extends BaseWidget
{
    protected static ?int $sort = 3;
    protected int | string | array $columnSpan = 'full';

    public function table(Table $table): Table
    {
        return $table
            ->query(Order::query()->latest()->limit(5))
            ->columns([
                Tables\Columns\TextColumn::make('order_number')
                    ->label('Order #')
                    ->searchable(),

                Tables\Columns\TextColumn::make('user.name')
                    ->label('Customer'),

                Tables\Columns\TextColumn::make('total')
                    ->money('THB')
                    ->sortable(),

                Tables\Columns\BadgeColumn::make('status')
                    ->colors([
                        'warning' => 'pending',
                        'primary' => 'processing',
                        'success' => 'completed',
                        'danger'  => 'cancelled',
                    ]),

                Tables\Columns\TextColumn::make('created_at')
                    ->dateTime()
                    ->since(),
            ])
            ->actions([
                Tables\Actions\Action::make('view')
                    ->url(fn (Order $record) => route('filament.admin.resources.orders.edit', $record))
                    ->icon('heroicon-o-eye'),
            ]);
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Listeners/SendFilamentNotification.php
namespace App\Listeners;

use App\Events\OrderPlaced;
use Filament\Notifications\Actions\Action;
use Filament\Notifications\Notification;
use App\Models\User;

class SendFilamentNotification
{
    public function handle(OrderPlaced $event): void
    {
        $admins = User::where('role', 'admin')->get();

        foreach ($admins as $admin) {
            Notification::make()
                ->title('New Order Received')
                ->body("Order #{$event->order->order_number} - ฿" . number_format($event->order->total, 2))
                ->icon('heroicon-o-shopping-cart')
                ->iconColor('success')
                ->actions([
                    Action::make('view')
                        ->label('View Order')
                        ->url(route('filament.admin.resources.orders.edit', $event->order))
                        ->button(),
                ])
                ->sendToDatabase($admin);
        }
    }
}
```

---

## ขั้นตอนที่ 1548: Custom Form Components

```php
<?php

declare(strict_types=1);

// app/Filament/Forms/Components/AddressComponent.php
namespace App\Filament\Forms\Components;

use Filament\Forms\Components\Component;
use Filament\Forms\Components\Grid;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;

class AddressComponent extends Component
{
    protected string $view = 'filament.forms.components.address';

    public static function make(string $field = 'address'): static
    {
        $static = app(static::class, ['field' => $field]);
        $static->configure();
        return $static;
    }

    protected function setUp(): void
    {
        parent::setUp();

        $this->schema([
            Grid::make(2)
                ->schema([
                    TextInput::make('address_line1')
                        ->label('Address Line 1')
                        ->required(),

                    TextInput::make('address_line2')
                        ->label('Address Line 2'),

                    TextInput::make('city')
                        ->required(),

                    TextInput::make('state')
                        ->label('Province/State')
                        ->required(),

                    TextInput::make('postal_code')
                        ->label('Postal Code')
                        ->required()
                        ->maxLength(10),

                    Select::make('country')
                        ->options(\App\Data\Countries::list())
                        ->searchable()
                        ->default('TH')
                        ->required(),
                ]),
        ]);
    }
}
```

---

## ขั้นตอนที่ 1549: Filament Policies และ Authorization

```php
<?php

declare(strict_types=1);

// app/Policies/PostPolicy.php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class PostPolicy
{
    use HandlesAuthorization;

    public function viewAny(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }

    public function view(User $user, Post $post): bool
    {
        return $user->hasRole('admin') || $post->author_id === $user->id;
    }

    public function create(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }

    public function update(User $user, Post $post): bool
    {
        return $user->hasRole('admin') || $post->author_id === $user->id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }

    public function restore(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }

    public function forceDelete(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Filament/Resources/PostResource.php (เพิ่ม navigation badge)
// เพิ่มใน PostResource class

public static function getNavigationBadge(): ?string
{
    return static::getModel()::where('status', 'draft')->count() ?: null;
}

public static function getNavigationBadgeColor(): string | array | null
{
    return static::getModel()::where('status', 'draft')->count() > 5
        ? 'danger'
        : 'warning';
}

public static function getGlobalSearchResultTitle(\Illuminate\Database\Eloquent\Model $record): string | \Illuminate\Contracts\Support\Htmlable
{
    return $record->title;
}

public static function getGlobalSearchResultDetails(\Illuminate\Database\Eloquent\Model $record): array
{
    return [
        'Category' => $record->category?->name,
        'Status'   => $record->status,
        'Author'   => $record->author?->name,
    ];
}

public static function getGloballySearchableAttributes(): array
{
    return ['title', 'content', 'meta_title'];
}
```

---

## ขั้นตอนที่ 1550: Multi-tenancy ใน Filament

```php
<?php

declare(strict_types=1);

// app/Providers/Filament/AdminPanelProvider.php (multi-tenant version)
namespace App\Providers\Filament;

use App\Models\Team;
use Filament\Http\Middleware\Authenticate;
use Filament\Panel;
use Filament\PanelProvider;
use Filament\Pages\Tenancy\EditTenantProfile;
use Filament\Pages\Tenancy\RegisterTenant;

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('admin')
            ->path('admin')
            ->tenant(Team::class)
            ->tenantRegistration(RegisterTenant::class)
            ->tenantProfile(EditTenantProfile::class)
            ->login()
            ->registration()
            ->authMiddleware([
                Authenticate::class,
            ]);
    }
}

// app/Models/Team.php
// implements HasCurrentTenantLabel interface
namespace App\Models;

use Filament\Models\Contracts\HasCurrentTenantLabel;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Team extends Model implements HasCurrentTenantLabel
{
    protected $fillable = ['name', 'slug'];

    public function getCurrentTenantLabel(): string
    {
        return $this->name;
    }

    public function members(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'team_user')
            ->withPivot('role')
            ->withTimestamps();
    }
}
```

---

## สรุป Part 56

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Filament Resource | CRUD อัตโนมัติพร้อม Form และ Table |
| Relation Managers | จัดการ relations จาก parent resource |
| Custom Widgets | Stats cards, Charts, Table widgets |
| Custom Pages | หน้า settings ที่ใช้ Filament Forms |
| Custom Actions | Actions และ Bulk Actions ที่กำหนดเอง |
| Authorization | Policy-based access control |
| Multi-tenancy | รองรับหลาย tenant ใน panel เดียว |

ถัดไป → Part 57: Elasticsearch Search Integration
