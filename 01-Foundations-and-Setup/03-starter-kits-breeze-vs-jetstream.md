### Detailed Conceptual Overview (70%)

Starter kits provide pre-built authentication and user management scaffolding, saving days of development time. Laravel offers two official options: Breeze (lightweight) and Jetstream (feature-rich). Understanding their architectural differences helps choose the right foundation for your application.

**Laravel Breeze: Minimal Authentication**

Breeze implements the simplest possible authentication system while following all Laravel best practices. It provides login, registration, password reset, email verification, and profile management. The code is deliberately minimal—controllers contain only essential logic, and views use Tailwind CSS with Blade components.

The Breeze philosophy is "generate and own." When you install Breeze, it publishes controllers, views, and routes to your application directories. You can then modify them freely since they become part of your codebase. This contrasts with invisible packages that hide implementation details.

Breeze uses the Token Guard for API authentication when the `--api` flag is selected. This guard checks API tokens from the `personal_access_tokens` table managed by Sanctum. Mobile applications use Bearer tokens, while SPAs can use cookie-based authentication for CSRF protection.

**Laravel Jetstream: Advanced Application Scaffolding**

Jetstream builds upon Fortify, a frontend-agnostic authentication backend. While Breeze publishes authentication controllers, Jetstream uses Fortify's action classes that handle authentication logic behind the scenes. Your application registers Fortify and configures its behavior through a service provider rather than modifying controller methods directly.

Beyond basic authentication, Jetstream adds team management, two-factor authentication, session management (viewing and logging out other sessions), API support via Sanctum, browser session management, and optional profile photo management. These features are complex to build correctly—Jetstream provides them tested and production-ready.

Jetstream's philosophy emphasizes configurability over code ownership. Instead of publishing controllers to modify, you configure Jetstream through its configuration file and service provider. Custom actions for password validation, profile photo processing, and user creation plug into Jetstream's pipeline without modifying framework code.

**Technology Stack Choices**

Both kits offer Blade with Alpine.js stacks providing server-side rendering with minimal JavaScript. Alpine handles dynamic behaviors like dropdowns, modals, and form validation without a full JavaScript framework. The Livewire stack replaces Alpine.js with Livewire components, keeping all logic in PHP while achieving dynamic behavior through AJAX calls. The Inertia.js stack uses Vue or React for the frontend while maintaining server-side routing.

**Architecture Comparison**

Breeze publishes routes to `routes/auth.php` and controllers to `app/Http/Controllers/Auth/`. You see exactly how authentication works and can customize every aspect. Jetstream routes are included through a package service provider, and controllers live in the vendor directory. You customize behavior through Fortify's configuration and action classes.

For API authentication, Breeze with API creates API routes and tokens directly. Jetstream integrates Sanctum for both cookie-based SPA authentication and token-based mobile API authentication. Jetstream also supports first-party client configuration for SPA domains.

### Production-Ready Code Snippets (30%)

**1. Installing Starter Kits**
```bash
# Breeze with Blade and Alpine.js (simplest stack)
composer require laravel/breeze --dev
php artisan breeze:install blade
php artisan migrate
npm install && npm run build

# Breeze with API support for mobile apps
php artisan breeze:install api
# Creates routes/api.php with auth endpoints

# Breeze with Inertia and Vue
php artisan breeze:install vue --inertia

# Jetstream with Livewire (full-featured)
composer require laravel/jetstream
php artisan jetstream:install livewire --teams --api
php artisan migrate
npm install && npm run build

# Jetstream with Inertia and React
php artisan jetstream:install inertia --stack=react --teams
```

**2. Customizing Jetstream User Registration**
```php
<?php
// app/Actions/Fortify/CreateNewUser.php
namespace App\Actions\Fortify;

use App\Models\User;
use App\Models\Team;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Laravel\Jetstream\Events\AddingTeam;
use Laravel\Jetstream\Jetstream;

class CreateNewUser
{
    /**
     * Validate and create a newly registered user.
     * This action class is called by Fortify during registration.
     */
    public function create(array $input): User
    {
        // Custom validation with business-specific rules
        Validator::make($input, [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => $this->passwordRules(),
            'terms' => ['required', 'accepted'],
            'company_name' => ['required', 'string', 'max:255'],
        ])->validate();

        return \DB::transaction(function () use ($input) {
            $user = User::create([
                'name' => $input['name'],
                'email' => $input['email'],
                'password' => Hash::make($input['password']),
            ]);

            // Custom team creation with additional data
            if (Jetstream::hasTeamFeatures()) {
                AddingTeam::dispatch($user);

                $team = Team::forceCreate([
                    'user_id' => $user->id,
                    'name' => $input['company_name'],
                    'personal_team' => false,
                ]);

                $user->ownedTeams()->save($team);
                $user->switchTeam($team);
            }

            return $user;
        });
    }
}
```

**3. Customizing Inertia Middleware for Authentication Data**
```php
<?php
// app/Http/Middleware/HandleInertiaRequests.php
namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    /**
     * Define the props shared with all Inertia pages.
     * This runs on every request for Inertia-rendered pages.
     */
    public function share(Request $request): array
    {
        return [
            ...parent::share($request),
            
            // Share auth information efficiently
            'auth' => [
                'user' => $request->user() ? [
                    'id' => $request->user()->id,
                    'name' => $request->user()->name,
                    'email' => $request->user()->email,
                    'avatar' => $request->user()->profile_photo_url,
                    'has_teams' => $request->user()->allTeams()->count() > 1,
                ] : null,
                'can' => [
                    'manage_teams' => $request->user()?->can('create', Team::class),
                    'impersonate' => $request->user()?->can('impersonate'),
                ],
            ],
            
            // Flash messages for user feedback
            'flash' => [
                'success' => $request->session()->get('success'),
                'error' => $request->session()->get('error'),
            ],
        ];
    }
}
```

**4. Breeze Authentication Controller Example**
```php
<?php
// app/Http/Controllers/Auth/AuthenticatedSessionController.php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\ValidationException;

class AuthenticatedSessionController extends Controller
{
    /**
     * Handle an incoming authentication request.
     * Breeze provides this controller for full customization.
     */
    public function store(LoginRequest $request): RedirectResponse
    {
        // Rate limiting is handled by Laravel's built-in throttle
        $request->authenticate();
        
        // Regenerate session to prevent session fixation attacks
        $request->session()->regenerate();
        
        // Custom: Track last login timestamp
        $request->user()->update(['last_login_at' => now()]);

        // Redirect based on user role
        return redirect()->intended(
            $request->user()->isAdmin() 
                ? route('admin.dashboard') 
                : route('dashboard')
        );
    }

    /**
     * Destroy an authenticated session.
     */
    public function destroy(Request $request): RedirectResponse
    {
        Auth::guard('web')->logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

**Best Practices (Staff Engineer Tips)**

1. **Starter Kit Selection Strategy**: Choose Breeze for projects where you need full control over authentication code and expect minimal customization. Choose Jetstream for SaaS applications needing teams, API tokens, and two-factor authentication out of the box. The complexity gap between them is significant—Jetstream includes over 50 additional components and classes.

2. **Stack Selection for Long-term Maintenance**: Blade with Alpine.js is easiest to maintain long-term—it requires no frontend build tooling knowledge beyond basic Vite configuration. Livewire is excellent for PHP developers avoiding JavaScript entirely. Inertia stacks require ongoing npm dependency management and JavaScript framework expertise.

3. **Customizing Without Fighting the Framework**: When using Jetstream, extend through Fortify actions and events rather than overriding vendor code. Listen for events like `Laravel\Jetstream\Events\TeamCreated` to add custom team setup logic. Use the `Jetstream::profilePhotoDisk()` method to store profile photos on cloud storage.

4. **API Token Management**: Breeze API tokens are simple personal access tokens. Jetstream provides token management UI with permissions scoping. For production APIs consumed by third parties, consider Passport for OAuth2 servers. For first-party mobile apps and SPAs, Sanctum is simpler and sufficient.

**Common Pitfalls and Solutions**

1. **Overcommitting to Jetstream Too Early**: Beginners often install Jetstream with teams and two-factor authentication for simple projects. This adds unnecessary complexity—database migrations for team tables, middleware for team scope, and UI components for team management. Start with Breeze and migrate to Jetstream only when you need its features.

2. **Inertia SSR Misconfiguration**: Inertia can run in server-side rendering mode for SEO benefits. This requires a Node.js server process running alongside PHP. Many developers enable SSR without understanding this requirement, causing production deployment issues. Keep SSR disabled unless you specifically need it and have the infrastructure.

3. **Mixing Authentication Guards**: A common error is creating API routes (`routes/api.php`) that use the `web` middleware group via copy-paste. API routes use `auth:sanctum` guard, while web routes use `auth` guard. Mixing these causes unexpected authentication failures. Keep API and web authentication entirely separate.

4. **Not Publishing Vendor Assets for Customization**: Developers sometimes directly modify files in `vendor/laravel/jetstream` to customize Jetstream. These changes are lost on every Composer update. Always use the framework's extension points: service providers, action classes, and configuration files for customization.

---
