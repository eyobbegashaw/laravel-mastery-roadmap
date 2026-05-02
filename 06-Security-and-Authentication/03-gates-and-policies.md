### Detailed Conceptual Overview (70%)

Authentication answers "Who are you?" Authorization answers "What can you do?" Gates and policies provide the authorization layer in Laravel, determining whether authenticated users can perform specific actions on resources. While authentication verifies identity, authorization enforces permissions.

**Gates: Action-Based Authorization**

Gates define authorization for actions not tied to specific models. A gate answers questions like "Can this user view the admin dashboard?" or "Can this user export reports?" Gates are closures or class methods registered in `AppServiceProvider` or a dedicated `AuthServiceProvider`.

Gates receive the authenticated user and optional parameters. The gate closure returns `true` (authorized) or `false` (denied). If a closure returns a response object, that response is sent to the user (useful for custom error messages). Gates are called with `Gate::allows('export-reports')` or checked in Blade with `@can('export-reports')`.

Before hooks run before any gate check, allowing super-admin users to bypass all authorization. After hooks run after a gate denies access, providing an opportunity to override denials. These hooks are powerful but should be used sparingly to avoid confusing authorization logic.

**Policies: Resource-Based Authorization**

Policies organize authorization logic around a model or resource. A `PostPolicy` contains methods for each action: `view`, `create`, `update`, `delete`, `restore`, `forceDelete`. Each method receives the authenticated user and the model instance being acted upon.

Policies are discovered automatically when they follow Laravel's naming conventions—a `PostPolicy` for a `Post` model. Custom registration is available for non-standard naming. Policy methods map to controller actions, enabling concise controller code: `$this->authorize('update', $post)`.

Policy auto-discovery works by convention. Placing `PostPolicy` in `app/Policies` and naming methods `viewAny`, `view`, `create`, `update`, `delete` automatically links to `Post` model authorization checks. The `viewAny` method handles authorization for listing all posts; `view` handles viewing a specific post.

**Using Gates and Policies Together**

Gates handle global, non-model actions. Policies handle model-specific actions. This separation keeps authorization logic organized. A gate controls access to an admin dashboard; a policy controls who can edit a specific blog post.

Both gates and policies are checked the same way: through the `Gate` facade, the `$this->authorize()` method in controllers, and `@can` directives in Blade. This uniform interface means you don't need to know whether a check uses a gate or policy—the syntax is identical.

**Guest Authorization**

Authorization checks for unauthenticated users require special handling. Gates and policies receive `null` for the user parameter when checking unauthenticated access. Define optional user parameters and handle the null case explicitly. The `@guest` Blade directive checks authentication state separately from authorization.

### Production-Ready Code Snippets (30%)

**1. Comprehensive Policy Example**
```php
<?php
// app/Policies/PostPolicy.php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;
use Illuminate\Auth\Access\Response;

class PostPolicy
{
    use HandlesAuthorization;

    /**
     * Determine if the user can view any posts.
     * Called when listing all posts.
     */
    public function viewAny(User $user): bool
    {
        // All authenticated users can view posts
        return true;
    }

    /**
     * Determine if the user can view a specific post.
     */
    public function view(?User $user, Post $post): bool
    {
        // Published posts are visible to everyone (even guests)
        if ($post->is_published) {
            return true;
        }

        // Draft posts are only visible to their authors
        return $user && $post->author_id === $user->id;
    }

    /**
     * Determine if the user can create posts.
     */
    public function create(User $user): Response
    {
        // Check if user has reached their post limit
        $postCount = $user->posts()->where('created_at', '>=', now()->startOfDay())->count();
        
        if ($postCount >= config('app.daily_post_limit', 10)) {
            return Response::deny('You have reached your daily post limit.');
        }

        return $user->hasVerifiedEmail()
            ? Response::allow()
            : Response::deny('You must verify your email address before creating posts.');
    }

    /**
     * Determine if the user can update the post.
     */
    public function update(User $user, Post $post): Response
    {
        // Admin can always update
        if ($user->hasRole('admin')) {
            return Response::allow();
        }

        // Author can update their own posts within 24 hours
        if ($post->author_id === $user->id) {
            if ($post->created_at->diffInHours(now()) <= 24) {
                return Response::allow();
            }
            return Response::deny('Posts can only be edited within 24 hours of creation.');
        }

        return Response::deny('You can only edit your own posts.');
    }

    /**
     * Determine if the user can delete the post.
     */
    public function delete(User $user, Post $post): bool
    {
        // Only allow deletion if post has no published comments
        if ($post->comments()->published()->exists()) {
            return false;
        }

        return $user->hasRole('admin') || $post->author_id === $user->id;
    }

    /**
     * Determine if the user can restore a soft-deleted post.
     */
    public function restore(User $user, Post $post): bool
    {
        return $user->hasRole('admin');
    }

    /**
     * Determine if the user can permanently delete a post.
     */
    public function forceDelete(User $user, Post $post): bool
    {
        return $user->hasRole('super-admin');
    }

    /**
     * Determine if the user can publish the post.
     */
    public function publish(User $user, Post $post): bool
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }
}
```

**2. Gate Definitions and Usage**
```php
<?php
// app/Providers/AuthServiceProvider.php
namespace App\Providers;

use App\Models\User;
use App\Models\Post;
use App\Policies\PostPolicy;
use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The model to policy mappings.
     */
    protected $policies = [
        Post::class => PostPolicy::class,
        // Comment::class => CommentPolicy::class,
    ];

    /**
     * Register any authentication / authorization services.
     */
    public function boot(): void
    {
        $this->registerPolicies();

        // Gate: Access admin dashboard
        Gate::define('access-admin', function (User $user) {
            return $user->hasAnyRole(['admin', 'super-admin']);
        });

        // Gate: Export reports (with logging)
        Gate::define('export-reports', function (User $user, string $reportType) {
            $hasAccess = $user->hasPermission('export_reports');
            
            if ($hasAccess && $reportType === 'financial') {
                $hasAccess = $user->hasPermission('export_financial_reports');
            }
            
            if (!$hasAccess) {
                \Log::warning('Unauthorized export attempt', [
                    'user_id' => $user->id,
                    'report_type' => $reportType,
                ]);
            }
            
            return $hasAccess;
        });

        // Gate: Manage users (with custom response)
        Gate::define('manage-users', function (User $user, User $targetUser = null) {
            if ($user->isSuperAdmin()) {
                return true;
            }

            if ($targetUser && $targetUser->isSuperAdmin()) {
                return false; // No one can manage super admins
            }

            return $user->hasRole('admin');
        });

        // Gate: Feature access based on subscription
        Gate::define('access-premium-feature', function (User $user) {
            return $user->onTrial() || $user->subscribed('premium');
        });

        // Before hook: Super admin bypasses all authorization
        Gate::before(function (User $user) {
            if ($user->hasRole('super-admin')) {
                return true;
            }
        });

        // After hook: Log denied authorization attempts
        Gate::after(function (User $user, string $ability, $result, array $arguments) {
            if (!$result) {
                \Log::debug('Authorization denied', [
                    'user' => $user->id,
                    'ability' => $ability,
                    'arguments' => json_encode($arguments),
                ]);
            }
        });
    }
}
```

**3. Controller Authorization Usage**
```php
<?php
// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    /**
     * Display a listing of posts with authorization.
     */
    public function index()
    {
        // Authorize the viewAny action
        Gate::authorize('viewAny', Post::class);

        $posts = Post::with('author')
            ->when(!auth()->user()?->isAdmin(), function ($query) {
                $query->published();
            })
            ->paginate(15);

        return view('posts.index', compact('posts'));
    }

    /**
     * Show the form for creating a post.
     */
    public function create()
    {
        // Using authorize helper (throws AuthorizationException)
        $this->authorize('create', Post::class);

        return view('posts.create');
    }

    /**
     * Store a newly created post.
     */
    public function store(Request $request)
    {
        $this->authorize('create', Post::class);

        $post = $request->user()->posts()->create($request->validated());

        return redirect()->route('posts.show', $post);
    }

    /**
     * Display the specified post.
     */
    public function show(Post $post)
    {
        // Guest users can view published posts
        if (Gate::denies('view', $post)) {
            abort(404); // Return 404 instead of 403 to hide existence
        }

        return view('posts.show', compact('post'));
    }

    /**
     * Update the specified post.
     */
    public function update(Request $request, Post $post)
    {
        // Using Gate::authorize with custom error message capability
        Gate::authorize('update', $post);

        $post->update($request->validated());

        return redirect()->route('posts.show', $post);
    }

    /**
     * Remove the specified post.
     */
    public function destroy(Post $post)
    {
        if (Gate::denies('delete', $post)) {
            return back()->with('error', 'You cannot delete this post.');
        }

        $post->delete();

        return redirect()->route('posts.index')
            ->with('success', 'Post deleted.');
    }

    /**
     * Custom action: Publish a post.
     */
    public function publish(Post $post)
    {
        $this->authorize('publish', $post);

        $post->update(['is_published' => true, 'published_at' => now()]);

        return back()->with('success', 'Post published!');
    }

    /**
     * API: Delete multiple posts (bulk action).
     */
    public function bulkDelete(Request $request)
    {
        $postIds = $request->input('post_ids', []);
        
        $posts = Post::whereIn('id', $postIds)->get();
        
        // Authorize each post individually
        foreach ($posts as $post) {
            $this->authorize('delete', $post);
        }

        Post::whereIn('id', $postIds)->delete();

        return response()->json(['message' => 'Posts deleted successfully.']);
    }
}
```

**4. Blade Authorization Directives**
```blade
{{-- resources/views/posts/show.blade.php --}}
@extends('layouts.app')

@section('content')
<div class="max-w-4xl mx-auto">
    <article>
        <h1>{{ $post->title }}</h1>
        <div>{{ $post->body }}</div>
    </article>

    {{-- Authorization-based actions --}}
    <div class="mt-6 flex gap-3">
        {{-- Check using Gate --}}
        @can('update', $post)
            <a href="{{ route('posts.edit', $post) }}" class="btn-primary">Edit</a>
        @endcan

        @can('delete', $post)
            <form action="{{ route('posts.destroy', $post) }}" method="POST">
                @csrf @method('DELETE')
                <button type="submit" class="btn-danger">Delete</button>
            </form>
        @endcan

        {{-- Check using policy method --}}
        @can('publish', $post)
            <form action="{{ route('posts.publish', $post) }}" method="POST">
                @csrf
                <button type="submit" class="btn-success">Publish</button>
            </form>
        @endcan

        {{-- Cannot directive --}}
        @cannot('update', $post)
            <p class="text-sm text-gray-500">You cannot edit this post.</p>
        @endcannot
    </div>

    {{-- Conditional content based on ability --}}
    @can('access-admin')
        <div class="mt-4 p-4 bg-gray-100 rounded">
            <h4>Admin Information</h4>
            <p>Post ID: {{ $post->id }}</p>
            <p>Author ID: {{ $post->author_id }}</p>
        </div>
    @endcan

    {{-- Any or all abilities check --}}
    @canany(['update', 'delete'], $post)
        <div class="mt-4">
            <h4>Moderation Tools</h4>
            {{-- Moderation actions --}}
        </div>
    @endcanany

    {{-- Guest check --}}
    @guest
        <div class="mt-4">
            <a href="{{ route('login') }}">Login to interact with this post</a>
        </div>
    @endguest
</div>
@endsection
```

**Best Practices (Staff Engineer Tips)**

1. **Policy Auto-Discovery**: Name policies `{Model}Policy` in the `app/Policies` directory. Define policy methods matching the standard CRUD actions. This enables auto-discovery, reducing registration boilerplate. Only manually register policies with non-standard naming.

2. **Authorization in Form Requests**: Move authorization checks to Form Request classes' `authorize()` method. This keeps controllers clean and ensures authorization runs before validation. Return descriptive denial messages from the `authorize()` method.

3. **Granular Permissions Over Broad Roles**: Prefer specific abilities like `publish-post` over role-based checks like `is-editor`. Gates and policies check abilities, not roles. Assign abilities to roles in a separate configuration or database layer. This enables flexible permission assignment without code changes.

4. **Caching Authorization Results**: Authorization checks on every request can be expensive. Cache role and permission assignments with configurable TTL. Invalidate cache when permissions change. Use Laravel's built-in cache for Gate and Policy results.

**Common Pitfalls and Solutions**

1. **Assuming Authentication Implies Authorization**: Being logged in doesn't mean the user can do everything. Always check authorization separately from authentication. A regular user can view their own posts but not others' drafts. Missing authorization checks create privacy breaches.

2. **Returning 403 vs 404**: When a user lacks permission to view a resource, decide between 403 (Forbidden) and 404 (Not Found). 404 hides the resource's existence from unauthorized users. Use 403 when the user knows the resource exists but can't access it. Be consistent within your application.

3. **Not Checking Authorization in Collections**: Listing resources requires `viewAny` authorization. Building dropdown lists with all users exposes data to unauthorized users. Filter collections based on what the current user is authorized to see. Use model scopes that respect authorization.

4. **Guest Authorization Neglect**: Null user references in policies cause TypeErrors if the parameter isn't marked nullable. Always handle the unauthenticated case explicitly in policies. Use `?User` type hints for methods that guests might call.

-