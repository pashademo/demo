[README.md](https://github.com/user-attachments/files/29037137/README.md)
# Демо-экзамен КОД 09.02.07-5-2026

## Содержание
- [Установка Laravel](#установка-laravel)
- [База данных](#база-данных)
- [Миграции](#миграции)
- [Модели](#модели)
- [Контроллеры](#контроллеры)
- [Маршруты](#маршруты)
- [Views (Blade)](#views-blade)
- [ 6 модуль ](#6-модуль)

---

## Установка Laravel

```bash
composer create-project laravel/laravel --prefer-dist .
```

---

## База данных

```sql
CREATE TABLE customers (
    id   INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE products (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(255)  NOT NULL,
    unit  VARCHAR(50)   NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

CREATE TABLE materials (
    id    INT AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(255)  NOT NULL,
    unit  VARCHAR(50)   NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

CREATE TABLE specifications (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT          NOT NULL,
    name       VARCHAR(255) NOT NULL,
    FOREIGN KEY (product_id) REFERENCES products(id)
);

CREATE TABLE specification_materials (
    id               INT AUTO_INCREMENT PRIMARY KEY,
    specification_id INT            NOT NULL,
    material_id      INT            NOT NULL,
    quantity         DECIMAL(10,3)  NOT NULL,
    FOREIGN KEY (specification_id) REFERENCES specifications(id),
    FOREIGN KEY (material_id)      REFERENCES materials(id)
);

CREATE TABLE orders (
    id           INT AUTO_INCREMENT PRIMARY KEY,
    customer_id  INT         NOT NULL,
    order_number VARCHAR(50) NOT NULL,
    order_date   DATE        NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE TABLE order_items (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    order_id   INT           NOT NULL,
    product_id INT           NOT NULL,
    quantity   INT           NOT NULL,
    price      DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id)   REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

---

## Миграции

### create_users_table

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->boolean('need_to_change_password')->default(true);
            $table->unsignedTinyInteger('wrong_counter')->default(0);
            $table->boolean('is_admin')->default(false);
            $table->timestamps();
        });

        Schema::create('password_reset_tokens', function (Blueprint $table) {
            $table->string('email')->primary();
            $table->string('token');
            $table->timestamp('created_at')->nullable();
        });

        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
        Schema::dropIfExists('password_reset_tokens');
        Schema::dropIfExists('sessions');
    }
};
```

---

## Модели

### User.php

```php
<?php

namespace App\Models;

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
        'need_to_change_password',
        'wrong_counter',
        'is_admin',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected function casts(): array
    {
        return [
            'email_verified_at'        => 'datetime',
            'password'                 => 'hashed',
            'need_to_change_password'  => 'boolean',
        ];
    }
}
```

---

## Контроллеры

### Создание контроллеров

```bash
php artisan make:controller ActionsController
php artisan make:controller ViewsController
```

### ActionsController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class ActionsController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'email'    => 'required|email',
            'password' => 'required|string|min:6',
        ]);

        if (Auth::attempt($request->only(['email', 'password']))) {
            $user = Auth::user();

            if ($user->wrong_counter === 3) {
                Auth::logout();
                return redirect()
                    ->route('login')
                    ->withErrors(['email' => 'Your account is locked. Please contact support.']);
            }

            if ($user->need_to_change_password) {
                return redirect()
                    ->route('change_password')
                    ->with('status', 'You need to change your password before proceeding.');
            }

            return redirect()->route('dashboard');
        }

        if ($user = User::firstWhere('email', $request->input('email'))) {
            if ($user->wrong_counter != 3) {
                $user->wrong_counter++;
                $user->save();
            }

            if ($user->wrong_counter === 3) {
                return redirect()
                    ->route('login')
                    ->withErrors(['email' => 'Your account is locked. Please contact support.']);
            }
        }

        return redirect()
            ->route('login')
            ->withErrors(['email' => 'Invalid credentials. Please try again.'])
            ->withInput($request->only('email'));
    }

    public function changePassword(Request $request)
    {
        $request->validate([
            'current_password' => 'required|string|min:6|current_password',
            'new_password'     => 'required|string|min:6|confirmed:new_password_repeat',
        ]);

        $user = Auth::user();
        $user->password = $request->input('new_password');
        $user->need_to_change_password = false;
        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'Password changed successfully.');
    }

    public function editUser(User $user, Request $request)
    {
        $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => "required|email|unique:users,email,{$user->id}",
            'password' => 'nullable|string|min:6|confirmed',
        ]);

        $user->name  = $request->input('name');
        $user->email = $request->input('email');

        if ($request->filled('password')) {
            $user->password = $request->input('password');
        }

        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'User updated successfully.');
    }

    public function createUser(Request $request)
    {
        $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => 'required|email|unique:users,email',
            'password' => 'required|string|min:6|confirmed',
        ]);

        $user = new User();
        $user->name     = $request->input('name');
        $user->email    = $request->input('email');
        $user->password = $request->input('password');
        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'User created successfully.');
    }

    public function deleteUser(User $user)
    {
        if ($user->id === Auth::id()) {
            return redirect()
                ->route('dashboard')
                ->withErrors(['user' => 'You cannot delete your own account.']);
        }

        $user->delete();

        return redirect()
            ->route('dashboard')
            ->with('status', 'User deleted successfully.');
    }

    public function logout(Request $request)
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect()->route('login');
    }

    public function unlockUser(User $user)
    {
        if ($user->wrong_counter !== 3) {
            $user->wrong_counter = 3;
            $user->save();

            return redirect()
                ->route('dashboard')
                ->with('status', 'User locked successfully.');
        }

        $user->wrong_counter = 0;
        $user->save();

        return redirect()
            ->route('dashboard')
            ->with('status', 'User unlocked successfully.');
    }
}
```

### ViewsController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class ViewsController extends Controller
{
    public function index()
    {
        return view('index', [
            'users' => User::all(),
        ]);
    }

    public function login()
    {
        return view('login');
    }

    public function changePassword()
    {
        return view('change_password');
    }

    public function editUser(?User $user = null)
    {
        return view('user_form', [
            'user' => $user,
        ]);
    }
}
```

---

## Маршруты

### web.php

```php
<?php

use Illuminate\Support\Facades\Route;

Route::get('/', [App\Http\Controllers\ViewsController::class, 'index'])
    ->name('dashboard')
    ->middleware(['auth']);

Route::get('/login', [App\Http\Controllers\ViewsController::class, 'login'])
    ->name('login')
    ->middleware(['guest']);

Route::post('/login', [App\Http\Controllers\ActionsController::class, 'login'])
    ->middleware(['guest']);

Route::post('/logout', [App\Http\Controllers\ActionsController::class, 'logout'])
    ->name('logout')
    ->middleware(['auth']);

Route::get('/login/change-password', [App\Http\Controllers\ViewsController::class, 'changePassword'])
    ->name('change_password')
    ->middleware(['auth']);

Route::post('/login/change-password', [App\Http\Controllers\ActionsController::class, 'changePassword'])
    ->name('login.change-password')
    ->middleware(['auth']);

Route::get('/user', [App\Http\Controllers\ViewsController::class, 'editUser'])
    ->name('user.create')
    ->middleware(['auth']);

Route::get('/user/{user}', [App\Http\Controllers\ViewsController::class, 'editUser'])
    ->name('user')
    ->middleware(['auth']);

Route::delete('/user/{user}', [App\Http\Controllers\ActionsController::class, 'deleteUser'])
    ->name('user.delete')
    ->middleware(['auth']);

Route::patch('/user/{user}', [App\Http\Controllers\ActionsController::class, 'unlockUser'])
    ->name('user.unlock')
    ->middleware(['auth']);

Route::post('/user', [App\Http\Controllers\ActionsController::class, 'createUser'])
    ->name('user.save')
    ->middleware(['auth']);

Route::put('/user/{user}', [App\Http\Controllers\ActionsController::class, 'editUser'])
    ->name('user.update')
    ->middleware(['auth']);
```

---

## Views (Blade)

### login.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login</title>
</head>
<body>
    <form action="{{ route('login') }}" method="POST">
        @csrf
        <div>
            <label for="email">Email:</label>
            <input type="email" id="email" name="email" value="{{ old('email') }}" required>
            @error('email')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" required>
            @error('password')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <button type="submit">Login</button>
    </form>
</body>
</html>
```

### change_password.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Change Password</title>
</head>
<body>
    <form action="{{ route('login.change-password') }}" method="POST">
        @csrf
        <div>
            <label for="current_password">Current Password:</label>
            <input type="password" id="current_password" name="current_password" required>
            @error('current_password')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <div>
            <label for="new_password">New Password:</label>
            <input type="password" id="new_password" name="new_password" required>
            @error('new_password')
                <div class="error">{{ $message }}</div>
            @enderror
        </div>
        <div>
            <label for="new_password_repeat">Repeat Password:</label>
            <input type="password" id="new_password_repeat" name="new_password_repeat" required>
        </div>
        <button type="submit">Change Password</button>
    </form>
</body>
</html>
```

### index.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dashboard</title>
</head>
<body>
    <h1>Welcome to Dashboard</h1>

    <form action="{{ route('logout') }}" method="POST" style="display:inline; float:right;">
        @csrf
        <button type="submit">Logout</button>
    </form>

    <h2>List of Users</h2>
    <table border="1">
        <thead>
            <tr>
                <th>Name</th>
                <th>Email</th>
                <th>Need to change password</th>
                <th>User is blocked</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach ($users as $user)
                <tr>
                    <td>{{ $user->name }}</td>
                    <td>{{ $user->email }}</td>
                    <td>{{ $user->need_to_change_password ? 'Yes' : 'No' }}</td>
                    <td>{{ $user->wrong_counter === 3 ? 'Yes' : 'No' }}</td>
                    <td>
                        <a href="{{ route('user', $user->id) }}">Edit</a>

                        <form action="{{ route('user.delete', $user->id) }}" method="POST" style="display:inline;">
                            @csrf
                            @method('DELETE')
                            <button type="submit">Delete</button>
                        </form>

                        <form action="{{ route('user.unlock', $user->id) }}" method="POST" style="display:inline;">
                            @csrf
                            @method('PATCH')
                            <button type="submit">
                                {{ $user->wrong_counter === 3 ? 'Unblock' : 'Block' }}
                            </button>
                        </form>
                    </td>
                </tr>
            @endforeach
        </tbody>
        <tfoot>
            <tr>
                <td colspan="5">
                    <a href="{{ route('user.create') }}">Create new user</a>
                </td>
            </tr>
        </tfoot>
    </table>

    @if (session('status'))
        <div>
            <strong>{{ session('status') }}</strong>
        </div>
    @endif
</body>
</html>
```

### user_form.blade.php

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ $user ? 'Edit User' : 'Create User' }}</title>
</head>
<body>
    <h1>{{ $user ? 'Edit User' : 'Create User' }}</h1>

    <form action="{{ $user ? route('user.update', $user->id) : route('user.save') }}" method="POST">
        @csrf
        @if ($user)
            @method('PUT')
        @endif

        <div>
            <label for="name">Name:</label>
            <input type="text" id="name" name="name" value="{{ $user->name ?? '' }}" required>
        </div>
        <div>
            <label for="email">Email:</label>
            <input type="email" id="email" name="email" value="{{ $user->email ?? '' }}" required>
        </div>
        <div>
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" {{ $user ? '' : 'required' }}>
        </div>
        <div>
            <label for="password_confirmation">Confirm Password:</label>
            <input type="password" id="password_confirmation" name="password_confirmation" {{ $user ? '' : 'required' }}>
        </div>

        <button type="submit">{{ $user ? 'Update' : 'Create' }}</button>
        <a href="{{ route('dashboard') }}">Cancel</a>
    </form>

    @if ($errors->any())
        <div>
            <h2>Errors:</h2>
            <ul>
                @foreach ($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
        </div>
    @endif
</body>
</html>
```

## 6 модуль

```
composer create-project laravel/laravel --prefer-dist .
```
### Контроллеры
```
php artisan make:controller TestCaseController
Удалить welcome.blade.php, создать index.blade.php
```
### TestCaseController.php:
```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class TestCaseController extends Controller
{
    public function show(Request $request)
    {
        return view('index');
    }

    public function getData()
    {
        $data = file_get_contents('http://localhost:8080/api/fullName');

        if ($data) {
            $data = json_decode($data)->value;
        }

        return redirect()->route('test-case')->with('value', $data);
    }

    public function checkData(Request $request)
    {
        $value = $request->input('value');

        if (preg_match('/^[А-Яа-яЁё]+ [А-Яа-яЁё]+ [А-Яа-яЁё]+$/u', $value)) {
            return redirect()->route('test-case')
                ->with('value', $value)
                ->with('message', 'ФИО корректно');
        } else {
            return redirect()->route('test-case')
                ->with('value', $value)
                ->with('message', 'ФИО содержит некорректные символы');
        }
    }
}
```

### index.blade.php:
```
<p>
    <form action="{{ route('test-case.get') }}">
        <button type="submit">Получить данные</button>
        <span>{{ session('value') }}</span>
    </form>
</p>

<p>
    <form method="POST" action="{{ route('test-case.check') }}">
        @csrf
        <input type="hidden" name="value" value="{{ session('value') }}">
        <button type="submit">Отправить результаты теста</button>
        <span>{{ session('message') }}</span>
    </form>
</p>
```
### web.php:
```
<?php

use App\Http\Controllers\TestCaseController;
use Illuminate\Support\Facades\Route;

Route::get('/', [TestCaseController::class, 'show'])
    ->name('test-case');

Route::get('/get', [TestCaseController::class, 'getData'])
    ->name('test-case.get');

Route::post('/check', [TestCaseController::class, 'checkData'])
    ->name('test-case.check');
```
