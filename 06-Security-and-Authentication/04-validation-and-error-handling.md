### Detailed Conceptual Overview (70%)

Validation and error handling form the defensive perimeter of your application. They protect your data integrity, provide meaningful feedback to users, and prevent security vulnerabilities that arise from malformed input. In Laravel, these layers work together to create a robust input processing pipeline.

**Form Request Validation: The Preferred Approach**

Form requests are custom request classes that encapsulate validation and authorization logic. They move validation rules out of controllers, keeping controllers focused on orchestration. A `StorePostRequest` validates that the title is required, the body meets minimum length, and tags exist in the database—all before the controller method executes.

Form requests run authorization checks through the `authorize()` method before validation. If authorization fails, a 403 response returns immediately. If authorization passes, validation rules from `rules()` execute. Only when both pass does the validated data reach the controller. This pipeline ensures no invalid or unauthorized data enters your business logic.

The `messages()` method customizes error messages per rule. The `attributes()` method provides human-readable attribute names for error messages. The `prepareForValidation()` method sanitizes or transforms input before validation rules apply—normalizing phone numbers, trimming whitespace, converting formats.

**Validation Rules: Comprehensive Input Checking**

Laravel provides over 90 validation rules covering common data validation scenarios. String validation (`required`, `min`, `max`, `email`, `url`, `regex`), numeric validation (`integer`, `numeric`, `min`, `max`, `between`), database validation (`unique`, `exists`), and file validation (`file`, `image`, `mimes`, `max`) handle most use cases.

Rule objects encapsulate custom validation logic. A `ValidDomain` rule might check MX records exist for an email domain. A `NotReservedUsername` rule prevents users from claiming system usernames. Rule objects keep complex validation logic testable and reusable across the application.

Conditional validation adds rules based on other input values. `required_if:payment_type,credit_card` makes credit card fields required only when credit card payment is selected. `exclude_if` and `exclude_unless` remove fields entirely when conditions aren't met, preventing validation of irrelevant data.

**Error Handling Strategy**

Laravel converts exceptions into HTTP responses through the exception handler. `ValidationException` becomes 422 responses with error details. `AuthenticationException` becomes 401 responses. `AuthorizationException` becomes 403 responses. `ModelNotFoundException` becomes 404 responses. This consistent mapping means controllers don't catch these exceptions—they propagate to the handler.

Custom exceptions add business context to errors. An `InsufficientFundsException` carries the account balance and required amount. The exception handler can render this as a structured JSON response for API consumers or a flash message for web users. Exceptions become part of your application's API contract.

**Logging and Monitoring**

Validation failures and exceptions should be logged at appropriate levels. Validation failures are `info` or `debug` level—they're expected user behavior. Server errors are `error` or `critical` level—they indicate bugs. Authentication failures are `warning` level—they could be attacks or forgetful users. Proper log levels enable effective monitoring.

### Production-Ready Code Snippets (30%)

**1. Comprehensive Form Request with Complex Validation**
```php
<?php
// app/Http/Requests/StoreOrderRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use App\Rules\ValidShippingAddress;
use App\Rules\SufficientInventory;

class StoreOrderRequest extends FormRequest
{
    /**
     * Determine if the user is authorized.
     */
    public function authorize(): bool
    {
        // User must be authenticated and have verified email
        return $this->user() && $this->user()->hasVerifiedEmail();
    }

    /**
     * Handle failed authorization.
     */
    protected function failedAuthorization()
    {
        throw new \Illuminate\Auth\Access\AuthorizationException(
            'You must verify your email before placing orders.'
        );
    }

    /**
     * Prepare data before validation.
     */
    protected function prepareForValidation(): void
    {
        // Normalize phone number format
        if ($this->has('shipping_phone')) {
            $this->merge([
                'shipping_phone' => preg_replace('/[^0-9]/', '', $this->shipping_phone),
            ]);
        }

        // Convert comma-separated string to array
        if (is_string($this->input('items'))) {
            $this->merge([
                'items' => json_decode($this->input('items'), true),
            ]);
        }
    }

    /**
     * Validation rules.
     */
    public function rules(): array
    {
        return [
            // Shipping information
            'shipping_address.street' => ['required', 'string', 'max:200'],
            'shipping_address.city' => ['required', 'string', 'max:100'],
            'shipping_address.state' => ['required', 'string', 'max:50'],
            'shipping_address.zip' => ['required', 'string', 'regex:/^\d{5}(-\d{4})?$/'],
            'shipping_address.country' => ['required', 'string', 'size:2'],
            'shipping_phone' => ['required', 'string', 'size:10'],
            'shipping_method' => [
                'required',
                Rule::in(['standard', 'express', 'overnight']),
            ],

            // Order items
            'items' => ['required', 'array', 'min:1', 'max:50'],
            'items.*.product_id' => [
                'required',
                'integer',
                'exists:products,id',
                new SufficientInventory($this->input('items.*.quantity')),
            ],
            'items.*.quantity' => ['required', 'integer', 'min:1', 'max:100'],
            'items.*.options' => ['nullable', 'array'],

            // Payment
            'payment_method' => ['required', Rule::in(['credit_card', 'paypal', 'apple_pay'])],
            'payment_token' => ['required_if:payment_method,credit_card', 'string'],

            // Coupon
            'coupon_code' => [
                'nullable',
                'string',
                'max:20',
                Rule::exists('coupons', 'code')->where(function ($query) {
                    $query->where('is_active', true)
                        ->where('expires_at', '>', now())
                        ->where('usage_limit', '>', 
                            \DB::raw('(SELECT COUNT(*) FROM orders WHERE coupon_id = coupons.id)')
                        );
                }),
            ],

            // Delivery notes
            'notes' => ['nullable', 'string', 'max:500'],
            'gift_message' => ['nullable', 'string', 'max:200'],

            // Shipping address validation
            'shipping_address' => [new ValidShippingAddress],
        ];
    }

    /**
     * Custom error messages.
     */
    public function messages(): array
    {
        return [
            'items.required' => 'Your order must contain at least one item.',
            'items.*.product_id.exists' => 'One of the products in your order is no longer available.',
            'shipping_phone.size' => 'Please enter a valid 10-digit phone number.',
            'coupon_code.exists' => 'This coupon code is invalid or has expired.',
            'payment_token.required_if' => 'Credit card details are required for this payment method.',
        ];
    }

    /**
     * Custom attribute names for errors.
     */
    public function attributes(): array
    {
        return [
            'shipping_address.street' => 'street address',
            'shipping_address.city' => 'city',
            'shipping_address.state' => 'state',
            'shipping_address.zip' => 'ZIP code',
            'items.*.product_id' => 'product',
            'items.*.quantity' => 'quantity',
        ];
    }

    /**
     * Configure the validator instance.
     */
    public function withValidator($validator): void
    {
        $validator->after(function ($validator) {
            // Validate total order amount meets minimum
            $total = collect($this->input('items'))->sum(function ($item) {
                $product = \App\Models\Product::find($item['product_id']);
                return ($product->price ?? 0) * $item['quantity'];
            });

            if ($total < 10.00) {
                $validator->errors()->add(
                    'items',
                    'Minimum order amount is $10.00. Your total is $' . number_format($total, 2)
                );
            }

            // Validate shipping method availability based on items
            $hasPerishableItems = \App\Models\Product::whereIn('id', 
                collect($this->input('items'))->pluck('product_id')
            )->where('perishable', true)->exists();

            if ($hasPerishableItems && !in_array($this->input('shipping_method'), ['express', 'overnight'])) {
                $validator->errors()->add(
                    'shipping_method',
                    'Perishable items require express or overnight shipping.'
                );
            }
        });
    }
}
```

**2. Custom Validation Rule**
```php
<?php
// app/Rules/ValidShippingAddress.php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use App\Services\AddressValidationService;

class ValidShippingAddress implements ValidationRule
{
    /**
     * Run the validation rule.
     * Uses an external address validation API.
     */
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $addressService = app(AddressValidationService::class);

        $result = cache()->remember(
            'address_validation_' . md5(json_encode($value)),
            now()->addDays(30),
            fn () => $addressService->validate(
                street: $value['street'] ?? '',
                city: $value['city'] ?? '',
                state: $value['state'] ?? '',
                zip: $value['zip'] ?? '',
                country: $value['country'] ?? '',
            )
        );

        if (!$result['valid']) {
            $fail('The shipping address could not be verified. Please check your address details.');
        }

        // Store normalized address for later use
        request()->merge([
            'shipping_address' => $result['normalized'] ?? $value,
            'shipping_address_verified' => true,
        ]);
    }
}

// app/Rules/SufficientInventory.php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use App\Models\Product;

class SufficientInventory implements ValidationRule
{
    public function __construct(
        private mixed $quantities
    ) {}

    /**
     * Check if product has sufficient inventory for the requested quantity.
     */
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        $product = Product::find($value);
        
        if (!$product) {
            $fail('Product not found.');
            return;
        }

        // Get the quantity for this specific product from the items array
        $index = explode('.', $attribute)[1]; // Get index from "items.0.product_id"
        $quantity = is_array($this->quantities) 
            ? ($this->quantities[$index] ?? 0)
            : $this->quantities;

        if ($product->stock_quantity < $quantity) {
            $fail("Insufficient stock for {$product->name}. " .
                  "Only {$product->stock_quantity} available, you requested {$quantity}.");
        }
    }
}
```

**3. Exception Handler Configuration**
```php
<?php
// bootstrap/app.php - Exception Handling Configuration
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Validation\ValidationException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Symfony\Component\HttpKernel\Exception\HttpException;

return Application::configure(basePath: dirname(__DIR__))
    ->withExceptions(function (Exceptions $exceptions) {
        
        // Custom validation error format for API
        $exceptions->renderable(function (ValidationException $e, $request) {
            if ($request->expectsJson() || $request->is('api/*')) {
                return response()->json([
                    'success' => false,
                    'message' => 'The given data was invalid.',
                    'errors' => $e->errors(),
                ], 422);
            }
        });

        // Hide resource existence from unauthorized users
        $exceptions->renderable(function (ModelNotFoundException $e, $request) {
            if ($request->expectsJson() || $request->is('api/*')) {
                return response()->json([
                    'success' => false,
                    'message' => 'Resource not found.',
                ], 404);
            }
        });

        // Custom authentication error
        $exceptions->renderable(function (AuthenticationException $e, $request) {
            if ($request->expectsJson() || $request->is('api/*')) {
                return response()->json([
                    'success' => false,
                    'message' => 'Unauthenticated. Please provide a valid token.',
                ], 401);
            }
        });

        // Rate limiting error (429)
        $exceptions->renderable(function (HttpException $e, $request) {
            if ($e->getStatusCode() === 429) {
                return response()->json([
                    'success' => false,
                    'message' => 'Too many requests. Please slow down.',
                    'retry_after' => $e->getHeaders()['Retry-After'] ?? 60,
                ], 429);
            }
        });

        // Don't report validation exceptions (they're expected)
        $exceptions->dontReport([
            ValidationException::class,
        ]);

        // Report custom exceptions with context
        $exceptions->reportable(function (\App\Exceptions\PaymentFailedException $e) {
            \Log::critical('Payment failed', [
                'order_id' => $e->getOrderId(),
                'amount' => $e->getAmount(),
                'reason' => $e->getMessage(),
            ]);
            
            // Notify ops team for high-value failures
            if ($e->getAmount() > 1000) {
                \Notification::route('slack', config('services.slack.ops_webhook'))
                    ->notify(new \App\Notifications\PaymentFailedAlert($e));
            }
        });

        // Custom HTTP exception responses
        $exceptions->respond(function ($response, $exception, $request) {
            if ($response->getStatusCode() === 419) {
                return back()->withInput()->with([
                    'error' => 'Your session has expired. Please try again.',
                ]);
            }
            
            return $response;
        });
    })
    ->create();
```

**Best Practices (Staff Engineer Tips)**

1. **Validation in Form Requests**: Always use Form Requests for non-trivial validation. Controllers with inline validation become bloated and hard to test. Form requests are self-contained, testable, and reusable. They also support authorization, which keeps controllers even cleaner.

2. **Fail Fast Validation**: Configure validation to stop on first failure for expensive validation scenarios. Use `bail` rule or `stopOnFirstFailure()` for API endpoints where detailed error arrays aren't needed. For forms where users need all errors at once, use the default all-errors behavior.

3. **After-Validation Hooks**: Use the `after` hook on validators for complex validation that depends on multiple fields. Cross-field validation (start date before end date, password confirmation, total order amount) belongs in after hooks, not individual field rules.

4. **Exception Hierarchy Design**: Create a hierarchy of application-specific exceptions. `BusinessException` as a base, with specific exceptions like `InsufficientFundsException` and `ProductUnavailableException` extending it. Handle the base exception globally and specific exceptions individually when needed.

**Common Pitfalls and Solutions**

1. **Missing Validation on Array Inputs**: Validating `array` type without validating array elements lets malicious data through. Always validate array structures fully: check array shape, validate each element with wildcard rules (`items.*.product_id`), and ensure no unexpected keys exist.

2. **Over-trusting Client-Side Validation**: Client-side validation is for user experience only. Never rely on it for security. Attackers bypass client validation easily. Server-side validation is the only security boundary. Duplicate critical validation on both sides.

3. **Generic Error Messages for Security**: Revealing specific information in error messages aids attackers. "Email not found" vs "Invalid credentials" tells attackers which emails are registered. Use generic "Invalid credentials" for all authentication failures. Similarly, avoid exposing table names, SQL errors, or file paths in production error responses.

4. **Not Handling ValidationException in APIs**: Throwing ValidationException in API controllers without custom rendering returns an HTML redirect response. Always configure the exception handler to return JSON error responses for API routes. Use `$request->expectsJson()` or check the request path.

---

 