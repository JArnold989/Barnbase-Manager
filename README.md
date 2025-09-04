# Barnbase-Manager

# Barn Base — Manager Repo Starter Pack (v0)

This pack contains copy‑pasteable stubs to spin up the **Manager** repo (`barnbase-manager`). It matches the approved Scope v0.

> **Stack**: Laravel 11, PHP 8.3, MySQL 8, Redis 7

---

## 1) README.md (stub)

````md
# Barn Base — Manager (v0)

Admin portal + public API for tenant resolution.

## Prerequisites
- PHP 8.3+
- Composer 2+
- MySQL 8+
- Redis 7+ (local cache)

## Quick start
```bash
cp .env.example .env
composer install
php artisan key:generate
php artisan migrate
php artisan db:seed --class=TenantSeeder
php artisan serve --host=manager.barnbase.test --port=8000
````

## .env essentials

* `APP_URL=http://manager.barnbase.test`
* `DB_*` for MySQL 8
* `REDIS_*` (optional; file cache is OK for v0)

## Routes

* **Portal**: `GET /tenants` (list), `POST /tenants` (create), edit/update, soft delete, restore
* **API**: `GET /api/tenants/resolve?subdomain={sub}` → 200/404/503

## Reserved subdomains

`www, admin, manager, api, app, mail, test, dev, staging`

## Testing & style

```bash
composer test
composer pint
```

* Coverage gate \~80% (see `phpunit.xml`)

## Caching/versioning

* No tenant cache in Manager.
* API includes `version` (hash of `updated_at + hex_color`).

## Security (v0)

* No auth on portal (keep non-public for v0).
* Rate limit API; strict validation; CORS only `*.barnbase.*`

## Seeding

```bash
php artisan tenants:seed
```

## License

Proprietary — Barn Base internal.

```
```

---

## 2) .env.example (minimal)

```dotenv
APP_NAME="Barn Base Manager"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://manager.barnbase.test

LOG_CHANNEL=stack
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=barnbase_manager
DB_USERNAME=root
DB_PASSWORD=

BROADCAST_DRIVER=log
CACHE_DRIVER=redis
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120

REDIS_CLIENT=phpredis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

CORS_ALLOWED_ORIGINS=http://*.barnbase.test,https://*.barnbase.com
```

---

## 3) Migration — `database/migrations/2025_09_04_000000_create_tenants_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('tenants', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('name', 150);
            $table->string('subdomain', 63)->unique();
            $table->char('hex_color', 7);
            $table->enum('status', ['active','suspended'])->default('active');
            $table->timestamps();
            $table->softDeletes();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tenants');
    }
};
```

---

## 4) Enum (optional) — `app/Enums/TenantStatus.php`

```php
<?php

namespace App\Enums;

enum TenantStatus: string
{
    case Active = 'active';
    case Suspended = 'suspended';
}
```

---

## 5) Model — `app/Models/Tenant.php`

```php
<?php

namespace App\Models;

use App\Enums\TenantStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Tenant extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'name','subdomain','hex_color','status',
    ];

    protected $casts = [
        'status' => TenantStatus::class,
    ];
}
```

---

## 6) Form Requests

**`app/Http/Requests/StoreTenantRequest.php`**

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreTenantRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'name' => ['required','string','max:150'],
            'subdomain' => ['required','string','max:63','regex:/^[a-z0-9-]+$/','unique:tenants,subdomain', 'not_in:www,admin,manager,api,app,mail,test,dev,staging'],
            'hex_color' => ['required','regex:/^#[0-9A-Fa-f]{6}$/'],
            'status' => ['required','in:active,suspended'],
        ];
    }
}
```

**`app/Http/Requests/UpdateTenantRequest.php`**

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateTenantRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'name' => ['required','string','max:150'],
            'subdomain' => [
                'required','string','max:63','regex:/^[a-z0-9-]+$/',
                Rule::unique('tenants','subdomain')->ignore($this->route('tenant')),
                'not_in:www,admin,manager,api,app,mail,test,dev,staging',
            ],
            'hex_color' => ['required','regex:/^#[0-9A-Fa-f]{6}$/'],
            'status' => ['required','in:active,suspended'],
        ];
    }
}
```

---

## 7) Controller — Portal `app/Http/Controllers/TenantController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreTenantRequest;
use App\Http\Requests\UpdateTenantRequest;
use App\Models\Tenant;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\View\View;

class TenantController extends Controller
{
    public function index(): View
    {
        $tenants = Tenant::withTrashed()->latest()->paginate(20);
        return view('tenants.index', compact('tenants'));
    }

    public function create(): View
    {
        return view('tenants.create');
    }

    public function store(StoreTenantRequest $request): RedirectResponse
    {
        Tenant::create($request->validated());
        return redirect()->route('tenants.index')->with('status','Tenant created');
    }

    public function edit(Tenant $tenant): View
    {
        return view('tenants.edit', compact('tenant'));
    }

    public function update(UpdateTenantRequest $request, Tenant $tenant): RedirectResponse
    {
        $tenant->update($request->validated());
        return redirect()->route('tenants.index')->with('status','Tenant updated');
    }

    public function destroy(Tenant $tenant): RedirectResponse
    {
        $tenant->delete();
        return back()->with('status','Tenant soft-deleted');
    }

    public function restore(int $id): RedirectResponse
    {
        $tenant = Tenant::withTrashed()->findOrFail($id);
        $tenant->restore();
        return back()->with('status','Tenant restored');
    }
}
```

---

## 8) Controller — API `app/Http/Controllers/Api/TenantResolveController.php`

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Tenant;
use App\Services\TenantVersionService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class TenantResolveController extends Controller
{
    public function __invoke(Request $request, TenantVersionService $version): JsonResponse
    {
        $sub = strtolower((string) $request->query('subdomain'));
        if ($sub === '' || !preg_match('/^[a-z0-9-]+$/', $sub)) {
            return response()->json(['message' => 'Invalid subdomain'], 422);
        }

        $tenant = Tenant::where('subdomain', $sub)->first();
        if (! $tenant || $tenant->trashed()) {
            return response()->json(['message' => 'Not found'], 404);
        }

        if ($tenant->status->value === 'suspended') {
            return response()->json(['message' => 'Tenant suspended'], 503);
        }

        return response()->json([
            'id' => $tenant->id,
            'name' => $tenant->name,
            'subdomain' => $tenant->subdomain,
            'hex_color' => $tenant->hex_color,
            'status' => $tenant->status->value,
            'updated_at' => $tenant->updated_at?->toISOString(),
            'version' => $version->hash($tenant->updated_at, $tenant->hex_color),
        ]);
    }
}
```

---

## 9) Service — `app/Services/TenantVersionService.php`

```php
<?php

namespace App\Services;

use Illuminate\Support\Carbon;

class TenantVersionService
{
    public function hash(?\DateTimeInterface $updatedAt, string $hexColor): string
    {
        $basis = ($updatedAt?->format('c') ?? 'null') . '|' . strtolower($hexColor);
        return substr(hash('xxh128', $basis), 0, 8);
    }
}
```

---

## 10) Routes

**`routes/web.php`**

```php
<?php

use App\Http\Controllers\TenantController;
use Illuminate\Support\Facades\Route;

Route::get('/', fn() => redirect()->route('tenants.index'));

Route::controller(TenantController::class)->group(function () {
    Route::get('/tenants', 'index')->name('tenants.index');
    Route::get('/tenants/create', 'create')->name('tenants.create');
    Route::post('/tenants', 'store')->name('tenants.store');
    Route::get('/tenants/{tenant}/edit', 'edit')->name('tenants.edit');
    Route::put('/tenants/{tenant}', 'update')->name('tenants.update');
    Route::delete('/tenants/{tenant}', 'destroy')->name('tenants.destroy');
    Route::post('/tenants/{id}/restore', 'restore')->name('tenants.restore');
});
```

**`routes/api.php`**

```php
<?php

use App\Http\Controllers\Api\TenantResolveController;
use Illuminate\Support\Facades\Route;

Route::get('/tenants/resolve', TenantResolveController::class)->middleware('throttle:60,1');
```

---

## 11) Seeder & Command

**`database/seeders/TenantSeeder.php`**

```php
<?php

namespace Database\Seeders;

use App\Models\Tenant;
use Illuminate\Database\Seeder;

class TenantSeeder extends Seeder
{
    public function run(): void
    {
        $rows = [
            ['name' => "Bob's Stables", 'subdomain' => 'bobsstables', 'hex_color' => '#3B82F6', 'status' => 'active'],
            ['name' => 'Acme Riding', 'subdomain' => 'acme', 'hex_color' => '#10B981', 'status' => 'active'],
            ['name' => 'Paused Yard', 'subdomain' => 'paused', 'hex_color' => '#F59E0B', 'status' => 'suspended'],
        ];
        foreach ($rows as $r) { Tenant::updateOrCreate(['subdomain' => $r['subdomain']], $r); }
    }
}
```

**Optional command `app/Console/Commands/TenantsSeedCommand.php`**

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;

class TenantsSeedCommand extends Command
{
    protected $signature = 'tenants:seed';
    protected $description = 'Seed demo tenants';

    public function handle(): int
    {
        $this->call('db:seed', ['--class' => 'Database\\Seeders\\TenantSeeder']);
        $this->info('Seeded demo tenants.');
        return self::SUCCESS;
    }
}
```

---

## 12) Tests (PHPUnit)

**`tests/Feature/TenantCrudTest.php`** (portal basics)

```php
<?php

use App\Models\Tenant;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('creates a tenant', function () {
    $resp = $this->post('/tenants', [
        'name' => 'Test', 'subdomain' => 'test', 'hex_color' => '#3366FF', 'status' => 'active',
    ]);
    $resp->assertRedirect(route('tenants.index'));
    expect(Tenant::where('subdomain','test')->exists())->toBeTrue();
});

it('rejects reserved subdomain', function () {
    $resp = $this->post('/tenants', [
        'name' => 'Bad', 'subdomain' => 'manager', 'hex_color' => '#3366FF', 'status' => 'active',
    ]);
    $resp->assertSessionHasErrors('subdomain');
});
```

**`tests/Feature/TenantResolveApiTest.php`**

```php
<?php

use App\Models\Tenant;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('resolves active tenant', function () {
    $t = Tenant::factory()->create(['subdomain' => 'bobs', 'status' => 'active', 'hex_color' => '#123456']);
    $resp = $this->getJson('/api/tenants/resolve?subdomain=bobs');
    $resp->assertOk()->assertJsonPath('subdomain', 'bobs')->assertJsonPath('hex_color', '#123456');
});

it('404 for unknown', function () {
    $this->getJson('/api/tenants/resolve?subdomain=unknown')->assertNotFound();
});

it('503 for suspended', function () {
    Tenant::factory()->create(['subdomain' => 'paused', 'status' => 'suspended']);
    $this->getJson('/api/tenants/resolve?subdomain=paused')->assertStatus(503);
});
```

> **Note**: If you prefer Pest, these examples are compatible. For PHPUnit classic, wrap in `class` test cases.

---

## 13) Factories (optional but handy)

**`database/factories/TenantFactory.php`**

```php
<?php

namespace Database\Factories;

use App\Models\Tenant;
use Illuminate\Database\Eloquent\Factories\Factory;

class TenantFactory extends Factory
{
    protected $model = Tenant::class;

    public function definition(): array
    {
        return [
            'name' => fake()->company(),
            'subdomain' => strtolower(fake()->unique()->bothify('tenant-##')),
            'hex_color' => '#' . strtoupper(fake()->hexColor()),
            'status' => 'active',
        ];
    }
}
```

---

## 14) Composer extras & tooling

**`composer.json` snippets**

```json
{
  "scripts": {
    "test": "phpunit --colors=always",
    "pint": "vendor/bin/pint",
    "ci": "composer install --no-interaction && php -v && composer test"
  },
  "require-dev": {
    "laravel/pint": "^1.0",
    "pestphp/pest": "^2.0",
    "pestphp/pest-plugin-laravel": "^2.0"
  }
}
```

**`pint.json`**

```json
{
  "$schema": "https://raw.githubusercontent.com/laravel/pint/main/pint.schema.json",
  "preset": "laravel"
}
```

**`.editorconfig`**

```ini
root = true
[*]
charset = utf-8
indent_style = space
indent_size = 4
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
```

---

## 15) PHPUnit config (coverage gate)

**`phpunit.xml` (fragment)**

```xml
<phpunit colors="true" bootstrap="vendor/autoload.php">
  <testsuites>
    <testsuite name="Feature"> <directory>tests/Feature</directory> </testsuite>
    <testsuite name="Unit"> <directory>tests/Unit</directory> </testsuite>
  </testsuites>
  <coverage processUncoveredFiles="true">
    <include> <directory suffix=".php">app</directory> </include>
    <report> <text outputFile="php://stdout" showOnlySummary="false"/> </report>
    <limits>
      <line>80</line>
    </limits>
  </coverage>
</phpunit>
```

---

## 16) GitHub Actions CI

**`.github/workflows/ci.yml`**

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: barnbase_manager
        ports: ['3306:3306']
        options: >-
          --health-cmd "mysqladmin ping -h localhost"
          --health-interval 10s --health-timeout 5s --health-retries 3
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.3' }
      - run: composer install --no-interaction --prefer-dist
      - run: cp .env.example .env
      - run: php artisan key:generate
      - run: php -r "file_put_contents('database/database.sqlite','');" || true
      - run: php artisan migrate --force
      - run: php artisan db:seed --class=TenantSeeder --force
      - run: vendor/bin/pest || vendor/bin/phpunit
```

---

## 17) Minimal Blade views (portal)

**`resources/views/layouts/app.blade.php`**

```blade
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Barn Base Manager</title>
  @vite('resources/css/app.css')
</head>
<body class="font-sans antialiased">
  <main class="max-w-5xl mx-auto p-6">
    @if(session('status'))
      <div class="mb-4 p-3 bg-green-100">{{ session('status') }}</div>
    @endif
    @yield('content')
  </main>
</body>
</html>
```

**`resources/views/tenants/index.blade.php`**

```blade
@extends('layouts.app')
@section('content')
<h1 class="text-2xl font-bold mb-4">Tenants</h1>
<a href="{{ route('tenants.create') }}" class="underline">Create</a>
<table class="w-full mt-4 border">
  <thead><tr><th>ID</th><th>Name</th><th>Subdomain</th><th>Status</th><th>Deleted?</th><th></th></tr></thead>
  <tbody>
  @foreach($tenants as $t)
    <tr>
      <td>{{ $t->id }}</td>
      <td>{{ $t->name }}</td>
      <td>{{ $t->subdomain }}</td>
      <td>{{ $t->status->value }}</td>
      <td>{{ $t->deleted_at ? 'yes' : 'no' }}</td>
      <td>
        <a href="{{ route('tenants.edit',$t) }}">Edit</a>
        @if(!$t->deleted_at)
          <form method="POST" action="{{ route('tenants.destroy',$t) }}" style="display:inline;">
            @csrf @method('DELETE')
            <button>Delete</button>
          </form>
        @else
          <form method="POST" action="{{ route('tenants.restore',$t->id) }}" style="display:inline;">
            @csrf
            <button>Restore</button>
          </form>
        @endif
      </td>
    </tr>
  @endforeach
  </tbody>
</table>
{{ $tenants->links() }}
@endsection
```

**`resources/views/tenants/create.blade.php` & `edit.blade.php`** — simple forms with fields: `name`, `subdomain`, `hex_color`, `status`.

---

## 18) CORS config (restrict to Barn Base)

**`app/Http/Middleware/Cors.php`** (or use `fruitcake/laravel-cors` config)

```php
// Prefer config/cors.php in Laravel 11
return [
  'paths' => ['api/*'],
  'allowed_origins' => explode(',', env('CORS_ALLOWED_ORIGINS','http://*.barnbase.test,https://*.barnbase.com')),
  'allowed_methods' => ['*'],
  'allowed_headers' => ['*'],
  'max_age' => 3600,
];
```

---

## 19) Notes

* Keep portal non-public for v0 (local/dev only, or HTTP Basic).
* API rate-limited, validated, and CORS‑restricted.
* `version` is provided by Manager; Application uses it to version `theme.css`.

---

**End of Manager Starter Pack**
