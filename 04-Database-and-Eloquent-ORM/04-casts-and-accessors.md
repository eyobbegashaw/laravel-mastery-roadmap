### Detailed Conceptual Overview (70%)

Casts and accessors transform raw database values into rich PHP objects and vice versa. They bridge the gap between your database's limited types (strings, integers, dates) and your application's rich domain objects (enums, value objects, collections, encrypted data).

**Attribute Casting: Automatic Type Conversion**

Attribute casting defines how Eloquent converts database values to PHP types when retrieving models and back to database values when saving. Declared in the model's `casts()` method, casting eliminates manual type conversion throughout your application.

The `datetime` cast converts database timestamps to Carbon instances, enabling date manipulation methods. The `boolean` cast converts 0/1 database integers to PHP booleans. The `array` and `json` casts automatically serialize and unserialize JSON columns. The `encrypted` cast encrypts values before storage and decrypts on retrieval.

Custom casts implement `CastsAttributes` interface for complex transformations. A `Money` cast converts database integers (stored as cents) to Money value objects. An `Address` cast converts JSON data to Address DTOs. Custom casts encapsulate transformation logic in reusable, testable classes.

**Accessors: Virtual Attributes and Computed Properties**

Eloquent accessors define attributes that don't exist as database columns but are computed from other data. An accessor named `getFullNameAttribute()` creates a `full_name` attribute by combining `first_name` and `last_name`. This virtual attribute appears in model arrays, JSON serialization, and can be accessed like any other attribute.

Laravel 11 introduced a preferred accessor syntax using `protected function fullName(): Attribute` returning an `Attribute` object. This approach is more explicit about which side of the transformation is being defined. The `get` parameter defines how to retrieve the value; the optional `set` parameter defines how to store it.

Accessors protect against null values elegantly. An accessor combining first and last names can return "Unknown User" when both fields are null, preventing " " display throughout the UI. Accessors can format values consistently—always formatting phone numbers, normalizing URLs, or capitalizing names.

**Mutators: Transforming Data on Assignment**

Mutators are the inverse of accessors—they transform values before storing them in the database. A `password` mutator automatically hashes values when setting the password attribute. An `email` mutator automatically lowercases email addresses. Mutators ensure data consistency regardless of where in the application the value is set.

**Value Objects and DTOs Through Casting**

Custom casts enable working with domain value objects instead of primitive types. Rather than passing around raw strings for email addresses, an `EmailAddress` cast wraps every email in a value object with validation and formatting methods. The model stores and retrieves the raw string, but your application code works with rich objects.

This pattern eliminates primitive obsession—the tendency to use strings and integers for everything. An `Order` with a `Money` cast for its `total_amount` attribute ensures proper currency handling without formatting logic scattered through views and controllers.

**Date Casting and Carbon**

Laravel's date casting converts MySQL datetime strings to Carbon instances automatically. Carbon provides an expressive API for date manipulation: `$user->created_at->diffForHumans()`, `$order->shipped_at->addDays(3)`, `$subscription->expires_at->isPast()`. Date serialization can be customized to JSON formats, ISO 8601, or custom formats.

### Production-Ready Code Snippets (30%)

```php
<?php
// app/Models/User.php - Comprehensive casting and accessors
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Casts\Attribute;
use App\Casts\PhoneNumber;
use App\Casts\EncryptedJson;
use App\ValueObjects\EmailAddress;
use App\Enums\UserStatus;

class User extends Authenticatable
{
    /**
     * Attribute casting for automatic type conversion.
     */
    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'last_login_at' => 'datetime',
            'password' => 'hashed', // Laravel 11 automatic hashing
            'is_active' => 'boolean',
            'preferences' => 'array',
            'notification_settings' => 'encrypted:array',
            'metadata' => EncryptedJson::class,
            'phone' => PhoneNumber::class,
            'status' => UserStatus::class, // Backed enum casting
            'date_of_birth' => 'date:Y-m-d',
            'account_balance' => 'decimal:2',
        ];
    }

    /**
     * Computed accessor for full name.
     * Combines first and last names, handles null gracefully.
     */
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) => 
                trim(sprintf('%s %s', 
                    $attributes['first_name'] ?? '', 
                    $attributes['last_name'] ?? ''
                )) ?: 'Unknown User',
        );
    }

    /**
     * Accessor with caching for expensive computation.
     */
    protected function lifetimeValue(): Attribute
    {
        return Attribute::make(
            get: function () {
                return cache()->remember(
                    "user:{$this->id}:ltv",
                    now()->addHours(6),
                    fn () => $this->orders()
                        ->where('status', 'delivered')
                        ->sum('total_amount')
                );
            }
        )->shouldCache();
    }

    /**
     * Mutator ensuring consistent email formatting.
     */
    protected function email(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => strtolower($value),
            set: fn (string $value) => strtolower(trim($value)),
        );
    }

    /**
     * Accessor formatting phone number for display.
     */
    protected function formattedPhone(): Attribute
    {
        return Attribute::make(
            get: function () {
                if (!$this->phone) {
                    return null;
                }
                // Format: (555) 123-4567
                $phone = $this->phone->value();
                return sprintf(
                    '(%s) %s-%s',
                    substr($phone, 0, 3),
                    substr($phone, 3, 3),
                    substr($phone, 6)
                );
            }
        );
    }
}

// app/Casts/Money.php - Custom value object cast
namespace App\Casts;

use App\ValueObjects\Money as MoneyValue;
use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Database\Eloquent\Model;

class Money implements CastsAttributes
{
    /**
     * Cast the stored value (cents as integer) to Money value object.
     */
    public function get(Model $model, string $key, mixed $value, array $attributes): ?MoneyValue
    {
        if ($value === null) {
            return null;
        }

        return new MoneyValue(
            amount: (int) $value,
            currency: $attributes['currency_code'] ?? 'USD'
        );
    }

    /**
     * Prepare the Money value object for storage (convert to cents).
     */
    public function set(Model $model, string $key, mixed $value, array $attributes): ?int
    {
        if ($value === null) {
            return null;
        }

        if ($value instanceof MoneyValue) {
            return $value->toCents();
        }

        // Handle raw float input (e.g., from forms)
        if (is_numeric($value)) {
            return (int) round((float) $value * 100);
        }

        throw new \InvalidArgumentException(
            'Value must be a Money object or numeric value.'
        );
    }
}

// app/ValueObjects/Money.php - Immutable value object
namespace App\ValueObjects;

use JsonSerializable;

readonly class Money implements JsonSerializable
{
    public function __construct(
        public int $amount, // In cents, to avoid floating point issues
        public string $currency = 'USD'
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Amount cannot be negative');
        }
    }

    /**
     * Format for display.
     */
    public function formatted(): string
    {
        return number_format($this->toDollars(), 2, '.', ',');
    }

    /**
     * Convert to dollars as float.
     */
    public function toDollars(): float
    {
        return $this->amount / 100;
    }

    /**
     * Convert to cents integer for database storage.
     */
    public function toCents(): int
    {
        return $this->amount;
    }

    /**
     * Add two Money objects (must be same currency).
     */
    public function add(self $other): self
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException('Cannot add money with different currencies');
        }

        return new self($this->amount + $other->amount, $this->currency);
    }

    /**
     * Apply percentage discount.
     */
    public function discount(float $percent): self
    {
        $discountAmount = (int) round($this->amount * ($percent / 100));
        return new self($this->amount - $discountAmount, $this->currency);
    }

    /**
     * JSON serialization.
     */
    public function jsonSerialize(): array
    {
        return [
            'amount' => $this->toDollars(),
            'currency' => $this->currency,
            'formatted' => $this->formatted(),
        ];
    }

    public function __toString(): string
    {
        return $this->formatted();
    }
}

// app/Enums/UserStatus.php - Backed enum for type-safe statuses
namespace App\Enums;

enum UserStatus: string
{
    case Active = 'active';
    case Inactive = 'inactive';
    case Suspended = 'suspended';
    case Banned = 'banned';

    /**
     * Get human-readable label.
     */
    public function label(): string
    {
        return match ($this) {
            self::Active => 'Active',
            self::Inactive => 'Inactive',
            self::Suspended => 'Suspended',
            self::Banned => 'Banned',
        };
    }

    /**
     * Get badge color for UI display.
     */
    public function color(): string
    {
        return match ($this) {
            self::Active => 'green',
            self::Inactive => 'gray',
            self::Suspended => 'yellow',
            self::Banned => 'red',
        };
    }

    /**
     * Check if user can access the application.
     */
    public function canAccess(): bool
    {
        return in_array($this, [self::Active]);
    }
}

// Usage examples in controller
namespace App\Http\Controllers;

use App\Models\User;
use App\ValueObjects\Money;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function show(User $user)
    {
        // Accessor provides computed attribute
        echo $user->full_name; // "John Doe"

        // Cast provides value object with methods
        $balance = $user->account_balance; // Money value object
        echo $balance->formatted(); // "$1,234.56"

        // Enum cast provides type-safe comparison
        if ($user->status === UserStatus::Active) {
            // Type-safe, no string comparison bugs
        }

        return view('users.show', compact('user'));
    }

    public function update(Request $request, User $user)
    {
        // Mutator automatically lowercases email
        $user->email = 'JOHN@EXAMPLE.COM';
        echo $user->email; // "john@example.com"

        // Money cast handles value objects
        $user->account_balance = new Money(5000); // $50.00 in cents
        $user->save();

        // Or raw float (setter converts to cents)
        $user->account_balance = 75.50;
        $user->save();
        // Database stores: 7550 (cents)

        return redirect()->back();
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Value Objects for Domain Concepts**: Any concept with validation rules or multiple representations (money with amount and currency, addresses with multiple fields) deserves a value object. Cast the value object at the model boundary and work with rich objects throughout the application.

2. **Encrypted Casting for Sensitive Data**: Use the `encrypted` cast for personally identifiable information (PII), health data, or financial details. Encryption happens transparently—your code works with plain values while the database stores encrypted blobs. Never log or cache unencrypted values.

3. **Enum Casting for Finite States**: Replace string status columns with backed enums. TypeScript-style type safety prevents invalid states. Enum methods encapsulate state-specific logic. Database values remain readable strings that map to enum cases.

4. **Computed Attribute Caching**: Expensive accessors (aggregating related models, external API calls) should cache their results. Use Laravel's cache with appropriate TTL. Invalidate cache on related model changes through observers.

**Common Pitfalls and Solutions**

1. **Casting Non-Nullable Columns Incorrectly**: Casts receive null values from nullable database columns. Handle null in your custom cast's `get()` method. Forgetting null checks causes TypeErrors that break model serialization.

2. **Overusing Array Casting**: The array cast causes full JSON serialization/deserialization on every model access. For frequently accessed models, this adds significant overhead. Use specific value objects instead of generic arrays. Only use array cast for truly dynamic, schema-less data.

3. **Accessor/Mutator Method Conflicts**: Defining both an accessor method and a `casts()` entry for the same attribute causes conflicts. Use the new `Attribute::make()` syntax for combined accessors and mutators. Keep models consistent by choosing one approach per attribute.

4. **Serialization Performance**: Models with many custom casts serialize slowly because each cast runs on serialization. API responses returning collections of such models can be slow. Use API resources with conditional attribute loading instead of full model serialization.

---

 