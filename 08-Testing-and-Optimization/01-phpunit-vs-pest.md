### Detailed Conceptual Overview (70%)

Testing is the foundation of maintainable software. Laravel provides two first-class testing frameworks: PHPUnit, the industry-standard PHP testing framework, and Pest, a modern testing framework built on top of PHPUnit with an expressive, minimal syntax. Understanding both enables you to choose the right tool and write effective tests.

**PHPUnit: The Battle-Tested Standard**

PHPUnit is the de facto testing framework for PHP. It's mature, extensively documented, and supported by every PHP IDE. PHPUnit tests are PHP classes extending `PHPUnit\Framework\TestCase` (for unit tests) or `Tests\TestCase` (for Laravel integration tests). Test methods are public methods prefixed with `test` or annotated with `@test`.

PHPUnit's assertion library provides methods for verifying expected behavior: `assertEquals()`, `assertTrue()`, `assertDatabaseHas()`, `assertJsonStructure()`. The framework discovers tests in configured directories, executes them in configurable orders, and generates detailed reports including code coverage.

Laravel extends PHPUnit with convenient helpers for testing web applications. The `Tests\TestCase` base class provides `actingAs()`, `get()`, `post()`, `assertStatus()`, `assertSee()`, and database assertion methods. Laravel's testing layer makes HTTP testing as expressive as unit testing.

**Pest: Modern, Expressive Testing**

Pest reimagines PHP testing with a functional, chainable API. Tests are standalone PHP files (not classes) using a closure-based syntax. The `test()` function receives a description and a closure containing assertions. The `it()` function creates tests with descriptions starting with "it". The `describe()` function groups related tests.

Pest's syntax reads like natural language: `test('users can view their profile')->get('/profile')->assertOk()`. The expect API provides expressive value testing: `expect($user->name)->toBe('John')->toStartWith('J')`. These chains create readable, self-documenting tests.

Pest plugins extend functionality. The Laravel plugin provides artisan commands, higher-order expectations, and convenience functions. The architecture plugin tests application structure. The mutation testing plugin evaluates test quality. Pest's plugin ecosystem grows rapidly.

**The Relationship: Pest Runs on PHPUnit**

Pest doesn't replace PHPUnit—it compiles to PHPUnit. All Pest tests are transpiled to PHPUnit classes before execution. This means every PHPUnit feature is available in Pest, and every third-party PHPUnit extension works with Pest. Code coverage, CI integration, and IDE support all function identically to PHPUnit.

The choice between PHPUnit and Pest is primarily syntactic. PHPUnit's class-based structure is familiar to developers from Java, C#, and other object-oriented backgrounds. Pest's functional syntax appeals to developers who prefer minimal boilerplate and higher readability.

**Test Types in Laravel**

Unit tests verify individual classes and methods in isolation. Dependencies are mocked. Unit tests run fast—hundreds per second—because they don't bootstrap the framework. They're ideal for testing algorithms, value objects, and service classes with complex business logic.

Feature tests verify that multiple components work together correctly. They boot the framework, hit endpoints, and verify responses. Feature tests typically interact with a test database, testing the full request lifecycle. They run slower than unit tests but verify real application behavior.

Browser tests (Laravel Dusk) automate real browser interactions. Dusk uses ChromeDriver to control Chrome, clicking buttons, filling forms, and verifying JavaScript behavior. Use browser tests sparingly—they're slow and brittle but verify critical user journeys.

**Test Databases and Factories**

Laravel creates an in-memory SQLite database for testing or uses a dedicated test database. The `RefreshDatabase` trait migrates a fresh database before each test. The `DatabaseTransactions` trait wraps each test in a transaction that rolls back, preserving the test database schema while keeping data isolated between tests.

Model factories generate test data. `User::factory()->create()` produces a user with randomized but valid attributes. Factories can create related models, apply states for specific scenarios, and generate large datasets for pagination or performance testing.

### Production-Ready Code Snippets (30%)

**1. PHPUnit Feature Tests for API**
```php
<?php
// tests/Feature/Api/OrderApiTest.php
namespace Tests\Feature\Api;

use App\Models\Order;
use App\Models\User;
use App\Models\Product;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\WithFaker;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

class OrderApiTest extends TestCase
{
    use RefreshDatabase, WithFaker;

    private User $user;
    private string $token;

    protected function setUp(): void
    {
        parent::setUp();
        
        // Create authenticated user with API token
        $this->user = User::factory()->create();
        $this->token = $this->user->createToken('test-token', ['orders:create'])->plainTextToken;
    }

    /** @test */
    public function authenticated_user_can_create_order()
    {
        $product = Product::factory()->create(['price' => 29.99, 'stock_quantity' => 10]);

        $response = $this->withHeaders([
            'Authorization' => 'Bearer ' . $this->token,
        ])->postJson('/api/v1/orders', [
            'items' => [
                ['product_id' => $product->id, 'quantity' => 2],
            ],
            'shipping_address' => [
                'street' => '123 Main St',
                'city' => 'Springfield',
                'state' => 'IL',
                'zip' => '62701',
                'country' => 'US',
            ],
            'shipping_phone' => '2175550123',
            'shipping_method' => 'standard',
            'payment_method' => 'credit_card',
            'payment_token' => 'tok_visa',
        ]);

        $response->assertStatus(201);
        $response->assertJsonStructure([
            'data' => [
                'id',
                'order_number',
                'total_amount',
                'status',
                'items' => [
                    '*' => ['product_id', 'quantity', 'unit_price', 'total_price'],
                ],
            ],
        ]);
        
        $this->assertDatabaseHas('orders', ['user_id' => $this->user->id]);
        $this->assertDatabaseHas('order_items', ['product_id' => $product->id, 'quantity' => 2]);
    }

    /** @test */
    public function cannot_create_order_with_insufficient_stock()
    {
        $product = Product::factory()->create(['price' => 29.99, 'stock_quantity' => 1]);

        $response = $this->withHeaders([
            'Authorization' => 'Bearer ' . $this->token,
        ])->postJson('/api/v1/orders', [
            'items' => [
                ['product_id' => $product->id, 'quantity' => 10], // Exceeds stock
            ],
            'shipping_address' => [
                'street' => '123 Main St',
                'city' => 'Springfield',
                'state' => 'IL',
                'zip' => '62701',
                'country' => 'US',
            ],
            'shipping_phone' => '2175550123',
            'shipping_method' => 'standard',
            'payment_method' => 'credit_card',
            'payment_token' => 'tok_visa',
        ]);

        $response->assertStatus(422);
        $response->assertJsonValidationErrors(['items.0.product_id']);
    }

    /** @test */
    public function user_cannot_access_other_users_orders()
    {
        $otherUser = User::factory()->create();
        $order = Order::factory()->for($otherUser)->create();

        $response = $this->withHeaders([
            'Authorization' => 'Bearer ' . $this->token,
        ])->getJson("/api/v1/orders/{$order->id}");

        $response->assertStatus(403);
    }

    /** @test */
    public function guest_cannot_create_order()
    {
        $response = $this->postJson('/api/v1/orders', []);

        $response->assertStatus(401);
    }
}
```

**2. Pest Test Examples**
```php
<?php
// tests/Feature/UserRegistrationTest.php
use App\Models\User;
use Illuminate\Support\Facades\Notification;
use App\Notifications\VerifyEmail;
use function Pest\Laravel\{postJson, assertDatabaseHas, assertDatabaseCount};

// Test grouping with describe
describe('User Registration', function () {
    
    beforeEach(function () {
        Notification::fake();
        // Clear any state before each test
    });

    it('allows users to register with valid data', function () {
        $response = postJson('/api/v1/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'Str0ng!Passw0rd',
            'password_confirmation' => 'Str0ng!Passw0rd',
        ]);

        $response->assertStatus(201)
            ->assertJsonStructure(['data' => ['id', 'name', 'email']]);
        
        assertDatabaseHas('users', ['email' => 'john@example.com']);
        assertDatabaseCount('users', 1);
    });

    it('validates required fields', function () {
        $response = postJson('/api/v1/register', []);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['name', 'email', 'password']);
    });

    it('requires password confirmation', function () {
        $response = postJson('/api/v1/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'Str0ng!Passw0rd',
            'password_confirmation' => 'DifferentPassword',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['password']);
    });

    it('prevents duplicate email registration', function () {
        User::factory()->create(['email' => 'john@example.com']);

        $response = postJson('/api/v1/register', [
            'name' => 'Another John',
            'email' => 'john@example.com',
            'password' => 'Str0ng!Passw0rd',
            'password_confirmation' => 'Str0ng!Passw0rd',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    });

    it('sends email verification notification', function () {
        postJson('/api/v1/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'Str0ng!Passw0rd',
            'password_confirmation' => 'Str0ng!Passw0rd',
        ]);

        $user = User::where('email', 'john@example.com')->first();
        
        Notification::assertSentTo(
            $user,
            VerifyEmail::class
        );
    });

    it('enforces strong password policy', function (string $weakPassword) {
        $response = postJson('/api/v1/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => $weakPassword,
            'password_confirmation' => $weakPassword,
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['password']);
    })->with([
        'password',      // Too short
        '12345678',      // No letters
        'password',      // Common password
        'abcdefgh',      // No numbers or symbols
    ]);
});

// tests/Unit/OrderTotalCalculationTest.php
use App\Models\Order;
use App\Models\OrderItem;
use App\Models\Product;

describe('Order Total Calculation', function () {
    
    it('calculates subtotal from item prices and quantities', function () {
        $order = Order::factory()->create();
        
        $items = collect([
            OrderItem::factory()->for($order)->create([
                'quantity' => 2,
                'unit_price' => 29.99,
            ]),
            OrderItem::factory()->for($order)->create([
                'quantity' => 1,
                'unit_price' => 49.99,
            ]),
        ]);

        $expectedSubtotal = (2 * 29.99) + (1 * 49.99);
        
        expect($order->calculateSubtotal())->toBe($expectedSubtotal);
    });

    it('applies percentage discount correctly', function () {
        $order = Order::factory()->create(['discount_percent' => 10]);
        
        OrderItem::factory()->for($order)->create([
            'quantity' => 1,
            'unit_price' => 100.00,
        ]);

        expect($order->calculateTotal())->toBe(90.00);
    });

    it('handles zero discount gracefully', function () {
        $order = Order::factory()->create(['discount_percent' => 0]);
        
        OrderItem::factory()->for($order)->create([
            'quantity' => 3,
            'unit_price' => 50.00,
        ]);

        expect($order->calculateTotal())->toBe(150.00);
    });

    it('enforces minimum order total', function () {
        $order = Order::factory()->create(['discount_percent' => 90]);
        
        OrderItem::factory()->for($order)->create([
            'quantity' => 1,
            'unit_price' => 100.00,
        ]);

        // Minimum $10 order enforced
        expect($order->calculateTotal())->toBe(10.00);
    });
});
```

**3. Unit Testing with Mocks**
```php
<?php
// tests/Unit/Services/OrderProcessorTest.php
namespace Tests\Unit\Services;

use App\Models\Order;
use App\Models\User;
use App\Services\OrderProcessor;
use App\Services\PaymentGateway;
use App\Services\InventoryService;
use App\Notifications\OrderConfirmation;
use Illuminate\Support\Facades\Notification;
use Mockery;
use Tests\TestCase;

class OrderProcessorTest extends TestCase
{
    private OrderProcessor $processor;
    private PaymentGateway $paymentGateway;
    private InventoryService $inventoryService;

    protected function setUp(): void
    {
        parent::setUp();

        // Create mock dependencies
        $this->paymentGateway = Mockery::mock(PaymentGateway::class);
        $this->inventoryService = Mockery::mock(InventoryService::class);
        Notification::fake();

        $this->processor = new OrderProcessor(
            $this->paymentGateway,
            $this->inventoryService
        );
    }

    /** @test */
    public function process_order_charges_payment_and_updates_status()
    {
        $user = User::factory()->make(['id' => 1]);
        $order = Order::factory()->make([
            'id' => 1,
            'user_id' => $user->id,
            'total_amount' => 150.00,
            'status' => 'pending',
        ]);

        // Set up expectations
        $this->paymentGateway
            ->shouldReceive('charge')
            ->with(150.00, 'tok_visa')
            ->once()
            ->andReturn((object) ['id' => 'ch_123', 'status' => 'succeeded']);

        $this->inventoryService
            ->shouldReceive('reserve')
            ->once()
            ->andReturn(true);

        $processedOrder = $this->processor->process($order, 'tok_visa');

        $this->assertEquals('paid', $processedOrder->status);
        $this->assertEquals('ch_123', $processedOrder->transaction_id);
    }

    /** @test */
    public function process_order_fails_when_payment_declined()
    {
        $this->expectException(\App\Exceptions\PaymentFailedException::class);

        $order = Order::factory()->make([
            'id' => 1,
            'total_amount' => 150.00,
            'status' => 'pending',
        ]);

        $this->paymentGateway
            ->shouldReceive('charge')
            ->with(150.00, 'tok_declined')
            ->once()
            ->andThrow(new \App\Exceptions\PaymentFailedException('Card declined'));

        $this->inventoryService
            ->shouldNotReceive('reserve'); // Doesn't reserve if payment fails

        $this->processor->process($order, 'tok_declined');
    }

    /** @test */
    public function process_order_sends_confirmation_notification()
    {
        $user = User::factory()->create();
        $order = Order::factory()->create([
            'user_id' => $user->id,
            'total_amount' => 150.00,
        ]);

        $this->paymentGateway
            ->shouldReceive('charge')
            ->andReturn((object) ['id' => 'ch_123', 'status' => 'succeeded']);

        $this->inventoryService
            ->shouldReceive('reserve')
            ->andReturn(true);

        $this->processor->process($order, 'tok_visa');

        Notification::assertSentTo($user, OrderConfirmation::class);
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Test Pyramid Strategy**: 60% unit tests (fast, isolated), 30% feature tests (integration, API), 10% browser tests (critical user journeys). Investment at lower pyramid levels provides the best ROI. A bug caught by a unit test costs seconds; caught in production, hours.

2. **Database Reset Strategy**: Use `RefreshDatabase` for most tests—it provides a clean schema per test. Use `DatabaseTransactions` for large test suites where migrations are slow. Never use shared test databases across parallel test runs.

3. **Factory Design for Tests**: Design factories with sensible defaults that create valid models without parameters. Use states for common scenarios. Use `create()` and `make()` appropriately—`make()` for unit tests (no database needed), `create()` for feature tests.

4. **Test Coverage Metrics**: Target 80%+ code coverage but don't worship the metric. Critical business logic (order processing, payment handling, authentication) should have 100% coverage. Config files, facades, and framework glue code don't need tests.

**Common Pitfalls and Solutions**

1. **Testing Implementation Details**: Tests verifying that a specific eloquent method was called or a particular query was executed are brittle. Test behavior, not implementation. If you refactor the implementation, behavior tests still pass; implementation tests break.

2. **Slow Test Suites**: Tests that interact with the database, filesystem, or external services are slow. Use mocks for external services. Use in-memory databases for unit tests. Profile slow tests and optimize them. A test suite that takes 20 minutes discourages running tests.

3. **Not Testing Edge Cases**: Testing only the happy path misses the most common source of bugs. Test empty inputs, maximum values, null values, concurrent requests, and unusual combinations. Edge cases are where production bugs live.

4. **Leaky Tests**: Tests sharing state through static properties, singletons, or the service container create interdependencies. Reset the application state between tests. Use `$this->refreshApplication()` in Laravel tests that modify container bindings.

---
