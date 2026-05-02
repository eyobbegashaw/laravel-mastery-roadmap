### Detailed Conceptual Overview (70%)

API authentication is fundamentally different from session-based authentication. APIs are stateless by nature—each request must contain all information needed to authenticate. Laravel provides two official packages for API authentication: Sanctum for simpler token-based auth and SPA authentication, and Passport for full OAuth2 server implementation.

**Laravel Sanctum: Lightweight API Authentication**

Sanctum provides a featherweight authentication system for SPAs, mobile applications, and simple token-based APIs. It allows each user to generate multiple API tokens with specific abilities (scopes). These tokens are stored in the database and can be revoked individually.

For single-page applications (SPAs) on the same domain, Sanctum uses cookie-based authentication with CSRF protection—essentially session authentication that works with SPAs. The SPA receives a CSRF cookie, then sends it with API requests. This is more secure than storing tokens in localStorage, which is vulnerable to XSS attacks.

For mobile applications and third-party API access, Sanctum issues API tokens. These tokens are long-lived and passed in the `Authorization: Bearer` header. Sanctum's token abilities provide simple scoping—a token can have "create-posts" ability but not "delete-users" ability. This isn't full OAuth scoping but works for most use cases.

**Laravel Passport: Full OAuth2 Server**

Passport provides a complete OAuth2 server implementation. OAuth2 is the industry standard for authorization—it's what allows you to "Sign in with Google" on third-party websites. Passport supports multiple grant types: authorization code (for third-party apps), client credentials (for machine-to-machine), password grant (deprecated, for trusted clients), and personal access tokens.

Passport creates OAuth2 clients that represent applications requesting access. Each client has a client ID and secret. When a user authorizes a client, Passport issues access tokens and optional refresh tokens. Access tokens are short-lived (typically 1 hour), while refresh tokens can obtain new access tokens without user intervention.

The complexity of Passport is warranted when building platforms that allow third-party applications to access user data. If you're building an API consumed only by your own mobile app and SPA, Passport is overkill—Sanctum is the right choice.

**Token Abilities and Scopes**

Both Sanctum and Passport support abilities (Sanctum) or scopes (Passport) to limit what a token can do. A token created for a mobile app might have "read-profile" and "create-posts" abilities but not "delete-posts". Middleware checks token abilities before allowing access to protected endpoints.

Sanctum's ability middleware: `Route::get('/posts', [PostController::class, 'index'])->middleware('auth:sanctum', 'ability:read-posts')`. Passport uses similar middleware: `->middleware('scopes:read-posts')`.

**Token Lifecycle Management**

API tokens should have configurable lifetimes. Access tokens should be short-lived (minutes to hours). Refresh tokens can persist longer (days to weeks). Sanctum allows setting token expiration when creating tokens. Passport manages expiration through its configuration.

Token revocation is critical for security. Users must be able to view and revoke their tokens. Admins must be able to revoke tokens for compromised accounts. Both Sanctum and Passport provide token management capabilities through their respective APIs.

**SPA Authentication vs Token Authentication**

SPA authentication uses cookies, which are automatically sent with requests and protected from JavaScript access (httpOnly). This eliminates the risk of token theft through XSS. However, cookies only work on the same domain.

Token authentication uses the `Authorization` header with Bearer tokens. Tokens must be explicitly stored (preferably in memory or httpOnly cookies, not localStorage) and sent with requests. This works cross-domain but requires careful token storage security.

### Production-Ready Code Snippets (30%)

**1. Sanctum API Token Management**
```php
<?php
// app/Http/Controllers/Api/AuthController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    /**
     * Issue a new API token for mobile app authentication.
     */
    public function token(Request $request): JsonResponse
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required|string',
            'device_name' => 'required|string|max:100',
            'abilities' => 'nullable|array',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        // Check if user account is active
        if (!$user->is_active) {
            return response()->json([
                'message' => 'Your account has been deactivated. Please contact support.',
            ], 403);
        }

        // Delete existing tokens for this device (one token per device)
        $user->tokens()->where('name', $request->device_name)->delete();

        // Create new token with specific abilities
        $abilities = $request->input('abilities', [
            'orders:read',
            'orders:create',
            'profile:read',
            'profile:update',
        ]);

        $token = $user->createToken(
            $request->device_name,
            $abilities,
            now()->addDays(30) // Token expires in 30 days
        );

        // Log token creation
        activity()
            ->by($user)
            ->withProperties([
                'device' => $request->device_name,
                'abilities' => $abilities,
            ])
            ->log('API token created');

        return response()->json([
            'token' => $token->plainTextToken,
            'type' => 'Bearer',
            'expires_at' => $token->accessToken->expires_at,
            'abilities' => $abilities,
        ]);
    }

    /**
     * Revoke a specific token.
     */
    public function revokeToken(Request $request, $tokenId): JsonResponse
    {
        $token = $request->user()->tokens()->findOrFail($tokenId);
        $token->delete();

        return response()->json([
            'message' => 'Token revoked successfully.',
        ]);
    }

    /**
     * Revoke all tokens except current.
     */
    public function revokeOtherTokens(Request $request): JsonResponse
    {
        $request->user()->tokens()
            ->where('id', '!=', $request->user()->currentAccessToken()->id)
            ->delete();

        return response()->json([
            'message' => 'All other sessions have been terminated.',
        ]);
    }

    /**
     * List all active tokens for the authenticated user.
     */
    public function listTokens(Request $request): JsonResponse
    {
        $tokens = $request->user()->tokens()
            ->select('id', 'name', 'abilities', 'created_at', 'expires_at', 'last_used_at')
            ->get()
            ->map(function ($token) {
                return [
                    'id' => $token->id,
                    'device' => $token->name,
                    'abilities' => $token->abilities,
                    'created' => $token->created_at->diffForHumans(),
                    'expires' => $token->expires_at?->diffForHumans(),
                    'last_used' => $token->last_used_at?->diffForHumans(),
                ];
            });

        return response()->json(['tokens' => $tokens]);
    }
}

// routes/api.php
Route::prefix('v1')->group(function () {
    // Public auth routes (no authentication required)
    Route::post('/auth/token', [AuthController::class, 'token'])
        ->middleware('throttle:5,1') // Max 5 attempts per minute
        ->name('api.token');
    
    Route::post('/auth/register', [AuthController::class, 'register']);

    // Protected routes (require valid Sanctum token)
    Route::middleware('auth:sanctum')->group(function () {
        
        // Token management
        Route::get('/tokens', [AuthController::class, 'listTokens']);
        Route::delete('/tokens/{tokenId}', [AuthController::class, 'revokeToken']);
        Route::delete('/tokens', [AuthController::class, 'revokeOtherTokens']);
        
        // User profile
        Route::get('/user', fn (Request $request) => $request->user());
        
        // Abilities-based routes
        Route::middleware('ability:orders:read')->get('/orders', [OrderController::class, 'index']);
        Route::middleware('ability:orders:create')->post('/orders', [OrderController::class, 'store']);
    });
});
```

**2. Sanctum SPA Authentication Setup**
```php
<?php
// config/sanctum.php - SPA Configuration
return [
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
        '%s%s',
        'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
        env('APP_URL') ? ','.parse_url(env('APP_URL'), PHP_URL_HOST) : ''
    ))),

    'guard' => ['web'],
    
    'expiration' => null,
    
    'token_prefix' => env('SANCTUM_TOKEN_PREFIX', ''),
    
    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => App\Http\Middleware\EncryptCookies::class,
        'verify_csrf_token' => App\Http\Middleware\VerifyCsrfToken::class,
    ],
];

// app/Http/Kernel.php or bootstrap/app.php
// Sanctum's middleware for SPA authentication
$middleware->api(prepend: [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
]);

// config/cors.php - CORS configuration for SPA
return [
    'paths' => [
        'api/*',
        'sanctum/csrf-cookie',
        'login',
        'logout',
    ],
    'allowed_methods' => ['*'],
    'allowed_origins' => [env('FRONTEND_URL', 'http://localhost:3000')],
    'allowed_origins_patterns' => [],
    'allowed_headers' => ['*'],
    'exposed_headers' => [],
    'max_age' => 0,
    'supports_credentials' => true,
];
```

```javascript
// resources/js/api/client.js - SPA API Client using Axios
import axios from 'axios';

const apiClient = axios.create({
    baseURL: '/api/v1',
    withCredentials: true, // Send cookies with requests
    headers: {
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
    },
});

// Initialize CSRF cookie before any POST/PUT/DELETE requests
export const initializeCsrf = async () => {
    await axios.get('/sanctum/csrf-cookie');
};

// Add response interceptor for error handling
apiClient.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 401) {
            // Redirect to login on authentication failure
            window.location.href = '/login';
        }
        
        if (error.response?.status === 419) {
            // CSRF token mismatch - refresh and retry
            return initializeCsrf().then(() => apiClient.request(error.config));
        }
        
        return Promise.reject(error);
    }
);

export default apiClient;

// Usage in Vue component
import apiClient, { initializeCsrf } from '@/api/client';

export default {
    methods: {
        async login(credentials) {
            await initializeCsrf();
            await apiClient.post('/auth/login', credentials);
            // User is now authenticated via session cookie
        },
        
        async fetchOrders() {
            const response = await apiClient.get('/orders');
            return response.data;
        },
    },
};
```

**3. Passport OAuth2 Server Configuration**
```php
<?php
// config/passport.php (or in AppServiceProvider)
use Laravel\Passport\Passport;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Passport::tokensExpireIn(now()->addDays(15));
        Passport::refreshTokensExpireIn(now()->addDays(30));
        Passport::personalAccessTokensExpireIn(now()->addMonths(6));

        // Define available OAuth2 scopes
        Passport::tokensCan([
            'read-user' => 'Read your profile information',
            'write-user' => 'Update your profile information',
            'read-orders' => 'View your order history',
            'create-orders' => 'Create new orders',
            'admin:users' => 'Manage users (admin only)',
            'admin:orders' => 'Manage all orders (admin only)',
        ]);

        // Set default scope
        Passport::setDefaultScope([
            'read-user',
        ]);
    }
}

// app/Http/Controllers/Api/OAuthController.php
namespace App\Http\Controllers\Api;

use Laravel\Passport\Http\Controllers\AccessTokenController;
use Psr\Http\Message\ServerRequestInterface;

class OAuthController extends AccessTokenController
{
    /**
     * Custom token issuance with audit logging.
     */
    public function issueToken(ServerRequestInterface $request)
    {
        $response = parent::issueToken($request);
        
        // Log token issuance for audit trail
        if ($response->getStatusCode() === 200) {
            $data = json_decode($response->getContent(), true);
            
            \Log::info('OAuth token issued', [
                'client_id' => $request->getParsedBody()['client_id'] ?? 'unknown',
                'grant_type' => $request->getParsedBody()['grant_type'],
                'ip' => request()->ip(),
            ]);
        }
        
        return $response;
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Token Storage Security**: Never store API tokens in localStorage—it's vulnerable to XSS attacks. For SPAs, use httpOnly cookies with Sanctum's SPA authentication. For mobile apps, store tokens in secure storage (Keychain on iOS, EncryptedSharedPreferences on Android). For web apps requiring token-based auth, store tokens in memory and refresh on reload.

2. **Token Rotation Strategy**: Implement refresh token rotation. Each refresh token can only be used once; using it issues a new refresh token and invalidates the old one. If a stolen refresh token is used, the legitimate holder's next request invalidates all tokens, alerting to the compromise.

3. **Ability/Scope Granularity**: Design abilities around resources and actions. Use a consistent naming pattern: `resource:action`. Prefer fine-grained abilities that can be combined. A token with `orders:read,orders:create` can view and create but not delete. Validate abilities in middleware before reaching controllers.

4. **API Versioning and Deprecation**: Include API version in routes (`/api/v1/...`). When deprecating tokens, use the `last_used_at` timestamp to identify and notify users of inactive tokens. Automatically revoke tokens unused for 90 days. Communicate breaking changes well in advance.

**Common Pitfalls and Solutions**

1. **Overusing Passport for Simple APIs**: Passport adds significant complexity with OAuth2 flows, client management, and scope resolution. If you only need token authentication for your own mobile app and SPA, Sanctum is simpler and sufficient. Passport is for platforms where third-party applications access your API.

2. **Missing Token Expiration**: Issuing tokens without expiration creates permanent access. Always set appropriate expiration based on token type and sensitivity. Short-lived access tokens (1 hour) with refresh tokens provide better security. Personal access tokens can live longer but should still expire eventually.

3. **Insecure CORS Configuration**: Using `allowed_origins: ['*']` with `supports_credentials: true` is a security vulnerability—it allows any website to make authenticated requests. Always specify exact allowed origins when using credentials. Use environment variables for different origins per environment.

4. **Forgetting to Protect CSRF for SPA APIs**: Sanctum's SPA authentication requires CSRF protection. Forgetting the `/sanctum/csrf-cookie` request before POST/PUT/DELETE endpoints results in 419 errors. Frontend code must initialize CSRF before making state-changing requests.

---

