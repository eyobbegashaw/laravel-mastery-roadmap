### Detailed Conceptual Overview (70%)

Resource controllers standardize CRUD operations into predictable controller structures. Rather than inventing unique controller methods for each entity, Laravel provides conventions that make applications consistent and development faster. Understanding resource controllers means understanding RESTful design principles implemented in practice.

**RESTful Resource Pattern**

The resource pattern maps seven standard actions to HTTP verbs and URIs. Index lists all resources (GET /posts). Create shows a creation form (GET /posts/create). Store saves a new resource (POST /posts). Show displays a single resource (GET /posts/{post}). Edit shows an edit form (GET /posts/{post}/edit). Update modifies a resource (PUT/PATCH /posts/{post}). Destroy removes a resource (DELETE /posts/{post}).

This standardization means any Laravel developer can navigate any Laravel application. Knowing the convention, you immediately understand that `PhotoController@destroy` handles DELETE requests to `/photos/{photo}` and removes the photo resource.

**Partial Resource Controllers**

Not every entity needs all seven actions. A `CommentController` might only need `store`, `update`, and `destroy`—comments are created inline, not on separate pages. The `only()` method restricts routes: `Route::resource('comments', CommentController::class)->only(['store', 'update', 'destroy'])`. The `except()` method excludes unnecessary routes.

Using partial resources communicates intent. When a developer sees `->only(['index', 'show'])`, they immediately understand this is a read-only resource. No one will accidentally link to a non-existent edit page or waste time building a form that will never be used.

**API Resource Controllers**

API resources omit the `create` and `edit` actions since APIs don't serve HTML forms. The remaining five actions (index, store, show, update, destroy) handle all necessary API operations. `Route::apiResource()` generates these automatically without the form endpoints.

API controllers return JSON responses instead of views. The controller methods use API resources for transformation, respond with appropriate HTTP status codes, and follow strict REST conventions. API resources typically implement validation differently—they return 422 Unprocessable Entity with error details instead of redirecting with flashed validation errors.

**Nested Resource Controllers**

Resources often have hierarchical relationships. A post has comments, a user has posts, a team has projects. Nested resources reflect these relationships in URLs: `/posts/{post}/comments/{comment}`. The `Route::resource('posts.comments', CommentController::class)` generates nested routes automatically.

Nested controllers receive the parent model as a parameter. The `CommentController@store` method receives `(Request $request, Post $post)`. The controller can scope queries to the parent, ensuring users can only access comments belonging to the specified post.

**Shallow Nesting**

Deeply nested URLs become unwieldy: `/users/{user}/posts/{post}/comments/{comment}`. Shallow nesting keeps parent scoping for creation (needing the parent context) but uses flat URLs for existing records. `Route::resource('posts.comments', CommentController::class)->shallow()` generates `/posts/{post}/comments` for `store` but `/comments/{comment}` for `show`, `update`, and `destroy`.

This keeps URLs clean while maintaining proper scoping. The comment's ID uniquely identifies it globally—you don't need the post ID in the URL. But for creating a new comment, you need to know which post it belongs to.

**Singleton Resource Controllers**

Some resources have exactly one instance per user or context. A user's profile, a site's settings page, a shopping cart. `Route::singleton('profile', ProfileController::class)` generates routes without ID parameters: `/profile` (show), `/profile/edit` (edit), `PATCH /profile` (update). The controller always operates on the current user's profile.

Singletons can be creatable (`->creatable()`) when the resource might not exist yet, adding `create` and `store` routes. They can be destroyable (`->destroyable()`) when the resource can be removed, adding a `destroy` route.

### Production-Ready Code Snippets (30%)

**1. Complete Resource Controller with Form Requests**
```php
<?php
// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Models\Post;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;
use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    /**
     * Display a listing of the resource.
     */
    public function index(): View
    {
        // Authorize using policy
        Gate::authorize('viewAny', Post::class);

        $posts = Post::with('author')
            ->latest()
            ->paginate(15);

        return view('posts.index', compact('posts'));
    }

    /**
     * Show the form for creating a new resource.
     */
    public function create(): View
    {
        Gate::authorize('create', Post::class);

        return view('posts.create');
    }

    /**
     * Store a newly created resource in storage.
     */
    public function store(StorePostRequest $request): RedirectResponse
    {
        Gate::authorize('create', Post::class);

        $post = $request->user()->posts()->create($request->validated());

        // Attach tags if provided
        if ($request->has('tags')) {
            $post->tags()->sync($request->input('tags'));
        }

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'Post created successfully.');
    }

    /**
     * Display the specified resource.
     */
    public function show(Post $post): View
    {
        Gate::authorize('view', $post);

        $post->load(['author', 'comments' => fn ($query) => 
            $query->approved()->latest()->limit(10)
        ]);

        return view('posts.show', compact('post'));
    }

    /**
     * Show the form for editing the specified resource.
     */
    public function edit(Post $post): View
    {
        Gate::authorize('update', $post);

        return view('posts.edit', compact('post'));
    }

    /**
     * Update the specified resource in storage.
     */
    public function update(UpdatePostRequest $request, Post $post): RedirectResponse
    {
        Gate::authorize('update', $post);

        $post->update($request->validated());

        return redirect()
            ->route('posts.edit', $post)
            ->with('success', 'Post updated successfully.');
    }

    /**
     * Remove the specified resource from storage.
     */
    public function destroy(Post $post): RedirectResponse
    {
        Gate::authorize('delete', $post);

        $post->delete();

        return redirect()
            ->route('posts.index')
            ->with('success', 'Post deleted successfully.');
    }
}
```

**2. API Resource Controller with Form Requests**
```php
<?php
// app/Http/Controllers/Api/V1/PostController.php
namespace App\Http\Controllers\Api\V1;

use App\Models\Post;
use App\Http\Controllers\Controller;
use App\Http\Resources\PostResource;
use App\Http\Resources\PostCollection;
use App\Http\Requests\Api\StorePostRequest;
use App\Http\Requests\Api\UpdatePostRequest;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Response;

class PostController extends Controller
{
    /**
     * Display a listing of the resource.
     */
    public function index(): PostCollection
    {
        $posts = Post::with('author')
            ->latest()
            ->paginate(20);

        return new PostCollection($posts);
    }

    /**
     * Store a newly created resource in storage.
     */
    public function store(StorePostRequest $request): JsonResponse
    {
        $post = $request->user()->posts()->create($request->validated());

        return (new PostResource($post->load('author')))
            ->response()
            ->setStatusCode(Response::HTTP_CREATED);
    }

    /**
     * Display the specified resource.
     */
    public function show(Post $post): PostResource
    {
        // Ensure the post is published or user owns it
        abort_if(
            !$post->is_published && $post->user_id !== auth()->id(),
            404
        );

        return new PostResource($post->load(['author', 'comments.user']));
    }

    /**
     * Update the specified resource in storage.
     */
    public function update(UpdatePostRequest $request, Post $post): PostResource
    {
        // Authorization handled in UpdatePostRequest
        $post->update($request->validated());

        return new PostResource($post->fresh('author'));
    }

    /**
     * Remove the specified resource from storage.
     */
    public function destroy(Post $post): JsonResponse
    {
        if ($post->user_id !== auth()->id()) {
            abort(403, 'You can only delete your own posts.');
        }

        $post->delete();

        return response()->json(null, Response::HTTP_NO_CONTENT);
    }
}
```

**3. Nested Resource Controller with Shallow Nesting**
```php
<?php
// app/Http/Controllers/CommentController.php
namespace App\Http\Controllers;

use App\Models\Post;
use App\Models\Comment;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;

class CommentController extends Controller
{
    /**
     * Display comments for a specific post.
     * URL: /posts/{post}/comments
     */
    public function index(Post $post)
    {
        $comments = $post->comments()
            ->approved()
            ->latest()
            ->paginate(30);

        return view('comments.index', compact('post', 'comments'));
    }

    /**
     * Store a new comment on a post.
     * URL: POST /posts/{post}/comments (needs parent context)
     */
    public function store(Request $request, Post $post): RedirectResponse
    {
        $validated = $request->validate([
            'body' => 'required|string|max:1000',
            'parent_id' => 'nullable|exists:comments,id',
        ]);

        $comment = $post->comments()->create([
            ...$validated,
            'user_id' => $request->user()->id,
        ]);

        return redirect()
            ->route('posts.show', $post)
            ->with('success', 'Comment added successfully.');
    }

    /**
     * Display a specific comment.
     * URL: /comments/{comment} (shallow - no post ID needed)
     */
    public function show(Comment $comment)
    {
        $comment->load(['user', 'replies.user']);

        return view('comments.show', compact('comment'));
    }

    /**
     * Update a comment.
     * URL: PATCH /comments/{comment} (shallow)
     */
    public function update(Request $request, Comment $comment): RedirectResponse
    {
        $this->authorize('update', $comment);

        $validated = $request->validate([
            'body' => 'required|string|max:1000',
        ]);

        $comment->update($validated);

        return redirect()
            ->back()
            ->with('success', 'Comment updated.');
    }

    /**
     * Delete a comment.
     * URL: DELETE /comments/{comment} (shallow)
     */
    public function destroy(Comment $comment): RedirectResponse
    {
        $this->authorize('delete', $comment);

        $comment->delete();

        return redirect()
            ->back()
            ->with('success', 'Comment deleted.');
    }
}

// Route registration for nested and shallow resources
// Route::resource('posts.comments', CommentController::class)->shallow();
```

**4. Singleton Resource Controller**
```php
<?php
// app/Http/Controllers/SettingsController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\View\View;
use Illuminate\Http\RedirectResponse;

class SettingsController extends Controller
{
    /**
     * Show the site settings page.
     * URL: GET /settings
     */
    public function show(): View
    {
        $settings = app('settings')->all();

        return view('settings.show', compact('settings'));
    }

    /**
     * Show the settings edit form.
     * URL: GET /settings/edit
     */
    public function edit(): View
    {
        $settings = app('settings')->all();

        return view('settings.edit', compact('settings'));
    }

    /**
     * Update the site settings.
     * URL: PATCH /settings
     */
    public function update(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'site_name' => 'required|string|max:100',
            'site_description' => 'nullable|string|max:500',
            'posts_per_page' => 'required|integer|min:5|max:100',
            'enable_comments' => 'boolean',
            'maintenance_mode' => 'boolean',
        ]);

        foreach ($validated as $key => $value) {
            app('settings')->set($key, $value);
        }

        return redirect()
            ->route('settings.edit')
            ->with('success', 'Settings updated successfully.');
    }
}

// Route registration for singleton
// Route::singleton('settings', SettingsController::class)->destroyable();
```

**Best Practices (Staff Engineer Tips)**

1. **Form Request Usage**: Always extract validation logic from controllers into form requests. This keeps controllers focused on orchestration, not validation. Form requests can also handle authorization, further cleaning controllers. A controller action should read like a narrative: validate, authorize, execute, respond.

2. **API Resource Transformation**: Never return raw Eloquent models from API controllers. Use API resource classes for consistent JSON structure. Resources handle attribute selection, relationship transformation, and conditional data inclusion. They evolve independently from your database schema changes.

3. **Consistent Response Patterns**: Establish response conventions. Web controllers redirect with flash messages. API controllers return JSON with appropriate status codes (201 for created, 204 for deleted, 422 for validation errors). Consistency enables building predictable client applications.

4. **Nested Resource Depth Limitation**: Limit nesting to one level maximum. `/posts/{post}/comments` is acceptable. `/users/{user}/posts/{post}/comments/{comment}` is not. Use shallow nesting for accessing individual resources. Deep nesting creates unmaintainable URL structures and confusing controller logic.

**Common Pitfalls and Solutions**

1. **Missing Authorization in Resource Controllers**: Generating controllers with `php artisan make:controller` doesn't add authorization. Beginners build full CRUD without protecting actions. Always add `$this->authorize()` calls or use form request `authorize()` methods. Use policies to centralize authorization logic.

2. **Returning HTML from API Controllers**: API controllers should never return views or redirects. A common mistake is copy-pasting web controller logic into API controllers. API controllers return JSON or specific HTTP status codes. Validation errors return 422 with error arrays, not redirects to create forms.

3. **Overfetching in API Controllers**: Loading all relationships in every API endpoint wastes bandwidth and processing. Use conditional eager loading based on `include` query parameters. Implement sparse fieldsets allowing clients to request only needed attributes. Standards like JSON:API provide patterns for this.

4. **Ignoring Resource Nesting Scope**: In nested controllers, developers often query records globally instead of scoping to the parent. A comments index route at `/posts/{post}/comments` should return only that post's comments. Returning all comments globally is a data leak exposing content from other posts.

---
