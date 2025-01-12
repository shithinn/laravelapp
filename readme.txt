1. **Install Laravel Project**

```bash
composer create-project laravel/laravel laravel-role-management
cd laravel-role-management
```

2. **Set up Authentication**

```bash
composer require laravel/ui
php artisan ui vue --auth
npm install && npm run dev
php artisan migrate
```

3. **Create Models and Migrations**

```bash
php artisan make:model Role -m
php artisan make:model Route -m
php artisan make:model UserRole -m
php artisan make:model RoleRoute -m
```

Edit migrations accordingly:

- **Roles Migration**

```php
Schema::create('roles', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->timestamps();
});
```

- **Routes Migration**

```php
Schema::create('routes', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('uri');
    $table->timestamps();
});
```

- **UserRoles Migration**

```php
Schema::create('user_roles', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->foreignId('role_id')->constrained()->onDelete('cascade');
    $table->timestamps();
});
```

- **RoleRoutes Migration**

```php
Schema::create('role_routes', function (Blueprint $table) {
    $table->id();
    $table->foreignId('role_id')->constrained()->onDelete('cascade');
    $table->foreignId('route_id')->constrained()->onDelete('cascade');
    $table->timestamps();
});
```

Run migrations:

```bash
php artisan migrate
```

4. **Define Relationships in Models**

- **Role.php**

```php
public function users()
{
    return $this->belongsToMany(User::class, 'user_roles');
}

public function routes()
{
    return $this->belongsToMany(Route::class, 'role_routes');
}
```

- **Route.php**

```php
public function roles()
{
    return $this->belongsToMany(Role::class, 'role_routes');
}
```

- **User.php**

```php
public function roles()
{
    return $this->belongsToMany(Role::class, 'user_roles');
}
```

5. **Middleware for Role-Based Access**

```bash
php artisan make:middleware RoleMiddleware
```

- **RoleMiddleware.php**

```php
public function handle($request, Closure $next, $role)
{
    $user = Auth::user();
    $route = $request->route()->getName();
    
    if (!$user || !$user->roles->pluck('id')->contains($role)) {
        abort(403);
    }

    $allowedRoutes = $user->roles->flatMap(function($role) {
        return $role->routes;
    })->pluck('uri')->toArray();

    if (!in_array($route, $allowedRoutes)) {
        abort(403);
    }

    return $next($request);
}
```

Register middleware in `Kernel.php`:

```php
protected $routeMiddleware = [
    // ...
    'role' => \App\Http\Middleware\RoleMiddleware::class,
];
```

6. **Routes and Controllers**

- **web.php**

```php
use App\Http\Controllers\AdminController;
use App\Http\Controllers\UserController;
use App\Http\Controllers\RouteController;

Route::group(['middleware' => 'auth'], function() {
    Route::get('/admin', [AdminController::class, 'index'])->name('admin.dashboard')->middleware('role:admin');
    Route::resource('users', UserController::class)->middleware('role:admin');
    Route::get('/editor', [RouteController::class, 'index'])->name('editor.dashboard')->middleware('role:editor');
    Route::get('/student', [RouteController::class, 'show'])->name('student.dashboard')->middleware('role:student');
});
```

- **AdminController.php**

```php
public function index()
{
    return view('admin.dashboard');
}
```

- **UserController.php**

```php
public function index()
{
    $users = User::all();
    return view('admin.users.index', compact('users'));
}

public function edit(User $user)
{
    $roles = Role::all();
    return view('admin.users.edit', compact('user', 'roles'));
}

public function update(Request $request, User $user)
{
    $user->roles()->sync($request->roles);
    return redirect()->route('users.index');
}
```

7. **Role Management**

- **RouteController.php**

```php
public function index()
{
    $routes = Route::all();
    return view('admin.routes.index', compact('routes'));
}

public function show()
{
    return view('student.dashboard');
}
```

- **RoleController.php**

```php
public function index()
{
    $roles = Role::with('routes')->get();
    return view('admin.roles.index', compact('roles'));
}

public function edit(Role $role)
{
    $routes = Route::all();
    return view('admin.roles.edit', compact('role', 'routes'));
}

public function update(Request $request, Role $role)
{
    $role->routes()->sync($request->routes);
    return redirect()->route('roles.index');
}
```

This complete code setup allows for role-based access control in a Laravel project, where the admin can manage users, roles, and assign specific route permissions.
