### Detailed Conceptual Overview (70%)

Route parameters transform static URLs into dynamic endpoints, while model binding bridges the gap between URL segments and database records. This seemingly simple feature contains sophisticated functionality that, when properly understood, eliminates boilerplate controller code.

**Route Parameter Fundamentals**

Route parameters are defined using curly braces in the route URI: `/posts/{post}` captures the segment after `/posts/` as a parameter named `post`. Required parameters must receive a value; optional parameters use a question mark: `/posts/{post?}` and require a default value in the handler. Regular expression constraints validate parameter formats: `/posts/{id}` where `id` must be `[0-9]+`.

Laravel extracts parameters and passes them to the route handler in order. A controller method receiving `(Request $request, $postId)` gets the parameter value directly. But with model binding, the framework goes further—it queries the database and injects the actual model instance.

**Implicit Route Model Binding**

Implicit binding is Laravel's most elegant feature for clean controllers. When a route parameter name matches an Eloquent model's route key, Laravel automatically performs a database lookup. The parameter `{user}` triggers a `User::find($value)` query. The parameter `{post:slug}` triggers `Post::where('slug', $value)->first()`.

The magic happens in the `SubstituteBindings` middleware, which runs in both web and API middleware groups. After route matching but before controller execution, this middleware examines each parameter, recognizes Eloquent model type hints in the controller method, resolves the model, and injects it into the method call.

If the model isn't found, Laravel throws a `ModelNotFoundException`, which the exception handler converts to a 404 response automatically. You never write queries for primary record loading in controllers—the framework handles it transparently.

**Custom Route Model Binding**

Sometimes implicit binding isn't sufficient. You might need to find soft-deleted models, apply global scopes, or use custom resolution logic. Custom binding is defined in the `boot` method of `AppServiceProvider` using the `Route::bind()` method or by overriding the `resolveRouteBinding()` method on the Eloquent model.

Custom binding resolvers give complete control. You can query only active records, include relationships eagerly, check permissions before resolving, or throw custom exceptions. The resolver receives the value from the URL segment and returns the resolved model (or null/exception for not found).

**Explicit Route Model Binding**

Explicit binding is the older, more verbose approach where you register bindings in `RouteServiceProvider`. This method is less common now that implicit binding handles most cases, but it remains useful for binding non-model parameters or implementing complex resolution that should be available across multiple routes.

**Multiple Route Parameters and Nested Resources**

Routes often need multiple parameters representing relationships: `/users/{user}/posts/{post}`. Laravel handles multiple bindings efficiently. The first parameter resolves a User, the second resolves a Post. Scoped binding goes further by ensuring the Post belongs to the resolved User: `Route::get('/users/{user}/posts/{post}')->scopeBindings()`.

Scoped binding prevents users from accessing other users' posts by manipulating URLs. If user 1 requests `/users/2/posts/5`, the binding checks that post 5 belongs to user 2, not just that both records exist independently.

### Production-Ready Code Snippets (30%)

**1. Advanced Route Parameter Configuration**
```php
<?php
// routes/web.php
use App\Models\Article;
use Illuminate\Support\Facades\Route;

// Basic required parameter with constraint
Route::get('/users/{id}', function (string $id) {
    // $id is '123' - parameter name matches variable
})->where('id', '[0-9]+')->whereNumber('id');

// Optional parameter with default
Route::get('/users/{user?}', function (?User $user = null) {
    // $user is User model or null if not provided
    return $user 
        ? view('users.show', ['user' => $user])
        : view('users.index');
});

// Explicit binding key for slug-based URLs
Route::get('/posts/{post:slug}', [PostController::class, 'show'])
    ->name('posts.show')
    ->where('post', '[a-z0-9-]+'); // Only lowercase slugs

// Multiple parameters with scoped bindings
Route::get('/teams/{team}/projects/{project:uuid}', function (Team $team, Project $project) {
    // $project automatically scoped to $team (if scopeBindings enabled)
    return view('projects.show', compact('team', 'project'));
})->scopeBindings();

// Route with custom binding for soft-deleted models
Route::get('/archived-posts/{post}', function (Post $post) {
    return view('posts.archived', ['post' => $post]);
})->withTrashed();
```

**2. Custom Route Model Binding in Service Provider**
```php
<?php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Models\Invoice;
use App\Models\Article;
use Illuminate\Support\Facades\Route;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Custom binding: find published articles by slug with eager loading
        Route::bind('article', function (string $value) {
            return Article::where('slug', $value)
                ->where('status', 'published')
                ->with(['author', 'category', 'tags'])
                ->firstOrFail();
        });

        // Custom binding: find invoice by reference number for authenticated user
        Route::bind('invoice', function (string $value) {
            $invoice = Invoice::where('reference', $value)
                ->where('user_id', auth()->id())
                ->first();

            if (! $invoice) {
                throw new \Illuminate\Database\Eloquent\ModelNotFoundException(
                    "Invoice {$value} not found or access denied."
                );
            }

            return $invoice;
        });

        // Custom binding: resolve user by username or email
        Route::bind('identifier', function (string $value) {
            return \App\Models\User::where('email', $value)
                ->orWhere('username', $value)
                ->firstOrFail();
        });
    }
}
```

**3. Model-Level Binding Customization**
```php
<?php
// app/Models/Post.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    /**
     * Override the default route binding resolution.
     * Allows finding by different keys based on request context.
     */
    public function resolveRouteBinding($value, $field = null)
    {
        // If field is explicitly set (like {post:slug}), use it
        if ($field) {
            return $this->where($field, $value)->firstOrFail();
        }

        // Check if value looks like a UUID (36 characters with dashes)
        if (preg_match('/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i', $value)) {
            return $this->where('uuid', $value)->firstOrFail();
        }

        // Fall back to slug lookup
        return $this->where('slug', $value)
            ->where('is_published', true)
            ->firstOrFail();
    }

    /**
     * Resolve child route binding for nested resources.
     * Ensures child models belong to the parent in nested routes.
     */
    public function resolveChildRouteBinding($childType, $value, $field)
    {
        // Only allow access to approved comments in nested routes
        if ($childType === 'comments') {
            return parent::resolveChildRouteBinding($childType, $value, $field)
                ->where('is_approved', true);
        }

        return parent::resolveChildRouteBinding($childType, $value, $field);
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Always Use Explicit Route Keys for Public URLs**: Never expose auto-incrementing IDs in URLs meant for users. Use slugs or UUIDs. Sequential IDs reveal record counts, enabling competitors to estimate your data size and attackers to iterate through records. Models should define `getRouteKeyName()` returning the public identifier.

2. **Eager Load in Custom Bindings**: Route model binding executes a separate query per model. For nested routes showing a user and their posts, this means two queries. If the controller always needs related data, eager load it in the custom binding resolver. This prevents N+1 queries when the view iterates over relationships.

3. **Scope Bindings for Multi-Tenant Applications**: In SaaS applications with teams or organizations, always use scope bindings. A URL like `/teams/5/projects/10` should verify project 10 belongs to team 5. Enable scope bindings globally in `AppServiceProvider` with `Route::scopeBindings()`.

4. **Cache Route Bindings**: For frequently accessed models with stable data, use Laravel's cache in custom bindings. Check cache before database query. Use cache tags for invalidation when models update. This can reduce database load by 80% for popular resources.

**Common Pitfalls and Solutions**

1. **SQL Injection via Custom Bindings**: Custom binding resolvers using raw SQL or uncontrolled input concatenation create injection vulnerabilities. Always use Eloquent's parameterized queries or the query builder. Never concatenate URL values into raw SQL strings.

2. **Missing 404 Handling**: Implicit binding automatically converts `ModelNotFoundException` to 404 responses. Custom bindings must throw `ModelNotFoundException` (not generic exceptions) to maintain this behavior. Custom exceptions require manual handler configuration.

3. **Nested Route Parameter Order Confusion**: The order of parameters in nested routes matters. `/users/{user}/posts/{post}` expects `User` model first, then `Post`. Swapping the order in the controller method signature causes binding failures. Always match controller method parameter order to route definition order.

4. **Soft-Deleted Model Binding**: Routes without explicit soft-delete handling return 404 for soft-deleted records. Users who soft-delete content expect to see "archived" states, not 404 errors. Add a separate route with `withTrashed()` for archived content access.

---
