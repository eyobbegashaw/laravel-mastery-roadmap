### Detailed Conceptual Overview (70%)

Middleware provides a mechanism for inspecting and filtering HTTP requests entering your application. Think of middleware as layers of security and processing that every request must pass through before reaching your application's core logic. Understanding middleware deeply enables you to write cleaner controllers and more secure applications.

**The Middleware Pipeline Architecture**

Middleware operates on the pipeline or onion pattern. The request enters the outer layer, passes through each middleware layer, reaches the application core (your route handler), then the response travels back through each layer in reverse order. Each middleware can act on the request before passing it inward and on the response before returning it outward.

This layered architecture creates clean separation of concerns. Authentication middleware verifies identity without controllers knowing about the auth mechanism. CORS middleware handles cross-origin requests transparently. Rate limiting middleware protects expensive endpoints without controller involvement. Each concern lives in its own isolated layer.

**Global Middleware**

Global middleware runs on every HTTP request, regardless of route or method. Laravel includes essential global middleware: `TrustProxies` for load balancer support, `HandleCors` for cross-origin requests, `PreventRequestsDuringMaintenance` for maintenance mode, `TrimStrings` to clean user input, and `ConvertEmptyStringsToNull` for database consistency.

Custom global middleware should be lightweight since it processes every request, even for static assets (though server configuration typically bypasses PHP for static files). Use global middleware for cross-cutting concerns like HTTPS enforcement, security headers, and request logging that apply universally.

**Route Middleware Groups**

Middleware groups bundle related middleware under a single name. The "web" group includes session handling, CSRF protection, and cookie encryption—features needed by browser-based applications. The "api" group includes rate limiting and is stateless—no sessions or CSRF needed for token-authenticated API requests.

Laravel 11 configures middleware groups in `bootstrap/app.php`. Groups enable reusable stacks—every web route gets session handling without explicitly listing each middleware. Custom groups can bundle application-specific requirements like localization detection, organization context setting for multi-tenant apps, or feature flag checking.

**Route-Specific Middleware**

Individual routes or route groups can specify middleware that runs only for those routes. A route handling file uploads might add content validation middleware. An admin section adds role verification middleware. The `middleware()` method accepts middleware class names, aliases, or parameters: `->middleware('auth', 'can:create-post')`.

Middleware parameters enable dynamic behavior. The `throttle:uploads,5,10` middleware limits upload routes to 5 attempts per 10 minutes. The `can:create,App\Models\Post` middleware checks policy authorization before the controller runs. Parameters make middleware reusable across different contexts with different configurations.

**Creating Custom Middleware**

Custom middleware encapsulates application-specific request processing. A `RedirectIfAuthenticated` middleware prevents logged-in users from seeing login pages. A `CheckUserRole` middleware verifies authorization before admin routes. A `DetectUserLocale` middleware sets the application language based on user preferences.

Middleware can modify the request by adding attributes. A middleware resolving the current organization might attach it to the request: `$request->attributes->set('organization', $organization)`. Controllers retrieve it without database queries. This pattern reduces duplicate queries across multiple controllers.

**Terminable Middleware**

Middleware implementing `TerminableMiddleware` runs after the response has been sent to the browser. This enables deferred processing—operations that shouldn't delay the user's response. Logging, metrics collection, and cache warming are ideal for termination handling.

Terminable middleware requires the FastCGI `fastcgi_finish_request` function, available in PHP-FPM environments. Without it, termination runs synchronously before the response is sent. For environments without FastCGI, use queued jobs for deferred processing instead.

**Middleware Priority**

Laravel processes middleware in a specific order defined by the priority list. Higher priority middleware runs first on the request journey (outer layer) and last on the response journey. Session start must run before authentication since auth stores the user in the session. CSRF validation must run after session start since tokens are session-bound.

Custom middleware may need specific priority positioning. A tenant resolution middleware should run before authentication so auth scopes queries to the correct tenant database. A localization middleware should run before request validation so validation messages appear in the correct language.

### Production-Ready Code Snippets (30%)

**1. Comprehensive Custom Middleware Examples**
```php
<?php
// app/Http/Middleware/EnforceHttps.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnforceHttps
{
    /**
     * Redirect HTTP requests to HTTPS in production.
     */
    public function handle(Request $request, Closure $next): Response
    {
        if (!$request->secure() && app()->environment('production')) {
            return redirect()->secure($request->getRequestUri(), 301);
        }

        return $next($request);
    }
}

// app/Http/Middleware/CheckSubscription.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckSubscription
{
    /**
     * Ensure user has an active subscription.
     * Fails with 402 Payment Required status.
     */
    public function handle(Request $request, Closure $next)
    {
        $user = $request->user();

        if (!$user) {
            return redirect()->route('login');
        }

        if (!$user->hasActiveSubscription()) {
            if ($request->expectsJson()) {
                return response()->json([
                    'message' => 'Active subscription required.',
                    'upgrade_url' => route('billing.plans')
                ], 402);
            }

            return redirect()
                ->route('billing.plans')
                ->with('warning', 'Please subscribe to access this feature.');
        }

        return $next($request);
    }
}

// app/Http/Middleware/SetOrganizationContext.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Models\Organization;

class SetOrganizationContext
{
    /**
     * Set the current organization context for multi-tenant app.
     * Resolves from subdomain or session.
     */
    public function handle(Request $request, Closure $next)
    {
        // Resolve organization from subdomain (acme.yourapp.com)
        $subdomain = explode('.', $request->getHost())[0];
        
        $organization = Organization::where('subdomain', $subdomain)->first();

        if (!$organization && !in_array($subdomain, ['www', 'app'])) {
            abort(404, 'Organization not found.');
        }

        if ($organization) {
            // Attach to request for controllers
            $request->attributes->set('organization', $organization);
            
            // Set default database connection for this tenant
            config(['database.connections.tenant.database' => "tenant_{$organization->id}"]);
        }

        return $next($request);
    }
}
```

**2. Middleware with Parameters**
```php
<?php
// app/Http/Middleware/CheckUserRole.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckUserRole
{
    /**
     * Verify user has one of the specified roles.
     * Usage: ->middleware('role:admin,moderator')
     */
    public function handle(Request $request, Closure $next, string ...$roles)
    {
        if (!$request->user()) {
            abort(401, 'Authentication required.');
        }

        // Check if user has any of the required roles
        foreach ($roles as $role) {
            if ($request->user()->hasRole($role)) {
                return $next($request);
            }
        }

        abort(403, 'Insufficient permissions. Required roles: ' . implode(', ', $roles));
    }
}

// app/Http/Middleware/DynamicThrottle.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Cache\RateLimiter;

class DynamicThrottle
{
    public function __construct(private RateLimiter $limiter) {}

    /**
     * Rate limit with different limits per user plan.
     * Usage: ->middleware('dynamic.throttle:uploads')
     */
    public function handle(Request $request, Closure $next, string $action)
    {
        $user = $request->user();
        
        // Different limits based on subscription plan
        $limits = [
            'free' => ['attempts' => 5, 'decay' => 60],
            'pro' => ['attempts' => 30, 'decay' => 60],
            'business' => ['attempts' => 100, 'decay' => 60],
        ];

        $plan = $user?->subscription_plan ?? 'free';
        $limit = $limits[$plan];
        
        $key = "dynamic_throttle:{$action}:{$user?->id}:{$request->ip()}";

        if ($this->limiter->tooManyAttempts($key, $limit['attempts'])) {
            return response()->json([
                'message' => 'Too many requests. Upgrade your plan for higher limits.',
                'retry_after' => $this->limiter->availableIn($key),
            ], 429);
        }

        $this->limiter->hit($key, $limit['decay']);

        $response = $next($request);

        // Add rate limit headers to response
        return $response->withHeaders([
            'X-RateLimit-Limit' => $limit['attempts'],
            'X-RateLimit-Remaining' => $this->limiter->remaining($key, $limit['attempts']),
        ]);
    }
}
```

**3. Terminable Middleware for Performance Monitoring**
```php
<?php
// app/Http/Middleware/PerformanceMonitor.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;
use Illuminate\Contracts\Foundation\Application;

class PerformanceMonitor
{
    public function __construct(private Application $app) {}

    /**
     * Track request performance metrics.
     */
    public function handle(Request $request, Closure $next): Response
    {
        $start = microtime(true);
        $startMemory = memory_get_usage(true);

        return $next($request);
    }

    /**
     * Runs after response is sent. Doesn't delay user response.
     */
    public function terminate(Request $request, Response $response): void
    {
        // These metrics would typically go to a monitoring service
        $duration = microtime(true) - $request->attributes->get('request_start', microtime(true));
        $memory = memory_get_usage(true) - $request->attributes->get('request_memory', memory_get_usage(true));
        
        // Log slow requests (over 1 second)
        if ($duration > 1.0) {
            \Log::warning('Slow request detected', [
                'url' => $request->fullUrl(),
                'method' => $request->method(),
                'duration_ms' => round($duration * 1000),
                'memory_mb' => round($memory / 1024 / 1024, 2),
                'user_id' => $request->user()?->id,
            ]);
        }

        // Store metrics for analysis
        if ($this->app->environment('production')) {
            \App\Jobs\StoreRequestMetric::dispatchAfterResponse([
                'url' => $request->fullUrl(),
                'route' => $request->route()?->getName(),
                'method' => $request->method(),
                'status' => $response->getStatusCode(),
                'duration' => $duration,
                'memory' => $memory,
                'timestamp' => now(),
            ]);
        }
    }
}
```

**4. Middleware Configuration in Laravel 11**
```php
<?php
// bootstrap/app.php
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(/* ... */)
    ->withMiddleware(function (Middleware $middleware) {
        
        // Global middleware (runs on every request)
        $middleware->use([
            \Illuminate\Http\Middleware\TrustProxies::class,
            \Illuminate\Http\Middleware\HandleCors::class,
            \App\Http\Middleware\EnforceHttps::class,
            \App\Http\Middleware\PerformanceMonitor::class,
        ]);

        // Append to existing web middleware group
        $middleware->web(append: [
            \App\Http\Middleware\SetOrganizationContext::class,
            \App\Http\Middleware\DetectUserLocale::class,
        ]);

        // Prepend to api middleware group
        $middleware->api(prepend: [
            \App\Http\Middleware\LogApiRequests::class,
        ]);

        // Custom middleware group
        $middleware->group('admin', [
            'auth',
            'verified',
            \App\Http\Middleware\CheckSubscription::class,
            \App\Http\Middleware\CheckUserRole::class . ':admin,super-admin',
        ]);

        // Middleware aliases with parameters support
        $middleware->alias([
            'role' => \App\Http\Middleware\CheckUserRole::class,
            'subscription' => \App\Http\Middleware\CheckSubscription::class,
            'dynamic.throttle' => \App\Http\Middleware\DynamicThrottle::class,
            'organization' => \App\Http\Middleware\SetOrganizationContext::class,
        ]);

        // Priority: who runs first
        $middleware->priority([
            \Illuminate\Session\Middleware\StartSession::class,
            \App\Http\Middleware\SetOrganizationContext::class,
            \Illuminate\Auth\Middleware\Authenticate::class,
            \App\Http\Middleware\CheckSubscription::class,
            \App\Http\Middleware\CheckUserRole::class,
        ]);
    })
    ->withExceptions(/* ... */)
    ->create();
```

**Best Practices (Staff Engineer Tips)**

1. **Thin Middleware, Thick Services**: Keep middleware focused on request/response transformation. Business logic belongs in service classes, not middleware. If middleware grows beyond 30 lines, extract logic to a dedicated class that the middleware calls. Middleware should orchestrate, not compute.

2. **Early Failure Pattern**: Check failure conditions first and return early. Don't process requests that will fail later. Authentication middleware that redirects unauthenticated users prevents controllers from running authorization checks. Rate limiting middleware that returns 429 prevents expensive operations from executing.

3. **Middleware Ordering Strategy**: Document why each middleware has its position in the priority list. Session start must precede authentication. Authentication must precede authorization checks. Tenant resolution should run before any database queries. Wrong ordering creates subtle bugs that are hard to diagnose.

4. **Avoid State in Middleware**: Middleware shouldn't mutate request state in ways that affect business logic. Adding attributes to requests is acceptable. Modifying request data in ways that change business outcomes creates hidden dependencies. Controllers should explicitly transform input, not rely on middleware transformations.

**Common Pitfalls and Solutions**

1. **Heavy Global Middleware**: Adding database queries to global middleware causes an extra query on every request, including asset requests. An organization resolution middleware querying the database adds latency to every page load. Cache organization lookups in global middleware or resolve from session data set during authentication.

2. **Middleware that Swallows Exceptions**: Catching all exceptions in middleware prevents proper error reporting. Middleware should catch specific exceptions it knows how to handle. Let unexpected exceptions propagate to the exception handler. Use the `$exceptions->renderable()` method for global exception formatting instead.

3. **Session Access in API Middleware**: API routes don't start sessions by default. Middleware that accesses `$request->session()` on API routes causes session initialization for every API call. Mark such middleware as web-only or check session availability before access.

4. **CSRF Protection in API Routes**: Adding API routes to the web middleware group subjects them to CSRF verification. API clients don't maintain sessions or receive CSRF tokens, so every request fails. API routes belong in `routes/api.php` with the api middleware group. Use Sanctum's SPA authentication for cookie-based API auth with CSRF protection.

---

 