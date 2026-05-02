### Detailed Conceptual Overview (70%)

Database migrations are version control for your database schema. They allow you to define and modify database tables using PHP code instead of raw SQL, making schema changes portable across different database systems and environments. Seeders populate your database with test data, enabling consistent development and testing environments.

**The Philosophy of Database Migrations**

In traditional development, database schemas drift apart. A developer adds a column locally, another adds an index in staging, and production falls out of sync. Migrations solve this by making schema changes explicit, sequential, and reversible. Every change to your database structure becomes a migration file that can be applied (migrated) or reverted (rolled back).

Migrations use a timestamp-based naming convention that ensures they run in creation order. The `migrations` table in your database tracks which migrations have been applied. When you run `php artisan migrate`, Laravel checks this table and runs any pending migrations. When you roll back, it reverses the most recent batch of migrations.

**Schema Builder: Database-Agnostic Table Definitions**

Laravel's schema builder provides a fluent interface for defining database structures. Instead of writing `CREATE TABLE` statements in MySQL syntax that won't work in PostgreSQL, you use PHP methods that translate to the appropriate SQL for your database driver. This enables switching database systems with minimal code changes.

The schema builder handles column types, indexes, foreign keys, and table options. It generates the appropriate SQL for MySQL, PostgreSQL, SQLite, and SQL Server. Column modifiers like `nullable()`, `default()`, and `after()` apply database-specific syntax when supported.

**Migration Structure and Lifecycle**

Each migration class contains two methods: `up()` and `down()`. The `up()` method defines the changes to apply when migrating forward—creating tables, adding columns, creating indexes. The `down()` method reverses those changes—dropping tables, removing columns, dropping indexes. This reversibility enables rollbacks during development and safe deployments.

Some migrations are destructive (dropping a column that contained data) and cannot be fully reversed. In these cases, the `down()` method should throw an exception or handle the reversal gracefully with documentation. Destructive migrations require extra caution in production.

**Foreign Key Constraints and Data Integrity**

Foreign keys enforce referential integrity at the database level. When an `order_items` table references an `orders` table, foreign keys prevent creating order items for non-existent orders and can automatically handle cascading deletes. Laravel makes foreign key definition straightforward with `foreign()`, `references()`, `on()`, and `onDelete()` methods.

Cascading actions define what happens when referenced records change. `onDelete('cascade')` automatically deletes child records when the parent is deleted. `onDelete('set null')` sets the foreign key to null. `onDelete('restrict')` prevents deletion of parents with existing children. Choose cascading actions based on your data model's business logic.

**Seeders: Populating Meaningful Test Data**

Database seeders fill tables with data needed for application operation. They serve multiple purposes: providing default data (admin users, settings, roles), generating test data for development (fake users, orders, products), and creating known state for automated tests.

Model factories generate realistic fake data using the Faker library. A `UserFactory` can create users with random names, unique emails, and hashed passwords. Factories can define relationships, creating related models through callbacks. The `has()` method on factories creates associated models in a single call.

**Production Seeders vs. Development Seeders**

Production seeders contain essential reference data: countries, currencies, default roles, system settings. Development seeders generate sample data for testing interfaces and workflows. Separate seeders for each environment prevent test data from appearing in production databases.

### Production-Ready Code Snippets (30%)

**1. Comprehensive Migration with Complex Schema**
```php
<?php
// database/migrations/2024_01_15_000001_create_orders_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     */
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            // Primary key using ULID for public-facing IDs
            $table->ulid('id')->primary();
            
            // Foreign key to users with cascade delete
            $table->foreignId('user_id')
                ->constrained()
                ->cascadeOnDelete();
            
            // Order identification
            $table->string('order_number', 50)->unique();
            
            // Financial columns with proper decimal precision
            $table->decimal('subtotal', 12, 2);
            $table->decimal('tax_amount', 12, 2)->default(0);
            $table->decimal('discount_amount', 12, 2)->default(0);
            $table->decimal('total_amount', 12, 2);
            
            // JSON column for flexible data
            $table->json('shipping_address');
            $table->json('billing_address')->nullable();
            $table->json('metadata')->nullable();
            
            // Enum-like status with check constraint
            $table->string('status', 30)->default('pending');
            $table->string('payment_status', 30)->default('pending');
            
            // Nullable foreign key (shipping method not always assigned yet)
            $table->foreignId('shipping_method_id')
                ->nullable()
                ->constrained()
                ->nullOnDelete();
            
            // Performance indexes for common queries
            $table->index('status');
            $table->index('payment_status');
            $table->index('created_at');
            $table->index(['user_id', 'status']);
            
            // Full-text search index (MySQL/PostgreSQL)
            $table->text('notes')->nullable();
            if (DB::getDriverName() === 'mysql') {
                $table->fullText('notes');
            }
            
            $table->timestamps();
            $table->softDeletes();
        });

        // Create order_items table in same migration for consistency
        Schema::create('order_items', function (Blueprint $table) {
            $table->ulid('id')->primary();
            
            $table->foreignUlid('order_id')
                ->constrained()
                ->cascadeOnDelete();
            
            $table->foreignId('product_id')
                ->constrained()
                ->restrictOnDelete(); // Prevent deleting products with orders
            
            $table->string('product_name'); // Denormalized for history
            $table->string('product_sku', 50); // Denormalized
            $table->integer('quantity');
            $table->decimal('unit_price', 10, 2);
            $table->decimal('total_price', 12, 2);
            $table->json('options')->nullable(); // Size, color, etc.
            
            $table->timestamps();
            
            $table->index('order_id');
        });
    }

    /**
     * Reverse the migrations.
     */
    public function down(): void
    {
        Schema::dropIfExists('order_items');
        Schema::dropIfExists('orders');
    }
};
```

**2. Migration with Data Transformation**
```php
<?php
// database/migrations/2024_02_20_000001_refactor_user_name_fields.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    /**
     * Split full_name into first_name and last_name.
     * This demonstrates migrating existing data safely.
     */
    public function up(): void
    {
        // Step 1: Add new columns (nullable initially)
        Schema::table('users', function (Blueprint $table) {
            $table->string('first_name')->nullable()->after('id');
            $table->string('last_name')->nullable()->after('first_name');
        });

        // Step 2: Migrate existing data in batches
        DB::table('users')
            ->whereNotNull('full_name')
            ->orderBy('id')
            ->chunk(100, function ($users) {
                foreach ($users as $user) {
                    $parts = explode(' ', $user->full_name, 2);
                    
                    DB::table('users')
                        ->where('id', $user->id)
                        ->update([
                            'first_name' => $parts[0] ?? '',
                            'last_name' => $parts[1] ?? '',
                        ]);
                }
            });

        // Step 3: Make columns required after data migration
        Schema::table('users', function (Blueprint $table) {
            $table->string('first_name')->nullable(false)->change();
            $table->string('last_name')->nullable(false)->change();
        });

        // Step 4: Drop old column
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('full_name');
        });
    }

    /**
     * Reverse the migration.
     */
    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('full_name')->nullable()->after('id');
        });

        DB::table('users')
            ->orderBy('id')
            ->chunk(100, function ($users) {
                foreach ($users as $user) {
                    DB::table('users')
                        ->where('id', $user->id)
                        ->update([
                            'full_name' => trim($user->first_name . ' ' . $user->last_name),
                        ]);
                }
            });

        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(['first_name', 'last_name']);
            $table->string('full_name')->nullable(false)->change();
        });
    }
};
```

**3. Model Factory with States and Relationships**
```php
<?php
// database/factories/OrderFactory.php
namespace Database\Factories;

use App\Models\Order;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class OrderFactory extends Factory
{
    protected $model = Order::class;

    /**
     * Define the model's default state.
     */
    public function definition(): array
    {
        $subtotal = fake()->randomFloat(2, 10, 1000);
        $tax = round($subtotal * 0.08, 2);
        $discount = fake()->boolean(20) ? fake()->randomFloat(2, 5, 50) : 0;

        return [
            'user_id' => User::factory(),
            'order_number' => 'ORD-' . fake()->unique()->regexify('[A-Z0-9]{10}'),
            'subtotal' => $subtotal,
            'tax_amount' => $tax,
            'discount_amount' => $discount,
            'total_amount' => $subtotal + $tax - $discount,
            'shipping_address' => [
                'street' => fake()->streetAddress(),
                'city' => fake()->city(),
                'state' => fake()->stateAbbr(),
                'zip' => fake()->postcode(),
                'country' => 'US',
            ],
            'status' => fake()->randomElement(['pending', 'confirmed', 'shipped', 'delivered']),
            'payment_status' => 'paid',
            'created_at' => fake()->dateTimeBetween('-3 months', 'now'),
        ];
    }

    /**
     * Order with items.
     */
    public function withItems(int $count = 3): static
    {
        return $this->has(
            OrderItem::factory()->count($count),
            'items'
        );
    }

    /**
     * Indicate the order has been shipped.
     */
    public function shipped(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'shipped',
            'shipped_at' => now()->subDays(fake()->numberBetween(1, 7)),
        ]);
    }

    /**
     * Indicate the order is for a premium user.
     */
    public function forPremiumUser(): static
    {
        return $this->state(fn (array $attributes) => [
            'discount_amount' => fake()->randomFloat(2, 20, 100),
        ])->afterMaking(function (Order $order) {
            $order->total_amount = $order->subtotal 
                + $order->tax_amount 
                - $order->discount_amount;
        });
    }
}
```

**4. Comprehensive Database Seeder**
```php
<?php
// database/seeders/DatabaseSeeder.php
namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     */
    public function run(): void
    {
        // Production seeders (always run)
        $this->call([
            RolesAndPermissionsSeeder::class,
            CountrySeeder::class,
            CurrencySeeder::class,
            SystemSettingsSeeder::class,
        ]);

        // Development/Testing seeders (only in non-production)
        if (!app()->environment('production')) {
            $this->call([
                TestUserSeeder::class,
                SampleContentSeeder::class,
            ]);
        }
    }
}

// database/seeders/TestUserSeeder.php
namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;

class TestUserSeeder extends Seeder
{
    public function run(): void
    {
        // Create admin user
        User::factory()
            ->admin()
            ->create([
                'email' => 'admin@example.com',
                'first_name' => 'Admin',
                'last_name' => 'User',
            ]);

        // Create regular users with posts
        User::factory()
            ->count(10)
            ->hasPosts(5)
            ->create();

        // Create users with different roles
        User::factory()
            ->count(5)
            ->moderator()
            ->create();

        // Create premium users with orders
        User::factory()
            ->count(3)
            ->premium()
            ->has(
                Order::factory()
                    ->count(3)
                    ->withItems(4)
                    ->shipped()
            )
            ->create();

        // Progress output for visibility
        $this->command->info('Test users seeded successfully!');
        $this->command->table(
            ['Role', 'Count'],
            [
                ['Admin', '1'],
                ['Regular Users', '10'],
                ['Moderators', '5'],
                ['Premium Users', '3'],
            ]
        );
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Migration Granularity**: Each migration should make one logical change. Creating a users table and posts table in the same migration couples their histories together. If you need to modify the posts table later, you create a new migration without touching the users table change. This enables fine-grained rollbacks.

2. **Avoid Raw SQL in Migrations**: Use the schema builder methods exclusively. Raw SQL bypasses Laravel's database abstraction and can break when switching databases. If you must use raw SQL for complex operations, wrap it in database driver checks and document the reason.

3. **Data Migrations vs. Schema Migrations**: Keep schema changes and data changes in separate migrations. A migration adding a column should not also populate it—use a separate migration that can be tested independently. Data migrations with chunking prevent memory issues on large tables.

4. **Seeder Idempotency**: Seeders should be safe to run multiple times. Use `updateOrCreate()` or check existence before insertion. Truncating and re-seeding is acceptable during development but destructive in shared environments. Production seeders must be idempotent.

**Common Pitfalls and Solutions**

1. **Migration Failures in Production**: Running migrations on large tables with millions of rows causes timeouts. Use `chunk()` for data transformations. Add indexes before running queries that use them. Consider using tools like `gh-ost` or `pt-online-schema-change` for zero-downtime migrations on MySQL.

2. **Foreign Key Constraint Errors**: Foreign keys prevent destructive operations. Deleting a user with orders fails unless `onDelete('cascade')` is set. Seeding order matters—seed parent tables before child tables. Use `DB::statement('SET FOREIGN_KEY_CHECKS=0;')` only in seeders when necessary and re-enable immediately after.

3. **Hardcoded IDs in Seeders**: Assuming auto-increment IDs starts at 1 creates fragile seeders. Always reference model instances (not IDs) when creating related data. Use factories with relationships instead of manually setting foreign key values.

4. **Rolling Back Migrations with Data Loss**: Adding a column with a migration and then rolling back drops the column and its data permanently. Always test rollbacks in staging before production. Back up the database before running new migrations. Consider making initial migrations non-reversible after data exists.

---
