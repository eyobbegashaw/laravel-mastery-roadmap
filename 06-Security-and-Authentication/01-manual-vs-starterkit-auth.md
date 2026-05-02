### Detailed Conceptual Overview (70%)

Authentication is the gatekeeper of your application. Laravel provides multiple paths to implement it: manual authentication using the built-in guards and providers, or pre-built solutions through starter kits like Breeze and Jetstream. Understanding both approaches deeply allows you to choose the right solution for your project's requirements.

**The Authentication Foundation: Guards and Providers**

Laravel's authentication system rests on two pillars: guards and providers. Guards define how users are authenticated for each request—session-based for web applications, token-based for APIs. Providers define how users are retrieved from persistent storage—typically from a database using Eloquent.

The session guard maintains authentication state across requests using Laravel's session system. When a user logs in successfully, their user ID is stored in the session. On subsequent requests, the guard retrieves the user from the database using this ID. The token guard (used by Sanctum) checks for an API token in the request header or query string.

This separation allows multiple authentication methods in a single application. Web users authenticate via sessions, while API clients authenticate via tokens. Both can coexist because guards isolate authentication logic from the rest of your application.

**Manual Authentication: Building from Scratch**

Manual authentication gives you complete control over authentication logic. You create controllers, views, and routes for login, registration, password reset, email verification, and profile management. This approach is valuable when you have unique authentication requirements—custom login flows, multi-step verification, or integration with legacy user systems.

The `Auth` facade provides access to the authentication system. `Auth::attempt($credentials)` validates credentials and logs in the user. `Auth::login($user)` logs in a user instance directly (useful after registration or social login). `Auth::logout()` clears the session. These simple methods provide full authentication control.

Password hashing is handled automatically through Laravel's `Hash` facade. User models should implement the `Authenticatable` contract, which the default `User` model already does. The `Hash::check()` method verifies passwords against stored hashes using bcrypt by default, with argon2id available as a stronger alternative.

**Starter Kit Authentication: Accelerated Development**

Laravel Breeze provides a minimal, customizable authentication scaffold. It publishes controllers, views, and routes to your application, making them part of your codebase. You can modify them freely—Breeze serves as a starting point, not a black-box dependency. Breeze includes login, registration, password reset, email verification, and profile management.

The controllers published by Breeze demonstrate best practices: form request validation, rate limiting on login attempts, session regeneration after authentication, and proper CSRF protection. New developers learn correct patterns by reading Breeze's code. Experienced developers save time by starting with well-structured authentication code.

Laravel Jetstream extends authentication with team management, two-factor authentication, session management, and API token management. It uses Fortify under the hood—a frontend-agnostic authentication backend. Unlike Breeze, Jetstream's controllers live in the vendor directory; you customize behavior through Fortify's action classes and configuration.

**The Authenticate Middleware Chain**

Authentication involves several middleware layers working together. The `Authenticate` middleware verifies that a user is logged in, redirecting or returning 401 for unauthenticated access. The `RedirectIfAuthenticated` middleware prevents authenticated users from seeing login and registration pages. Together, they create the authentication flow users expect.

Session management is critical for security. After login, `session()->regenerate()` prevents session fixation attacks by creating a new session ID. After logout, both `session()->invalidate()` and `session()->regenerateToken()` clear all session data and create a fresh CSRF token.

**Rate Limiting Authentication Attempts**

Brute force protection is essential for authentication endpoints. Laravel's `ThrottlesLogins` trait, used in login controllers, limits failed login attempts. After too many failures from an IP address or username, further attempts are blocked for a cooling period. This prevents dictionary attacks and credential stuffing.

### Production-Ready Code Snippets (30%)

**1. Custom Manual Authentication Controller**
```php
<?php
// app/Http/Controllers/Auth/ManualAuthController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class ManualAuthController extends Controller
{
    /**
     * Show the login form.
     */
    public function showLoginForm(): View
    {
        return view('auth.login');
    }

    /**
     * Handle login attempt with comprehensive security measures.
     */
    public function login(Request $request): RedirectResponse
    {
        // Validate input
        $credentials = $request->validate([
            'email' => ['required', 'email', 'string'],
            'password' => ['required', 'string'],
        ]);

        // Add rate limiting (max 5 attempts per minute per IP + email)
        $throttleKey = 'login:' . $request->ip() . ':' . $request->input('email');
        
        if (app('cache')->has($throttleKey . ':locked')) {
            throw ValidationException::withMessages([
                'email' => 'Too many login attempts. Please try again in ' . 
                    app('cache')->get($throttleKey . ':locked') . ' seconds.',
            ]);
        }

        // Attempt authentication with remember me
        $remember = $request->boolean('remember');

        if (!Auth::attempt($credentials, $remember)) {
            // Increment failed attempts
            $attempts = app('cache')->increment($throttleKey, 1);
            
            if ($attempts >= 5) {
                app('cache')->put($throttleKey . ':locked', 300, 300); // 5 minutes lock
            }

            throw ValidationException::withMessages([
                'email' => __('auth.failed'),
            ]);
        }

        // Clear failed attempts on successful login
        app('cache')->forget($throttleKey);
        app('cache')->forget($throttleKey . ':locked');

        $request->session()->regenerate();

        // Log successful login for security audit
        activity()
            ->by(Auth::user())
            ->withProperties([
                'ip' => $request->ip(),
                'user_agent' => $request->userAgent(),
            ])
            ->log('User logged in');

        // Redirect based on user role or intended URL
        if (Auth::user()->isAdmin()) {
            return redirect()->intended(route('admin.dashboard'));
        }

        return redirect()->intended(route('dashboard'))
            ->with('success', 'Welcome back, ' . Auth::user()->first_name . '!');
    }

    /**
     * Handle user registration with email verification.
     */
    public function register(Request $request): RedirectResponse
    {
        $validated = $request->validate([
            'first_name' => ['required', 'string', 'max:100'],
            'last_name' => ['required', 'string', 'max:100'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'string', 'min:12', 'confirmed'],
            'terms' => ['required', 'accepted'],
        ]);

        $user = User::create([
            'first_name' => $validated['first_name'],
            'last_name' => $validated['last_name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
            'last_login_at' => now(),
        ]);

        // Assign default role
        $user->assignRole('customer');

        // Send email verification notification
        $user->sendEmailVerificationNotification();

        // Log the user in
        Auth::login($user);
        $request->session()->regenerate();

        return redirect()->route('verification.notice')
            ->with('success', 'Account created! Please verify your email address.');
    }

    /**
     * Logout with session cleanup.
     */
    public function logout(Request $request): RedirectResponse
    {
        // Log the logout event
        activity()
            ->by(Auth::user())
            ->log('User logged out');

        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/')
            ->with('success', 'You have been logged out successfully.');
    }
}
```

**2. Custom Login Form with Security Features**
```blade
{{-- resources/views/auth/login.blade.php --}}
@extends('layouts.guest')

@section('title', 'Login')

@section('content')
<div class="min-h-screen flex items-center justify-center bg-gray-50 py-12 px-4 sm:px-6 lg:px-8">
    <div class="max-w-md w-full space-y-8">
        {{-- Header --}}
        <div>
            <h2 class="mt-6 text-center text-3xl font-extrabold text-gray-900">
                Sign in to your account
            </h2>
            <p class="mt-2 text-center text-sm text-gray-600">
                Or 
                <a href="{{ route('register') }}" class="font-medium text-indigo-600 hover:text-indigo-500">
                    create a new account
                </a>
            </p>
        </div>

        {{-- Session status (for password reset success) --}}
        @if (session('status'))
            <div class="rounded-md bg-green-50 p-4">
                <div class="text-sm text-green-700">
                    {{ session('status') }}
                </div>
            </div>
        @endif

        <form class="mt-8 space-y-6" action="{{ route('login') }}" method="POST" novalidate>
            @csrf

            {{-- Honeypot field for bot detection --}}
            <div class="hidden" aria-hidden="true">
                <input type="text" name="website" tabindex="-1" autocomplete="off">
            </div>

            <div class="rounded-md shadow-sm -space-y-px">
                {{-- Email --}}
                <div>
                    <label for="email" class="sr-only">Email address</label>
                    <input id="email" 
                           name="email" 
                           type="email" 
                           autocomplete="email"
                           value="{{ old('email') }}"
                           required 
                           class="appearance-none rounded-none relative block w-full px-3 py-2 border border-gray-300 placeholder-gray-500 text-gray-900 rounded-t-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm @error('email') border-red-500 @enderror" 
                           placeholder="Email address">
                    @error('email')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>

                {{-- Password --}}
                <div>
                    <label for="password" class="sr-only">Password</label>
                    <input id="password" 
                           name="password" 
                           type="password" 
                           autocomplete="current-password"
                           required 
                           class="appearance-none rounded-none relative block w-full px-3 py-2 border border-gray-300 placeholder-gray-500 text-gray-900 rounded-b-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm @error('password') border-red-500 @enderror" 
                           placeholder="Password">
                    @error('password')
                        <p class="mt-1 text-sm text-red-600">{{ $message }}</p>
                    @enderror
                </div>
            </div>

            {{-- Remember me and forgot password --}}
            <div class="flex items-center justify-between">
                <div class="flex items-center">
                    <input id="remember" 
                           name="remember" 
                           type="checkbox" 
                           class="h-4 w-4 text-indigo-600 focus:ring-indigo-500 border-gray-300 rounded">
                    <label for="remember" class="ml-2 block text-sm text-gray-900">
                        Remember me
                    </label>
                </div>

                <div class="text-sm">
                    <a href="{{ route('password.request') }}" 
                       class="font-medium text-indigo-600 hover:text-indigo-500">
                        Forgot your password?
                    </a>
                </div>
            </div>

            {{-- Submit --}}
            <div>
                <button type="submit" 
                        class="group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500">
                    <span class="absolute left-0 inset-y-0 flex items-center pl-3">
                        <svg class="h-5 w-5 text-indigo-500 group-hover:text-indigo-400" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
                            <path fill-rule="evenodd" d="M5 9V7a5 5 0 0110 0v2a2 2 0 012 2v5a2 2 0 01-2 2H5a2 2 0 01-2-2v-5a2 2 0 012-2zm8-2v2H7V7a3 3 0 016 0z" clip-rule="evenodd" />
                        </svg>
                    </span>
                    Sign in
                </button>
            </div>
        </form>
    </div>
</div>
@endsection
```

**3. Custom Authentication Guard for Legacy System Integration**
```php
<?php
// app/Providers/AuthServiceProvider.php
namespace App\Providers;

use App\Services\LegacyAuthService;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Register custom user provider for legacy system
        Auth::provider('legacy', function ($app, array $config) {
            return new LegacyUserProvider(
                $app->make(LegacyAuthService::class)
            );
        });

        // Register custom guard using the legacy provider
        Auth::extend('legacy', function ($app, $name, array $config) {
            return new LegacyGuard(
                Auth::createUserProvider($config['provider']),
                $app->make('request')
            );
        });
    }
}

// app/Auth/LegacyUserProvider.php
namespace App\Auth;

use Illuminate\Contracts\Auth\UserProvider;
use Illuminate\Contracts\Auth\Authenticatable;
use App\Services\LegacyAuthService;

class LegacyUserProvider implements UserProvider
{
    public function __construct(
        private LegacyAuthService $legacyService
    ) {}

    /**
     * Retrieve a user by their unique identifier.
     */
    public function retrieveById($identifier): ?Authenticatable
    {
        $legacyUser = $this->legacyService->findUser($identifier);
        
        if (!$legacyUser) {
            return null;
        }

        return $this->mapToLaravelUser($legacyUser);
    }

    /**
     * Validate credentials with legacy system.
     */
    public function validateCredentials(Authenticatable $user, array $credentials): bool
    {
        return $this->legacyService->verifyCredentials(
            $credentials['email'],
            $credentials['password']
        );
    }

    /**
     * Map legacy user data to Laravel Authenticatable model.
     */
    private function mapToLaravelUser(array $legacyUser): Authenticatable
    {
        return new \App\Models\User([
            'id' => $legacyUser['user_id'],
            'email' => $legacyUser['email'],
            'name' => $legacyUser['full_name'],
            'legacy_token' => $legacyUser['auth_token'],
        ]);
    }

    // Other required interface methods...
    public function retrieveByToken($identifier, $token) {}
    public function updateRememberToken(Authenticatable $user, $token) {}
    public function retrieveByCredentials(array $credentials) {}
}
```

**Best Practices (Staff Engineer Tips)**

1. **Password Policy Enforcement**: Implement minimum password length (at least 12 characters), complexity requirements, and compromised password checking. Use the `HaveIBeenPwned` API through Laravel's `uncompromised` validation rule. Consider passwordless authentication or passkeys for modern applications.

2. **Session Security Configuration**: Configure `config/session.php` properly. Set `secure` to `true` (HTTPS only), `http_only` to `true` (prevent JavaScript access), and `same_site` to `'lax'` (CSRF protection). Reduce session lifetime for sensitive applications. Implement session fingerprinting with IP and user agent validation.

3. **Account Enumeration Prevention**: Return identical error messages for invalid email and invalid password. Don't reveal whether an account exists. Use identical timing for both responses to prevent timing attacks. Implement generic "Invalid credentials" messages.

4. **Multi-Factor Authentication Strategy**: Implement 2FA using TOTP (Time-based One-Time Password) for sensitive accounts. Use recovery codes for backup access. Consider WebAuthn/passkeys for phishing-resistant authentication. Jetstream provides 2FA out of the box.

**Common Pitfalls and Solutions**

1. **Storing Plain Text Passwords**: Never store passwords in plain text or using reversible encryption. Always use `Hash::make()` or the `hashed` cast on model attributes. Password reset tokens should also be hashed before storage.

2. **Session Fixation Vulnerability**: Always regenerate session IDs after authentication state changes (login, logout, privilege escalation). Laravel does this automatically in most cases, but custom authentication implementations might miss this critical security step.

3. **Missing Rate Limiting on Auth Routes**: Login, registration, password reset, and email verification endpoints must have rate limiting. Without it, attackers can brute force credentials or spam password reset emails. Laravel's `throttle` middleware provides easy protection.

4. **Insecure Remember Me Implementation**: Remember tokens must be long, random strings. They should rotate on every use and invalidate on password change. Laravel's built-in remember me handles this correctly, but custom "keep me logged in" implementations often get it wrong.

---
