### Detailed Conceptual Overview (70%)

Eloquent relationships transform complex SQL joins into simple, expressive PHP methods. Understanding relationships deeply—not just their syntax but their underlying queries and performance characteristics—separates developers who build scalable applications from those who create N+1 query nightmares.

**One-to-One Relationships: The Simplest Connection**

A one-to-one relationship exists when a record in one table corresponds to exactly one record in another table. A user has one profile, an order has one invoice, an employee has one parking spot. The relationship is defined by a foreign key on one of the tables, typically the one that "belongs to" the other.

The owning side of the relationship holds the foreign key. A `profiles` table has a `user_id` column linking to the `users` table. The `User` model defines `hasOne(Profile::class)`—users have one profile. The `Profile` model defines `belongsTo(User::class)`—profiles belong to a user. The semantics clarify the data direction.

Eloquent determines foreign keys by convention: the related model name in snake_case with `_id` suffix. For `User` and `Profile`, Eloquent looks for `user_id` on the profiles table. Non-conventional keys require explicit specification: `hasOne(Profile::class, 'owner_id')`.

**One-to-Many Relationships: The Workhorse Pattern**

One-to-many relationships handle the most common data pattern: one record owns multiple child records. A blog post has many comments, a customer has many orders, a category has many products. This relationship is the foundation of most application data models.

The "many" side defines `belongsTo()` and holds the foreign key. The "one" side defines `hasMany()`. Eloquent provides methods to interact with the relationship: `$post->comments` retrieves all comments, `$post->comments()->create()` adds a new comment, `$post->comments()->count()` counts without loading.

**Many-to-Many Relationships: The Pivot Pattern**

Many-to-many relationships model complex associations. A user belongs to many roles, and a role has many users. A product belongs to many categories, and a category contains many products. These relationships require a pivot table containing foreign keys to both related tables.

Pivot tables follow naming convention: the singular names of both models in alphabetical order, separated by an underscore. Users and roles use `role_user`. Products and categories use `category_product`. Laravel provides `belongsToMany()` on both models, set up symmetrically.

Pivot tables can carry additional data beyond the foreign keys. A `role_user` table might include `assigned_at` and `expires_at` timestamps. Access pivot data with `$user->roles->first()->pivot->assigned_at`. Custom pivot models provide type casting and relationships on this intermediate data.

**Has-Many-Through: Accessing Distant Relations**

Has-many-through provides convenient access to distant relationships through an intermediate model. A country has many users through posts—each post belongs to a user, and each user belongs to a country. This pattern avoids manual joins when you need to query "all posts from users in Canada."

The relationship requires that the intermediate table has a foreign key to the far table. `posts` has a `user_id`, `users` has a `country_id`. The `Country` model defines `hasManyThrough(Post::class, User::class)`—"get posts through users."

**Polymorphic Relationships: One Table, Many Owners**

Polymorphic relationships allow a model to belong to different types of models using a single association table. Comments can belong to posts and videos. Images can belong to products and user profiles. The polymorphic pattern uses two columns: the morph type (the related model class) and the morph ID (the related model's primary key).

Morph maps map model class names to shorter strings stored in the database. Using `'post'` instead of `'App\Models\Post'` makes the database more readable and allows model restructuring without updating database values.

### Production-Ready Code Snippets (30%)

**1. Comprehensive Eloquent Models with All Relationship Types**
```php
<?php
// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Relations\{HasOne, HasMany, BelongsToMany};

class User extends Authenticatable
{
    /**
     * One-to-One: User has one profile.
     */
    public function profile(): HasOne
    {
        return $this->hasOne(Profile::class);
    }

    /**
     * One-to-Many: User has many orders.
     */
    public function orders(): HasMany
    {
        return $this->hasMany(Order::class);
    }

    /**
     * One-to-Many with additional constraints: Active orders only.
     */
    public function activeOrders(): HasMany
    {
        return $this->orders()->whereIn('status', ['pending', 'confirmed', 'shipped']);
    }

    /**
     * Many-to-Many: User belongs to many roles.
     */
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)
            ->withPivot('assigned_at', 'expires_at')
            ->withTimestamps()
            ->using(RoleUserPivot::class);
    }

    /**
     * Has-Many-Through: Get all order items through orders.
     */
    public function orderItems()
    {
        return $this->hasManyThrough(
            OrderItem::class,
            Order::class,
            'user_id', // Foreign key on orders table
            'order_id', // Foreign key on order_items table
            'id',       // Local key on users table
            'id'        // Local key on orders table
        );
    }

    /**
     * Polymorphic One-to-Many: User has many addresses.
     */
    public function addresses()
    {
        return $this->morphMany(Address::class, 'addressable');
    }

    /**
     * Polymorphic One-to-One: User's default shipping address.
     */
    public function defaultShippingAddress()
    {
        return $this->morphOne(Address::class, 'addressable')
            ->where('type', 'shipping')
            ->where('is_default', true);
    }

    /**
     * Polymorphic Many-to-Many: User's tags.
     */
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable')
            ->withTimestamps();
    }
}

// app/Models/Post.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\{BelongsTo, HasMany, MorphMany, MorphToMany};

class Post extends Model
{
    /**
     * Inverse One-to-Many: Post belongs to a user.
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /**
     * One-to-Many: Post has many comments.
     */
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    /**
     * Polymorphic One-to-Many: Post has many comments (using polymorphic).
     */
    public function polyComments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }

    /**
     * Polymorphic Many-to-Many: Post has many tags.
     */
    public function tags(): MorphToMany
    {
        return $this->morphToMany(Tag::class, 'taggable')
            ->withPivot('order')
            ->orderByPivot('order');
    }

    /**
     * Has-Many-Through: Get all comment authors through comments.
     */
    public function commentAuthors()
    {
        return $this->hasManyThrough(
            User::class,
            Comment::class,
            'post_id',
            'id',
            'id',
            'user_id'
        )->distinct();
    }
}
```

**2. Advanced Relationship Querying and Eager Loading**
```php
<?php
// app/Http/Controllers/ReportingController.php
namespace App\Http\Controllers;

use App\Models\User;
use App\Models\Order;
use App\Models\Post;
use Illuminate\Http\Request;

class ReportingController extends Controller
{
    /**
     * Generate user activity report with optimized queries.
     */
    public function userActivityReport(Request $request)
    {
        // Eager load with constraints and nested relationships
        $users = User::query()
            ->with([
                'profile', // Always loaded
                
                // Conditional eager loading
                'orders' => function ($query) {
                    $query->where('created_at', '>=', now()->subMonths(3))
                        ->with('items.product') // Nested relationship
                        ->withSum('items as total_items', 'quantity') // Aggregate
                        ->withCount('items'); // Count relationship
                },
                
                // Load only if condition met
                'posts' => function ($query) {
                    $query->published()
                        ->withCount('comments')
                        ->with(['tags:id,name']);
                },
                
                // Check existence without loading
                'roles',
            ])
            ->withExists(['orders as has_recent_orders' => function ($query) {
                $query->where('created_at', '>=', now()->subMonth());
            }])
            ->withCount('orders as lifetime_orders')
            ->get();

        // Filter users without loading all orders into memory
        $activeUsers = User::whereHas('orders', function ($query) {
            $query->where('created_at', '>=', now()->subMonth())
                ->where('total_amount', '>', 100);
        }, '>=', 3) // Users with 3+ large orders recently
        ->get();

        // Querying relationship existence with conditions
        $usersWithPublishedPosts = User::whereHas('posts', function ($query) {
            $query->published();
        })->get();

        // Get users WITHOUT any orders (anti-pattern check)
        $usersWithoutOrders = User::whereDoesntHave('orders')->get();

        return view('reports.users', compact(
            'users', 
            'activeUsers', 
            'usersWithPublishedPosts',
            'usersWithoutOrders'
        ));
    }

    /**
     * Lazy eager loading for conditional data loading.
     */
    public function showUser(Request $request, User $user)
    {
        // Load profile if not already loaded
        if (!$user->relationLoaded('profile')) {
            $user->load('profile');
        }

        // Conditionally load relationships based on request
        if ($request->has('include_orders')) {
            $user->load(['orders' => function ($query) use ($request) {
                $query->whereBetween('created_at', [
                    $request->date('from', now()->subMonth()),
                    $request->date('to', now()),
                ]);
            }]);
        }

        // Load count without loading models
        $user->loadCount([
            'orders',
            'posts',
            'comments',
            'orders as completed_orders_count' => function ($query) {
                $query->where('status', 'delivered');
            },
        ]);

        return view('users.show', compact('user'));
    }
}
```

**3. Polymorphic Relationship Implementation**
```php
<?php
// app/Models/Comment.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Comment extends Model
{
    /**
     * Get the parent commentable model (post, video, etc.).
     */
    public function commentable(): MorphTo
    {
        return $this->morphTo();
    }

    /**
     * Polymorphic: Comment belongs to a user.
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }

    /**
     * Polymorphic One-to-Many: Comment has many likes.
     */
    public function likes()
    {
        return $this->morphMany(Like::class, 'likeable');
    }
}

// app/Models/Tag.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

class Tag extends Model
{
    /**
     * Get all posts associated with this tag.
     */
    public function posts(): MorphToMany
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }

    /**
     * Get all products associated with this tag.
     */
    public function products(): MorphToMany
    {
        return $this->morphedByMany(Product::class, 'taggable');
    }

    /**
     * Get all videos associated with this tag.
     */
    public function videos(): MorphToMany
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
}

// Migration for polymorphic relationships
// database/migrations/2024_03_01_000001_create_comments_table.php
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->text('body');
    
    // Polymorphic relation columns
    $table->morphs('commentable'); // Creates commentable_type and commentable_id
    
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->timestamps();
});

// Morph map for shorter database values
// app/Providers/AppServiceProvider.php
use Illuminate\Database\Eloquent\Relations\Relation;

public function boot(): void
{
    Relation::enforceMorphMap([
        'post' => 'App\Models\Post',
        'video' => 'App\Models\Video',
        'product' => 'App\Models\Product',
        'user' => 'App\Models\User',
    ]);
}
```

**4. Custom Pivot Model with Additional Data**
```php
<?php
// app/Models/Pivots/RoleUserPivot.php
namespace App\Models\Pivots;

use Illuminate\Database\Eloquent\Relations\Pivot;

class RoleUserPivot extends Pivot
{
    /**
     * The attributes that should be cast.
     */
    protected function casts(): array
    {
        return [
            'assigned_at' => 'datetime',
            'expires_at' => 'datetime',
            'is_active' => 'boolean',
        ];
    }

    /**
     * Check if this role assignment is currently active.
     */
    public function isActive(): bool
    {
        if (!$this->is_active) {
            return false;
        }

        if ($this->expires_at && $this->expires_at->isPast()) {
            return false;
        }

        return true;
    }

    /**
     * Scope query to active assignments.
     */
    public function scopeActive($query)
    {
        return $query->where('is_active', true)
            ->where(function ($q) {
                $q->whereNull('expires_at')
                  ->orWhere('expires_at', '>', now());
            });
    }

    /**
     * Custom relationship through pivot.
     */
    public function assignedBy()
    {
        return $this->belongsTo(User::class, 'assigned_by');
    }
}

// Usage in User model
public function roles(): BelongsToMany
{
    return $this->belongsToMany(Role::class)
        ->withPivot(['assigned_at', 'expires_at', 'is_active', 'assigned_by'])
        ->withTimestamps()
        ->using(RoleUserPivot::class)
        ->wherePivot('is_active', true); // Only active roles by default
}

// Usage in controller
$user = User::with('roles')->first();
foreach ($user->roles as $role) {
    $pivot = $role->pivot; // RoleUserPivot instance
    echo "Assigned: {$pivot->assigned_at->diffForHumans()}";
    echo "Active: " . ($pivot->isActive() ? 'Yes' : 'No');
}
```

**Best Practices (Staff Engineer Tips)**

1. **Eager Loading by Default**: Configure `$with` on models for relationships always needed. Users almost always need their profile, so add `protected $with = ['profile']`. Override with `without()` when loading is unnecessary. For frequently accessed relationships, this eliminates N+1 queries systematically.

2. **Relationship Naming Conventions**: Use singular names for singular relationships (profile, address) and plural for collections (comments, orders). BelongsTo methods should be named after the related model in singular: `user()`, `category()`, `parent()`. Consistent naming enables the `load()` and `with()` methods to work predictably.

3. **Querying Relationships Instead of Collections**: Always query relationships through the relationship method when filtering, sorting, or limiting. `$user->posts()->where('published', true)->get()` generates a single optimized query. `$user->posts->where('published', true)` loads all posts into memory and filters in PHP.

4. **Polymorphic Relationship Design**: Use polymorphic relationships only when the related models are genuinely different types. Comments on posts and videos make sense polymorphically. Posts and pages (which are nearly identical) should share a common model or interface instead.

**Common Pitfalls and Solutions**

1. **N+1 Query Problem**: The most common and severe performance issue. A loop accessing `$post->user->name` triggers a query for each post's user. Always eager load with `with(['user'])`. Use Laravel Debugbar or Telescope in development to identify N+1 queries. Configure `Model::preventLazyLoading()` in development to throw exceptions.

2. **Incorrect Foreign Key Conventions**: Breaking naming conventions requires explicit key specification everywhere the relationship is used. A `Post` model with `hasMany(Comment::class, 'post_id', 'id')` works, but every query, eager load, and relationship method needs the same explicit keys. Follow conventions to eliminate configuration.

3. **Massive Pivot Tables**: Many-to-many relationships create large intermediate tables. Index both foreign key columns. Consider using `attach()` and `detach()` with arrays instead of loops. Use `sync()` for complete relationship updates. Paginate pivot queries when accessing relationships with thousands of records.

4. **Polymorphic Querying Limitations**: Polymorphic relationships prevent foreign key constraints at the database level. Performance degrades with millions of records because the database can't efficiently join across arbitrary types. Consider separate tables for each relationship type for high-traffic polymorphic associations.

---
