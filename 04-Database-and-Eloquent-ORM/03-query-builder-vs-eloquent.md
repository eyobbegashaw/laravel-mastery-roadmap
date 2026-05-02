### Detailed Conceptual Overview (70%)

Laravel provides two database abstraction layers: the Query Builder and Eloquent ORM. Understanding when to use each is crucial for building performant, maintainable applications. They're not competitors but complementary tools with different strengths.

**The Query Builder: Direct, Efficient Database Access**

The query builder provides a fluent interface for creating and running database queries. It works directly with table names and columns, returning `stdClass` objects or arrays. It's thin abstraction over raw SQL—every query builder method maps to a SQL clause: `->select()` becomes `SELECT`, `->where()` becomes `WHERE`, `->join()` becomes `JOIN`.

The query builder excels at aggregate operations, complex joins, and bulk updates. `DB::table('orders')->sum('total')` runs `SELECT SUM(total) FROM orders` efficiently. `DB::table('users')->where('status', 'inactive')->update(['archived_at' => now()])` performs a mass update in one query without loading models.

For reporting queries, dashboards, and data exports, the query builder provides precise control over the SQL generated. You're not constrained by Eloquent's relationship abstraction—you can write efficient queries with complex joins, subqueries, and window functions.

**Eloquent ORM: Rich Domain Models**

Eloquent wraps database tables in model classes that encapsulate business logic, relationships, and data transformation. A `User` model isn't just a row in the `users` table—it's a rich object with authentication methods, relationship accessors, event handlers, and serialization rules.

Eloquent shines in application logic where you work with individual records and their relationships. Creating a new order with items involves multiple models but reads naturally: `$user->orders()->create()->items()->createMany()`. Eloquent handles foreign key assignment, timestamps, and relationship syncing automatically.

**When to Use Which**

Use Eloquent when working with individual records, CRUD operations, and application business logic. The overhead of model instantiation (a few milliseconds per hundred records) is negligible compared to the developer productivity gains and code clarity.

Use the query builder for bulk operations, complex reports, data exports, and high-performance queries. A query builder statement updating 50,000 records runs in milliseconds. The same operation through Eloquent would load 50,000 models into memory and update them one at time—potentially exhausting memory and taking minutes.

**The Hybrid Approach**

You're not forced to choose exclusively. A single controller method might use Eloquent for authorization and basic queries, then drop to the query builder for a performance-critical aggregate. Eloquent models provide a `toBase()` method that returns a query builder instance scoped to the model's table, giving you builder power with model context.

**Performance Characteristics**

Eloquent's overhead comes from hydration—converting raw database results into model objects. Each model instantiation involves event firing, mutator processing, and relationship setup. Loading 10,000 records through Eloquent creates 10,000 model objects, each consuming memory and CPU cycles.

The query builder returns simple objects or arrays. Loading 10,000 records returns 10,000 lightweight `stdClass` instances or associative arrays. Memory usage is typically 5-10x lower than equivalent Eloquent queries.

### Production-Ready Code Snippets (30%)

```php
<?php
// app/Repositories/ReportRepository.php
namespace App\Repositories;

use App\Models\Order;
use App\Models\User;
use Illuminate\Support\Facades\DB;

class ReportRepository
{
    /**
     * Generate sales report using Query Builder for performance.
     * Complex aggregations with joins - Eloquent would be inefficient.
     */
    public function monthlySalesReport(int $year): array
    {
        return DB::table('orders')
            ->join('users', 'orders.user_id', '=', 'users.id')
            ->select(
                DB::raw('MONTH(orders.created_at) as month'),
                DB::raw('COUNT(*) as total_orders'),
                DB::raw('SUM(orders.total_amount) as revenue'),
                DB::raw('AVG(orders.total_amount) as average_order_value'),
                DB::raw('COUNT(DISTINCT users.id) as unique_customers')
            )
            ->whereYear('orders.created_at', $year)
            ->where('orders.status', '!=', 'cancelled')
            ->groupBy(DB::raw('MONTH(orders.created_at)'))
            ->orderBy('month')
            ->get()
            ->toArray();
    }

    /**
     * Complex query with subqueries and window functions.
     * Query Builder allows precise SQL control.
     */
    public function topCustomersByCategory(int $categoryId, int $limit = 10): array
    {
        return DB::table('users')
            ->join('orders', 'users.id', '=', 'orders.user_id')
            ->join('order_items', 'orders.id', '=', 'order_items.order_id')
            ->join('products', 'order_items.product_id', '=', 'products.id')
            ->select(
                'users.id',
                'users.name',
                'users.email',
                DB::raw('SUM(order_items.total_price) as total_spent'),
                DB::raw('COUNT(DISTINCT orders.id) as order_count')
            )
            ->where('products.category_id', $categoryId)
            ->where('orders.status', 'delivered')
            ->where('orders.created_at', '>=', now()->subYear())
            ->groupBy('users.id', 'users.name', 'users.email')
            ->orderByDesc('total_spent')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    /**
     * Bulk update using Query Builder - 50x faster than Eloquent.
     */
    public function applySeasonalDiscount(string $category, float $discountPercent): int
    {
        // Query Builder: Single UPDATE query, instant execution
        return DB::table('products')
            ->where('category', $category)
            ->where('is_active', true)
            ->update([
                'discount_percent' => $discountPercent,
                'discounted_price' => DB::raw("price * (1 - {$discountPercent} / 100)"),
                'updated_at' => now(),
            ]);

        // Eloquent equivalent (DON'T DO THIS for bulk updates):
        // Product::where('category', $category)
        //     ->where('is_active', true)
        //     ->each(function ($product) use ($discountPercent) {
        //         $product->update(['discount_percent' => $discountPercent]);
        //     }); // 10,000 products = 10,000 queries + model instantiation
    }
}

// app/Models/Order.php - Eloquent for business logic
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

class Order extends Model
{
    /**
     * Business logic method using Eloquent relationships.
     */
    public function recalculateTotal(): void
    {
        // Eloquent relationship access
        $subtotal = $this->items->sum('total_price');
        $tax = $subtotal * $this->tax_rate;
        
        $this->update([
            'subtotal' => $subtotal,
            'tax_amount' => $tax,
            'total_amount' => $subtotal + $tax - $this->discount_amount,
        ]);
    }

    /**
     * Scope for complex filtering with Eloquent.
     */
    public function scopeHighValue(Builder $query, float $threshold = 500): Builder
    {
        return $query->where('total_amount', '>=', $threshold)
            ->where('status', '!=', 'cancelled');
    }

    /**
     * Hybrid: Use Query Builder for related aggregates.
     */
    public function scopeWithCustomerStats(Builder $query): Builder
    {
        return $query->addSelect([
            'customer_lifetime_orders' => Order::selectRaw('COUNT(*)')
                ->whereColumn('user_id', 'orders.user_id')
                ->limit(1),
            'customer_total_spent' => Order::selectRaw('SUM(total_amount)')
                ->whereColumn('user_id', 'orders.user_id')
                ->where('status', 'delivered')
                ->limit(1),
        ]);
    }
}

// app/Services/DataExportService.php - Hybrid approach
namespace App\Services;

use App\Models\Order;
use Illuminate\Support\Facades\DB;
use League\Csv\Writer;

class DataExportService
{
    /**
     * Export large dataset efficiently.
     * Uses chunking with Eloquent for model events, IDs for memory efficiency.
     */
    public function exportOrders(string $filename): string
    {
        $csv = Writer::createFromPath(storage_path("exports/{$filename}"), 'w+');
        $csv->insertOne(['Order #', 'Customer', 'Total', 'Status', 'Date']);

        // Eloquent chunking processes thousands of records safely
        Order::with('user:id,name,email')
            ->select('id', 'user_id', 'order_number', 'total_amount', 'status', 'created_at')
            ->where('created_at', '>=', now()->subMonth())
            ->chunk(500, function ($orders) use ($csv) {
                foreach ($orders as $order) {
                    $csv->insertOne([
                        $order->order_number,
                        $order->user->name,
                        $order->total_amount,
                        $order->status,
                        $order->created_at->toDateString(),
                    ]);
                }
            });

        return $filename;
    }

    /**
     * Raw query for maximum performance on complex exports.
     */
    public function exportProductPerformance(): string
    {
        // Direct query builder - no Eloquent overhead for this analytics query
        $products = DB::table('products')
            ->leftJoin('order_items', 'products.id', '=', 'order_items.product_id')
            ->leftJoin('orders', 'order_items.order_id', '=', 'orders.id')
            ->select(
                'products.name',
                'products.sku',
                DB::raw('COUNT(DISTINCT orders.id) as times_ordered'),
                DB::raw('COALESCE(SUM(order_items.quantity), 0) as units_sold'),
                DB::raw('COALESCE(SUM(order_items.total_price), 0) as total_revenue'),
                DB::raw('COALESCE(AVG(products.rating), 0) as avg_rating')
            )
            ->where('orders.status', 'delivered')
            ->orWhereNull('orders.id') // Include products never ordered
            ->groupBy('products.id', 'products.name', 'products.sku')
            ->having('units_sold', '>', 0)
            ->orderByDesc('total_revenue')
            ->get();

        return $this->formatAsCsv($products);
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Use Query Builder for Aggregates**: Any query using `SUM`, `COUNT`, `AVG`, `MAX`, `MIN` should use the query builder directly. Eloquent collections can compute these values, but loading all records to sum a column is enormously wasteful. `DB::table('orders')->sum('total')` is a single efficient query.

2. **Eloquent for Individual Record Operations**: Creating, updating, and deleting individual records should use Eloquent. Model events fire, mutators process data, and relationships stay consistent. Using `DB::table()->insert()` bypasses these features and can create data integrity issues.

3. **Chunking for Large Datasets**: Both Eloquent and query builder offer chunking. Eloquent's `chunk()` method loads models in batches, allowing event processing and relationship access. Query builder's `chunkById()` is slightly faster for read-only operations. Never load entire tables into memory.

4. **Subqueries in Select Statements**: Eloquent provides `addSelect()` and subquery methods that compile to efficient SQL subqueries rather than additional database roundtrips. Use `selectRaw()` with subqueries instead of eager loading and counting relationship collections in PHP.

**Common Pitfalls and Solutions**

1. **Using Eloquent for Bulk Operations**: Calling `update()` on 100,000 records through Eloquent fires model events, processes mutators, and touches timestamps for each record individually. Use the query builder's `update()` method for bulk changes. If events are necessary, use `chunk()` and manage memory carefully.

2. **N+1 with Query Builder Joins**: Developers sometimes avoid Eloquent relationships to prevent N+1 queries, then manually join tables in query builder code, creating unmaintainable query logic. The solution is Eloquent with proper eager loading, not abandoning the ORM.

3. **Mixed Return Types**: A method returning Eloquent collections in some branches and query builder results in others creates type confusion. Query builder returns `stdClass`; Eloquent returns model instances. Decide on one approach per method and document the return type clearly.

4. **Ignoring Database Transactions**: Query builder operations don't automatically run in transactions. Multiple query builder calls that should be atomic (credit one account, debit another) require explicit transaction wrapping with `DB::transaction()`. Eloquent models in the same transaction benefit from automatic dirty checking.

---
