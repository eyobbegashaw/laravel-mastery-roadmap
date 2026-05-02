### Detailed Conceptual Overview (70%)

Deployment transforms your locally developed application into a publicly accessible service. Laravel provides two primary deployment platforms: Forge for traditional server management and Vapor for serverless deployment on AWS Lambda. Understanding both enables you to choose the right infrastructure for your application's needs.

**Laravel Forge: Managed Server Infrastructure**

Forge abstracts away server provisioning and management. Instead of manually installing PHP, Nginx, MySQL, Redis, and configuring firewalls, Forge provisions servers on DigitalOcean, AWS, Linode, or other cloud providers with everything pre-configured for Laravel applications. Server creation takes minutes instead of hours.

Forge handles ongoing server maintenance: security updates, SSL certificate provisioning and renewal via Let's Encrypt, firewall configuration, and service monitoring. It manages cron job scheduling, queue worker supervision, and log rotation. Developers focus on code; Forge manages infrastructure.

The deployment workflow integrates with Git. Push to your repository's production branch, and Forge pulls changes, runs Composer, executes migrations, restarts queues, and reloads PHP-FPM. This automated deployment pipeline reduces human error and enables continuous deployment practices.

**Server Configuration Management**

Forge provisions servers with Nginx as the web server, PHP-FPM for request processing, and MySQL/PostgreSQL for databases. It configures PHP with optimal settings for Laravel: OPcache for opcode caching, appropriate memory limits, and required extensions. Nginx configuration follows Laravel best practices with proper URL rewriting, static file handling, and security headers.

Multiple projects can share a server or have dedicated resources. Forge supports load balancers for horizontal scaling. Database servers can be provisioned as managed services (AWS RDS, DigitalOcean Managed Databases) for better reliability and automated backups.

**Laravel Vapor: Serverless Laravel**

Vapor represents a paradigm shift in Laravel deployment. Instead of managing servers, Vapor deploys Laravel as serverless functions on AWS Lambda. There are no servers to provision, update, or maintain. Application scales automatically from zero to thousands of concurrent requests without configuration changes.

Under the hood, Vapor packages your Laravel application into a Lambda-compatible format. HTTP requests arrive through API Gateway, which invokes Lambda functions. Static assets deploy to S3 and serve through CloudFront CDN. Database operations use Aurora Serverless or RDS. File storage uses S3. Queues use SQS.

This architecture provides automatic scaling, pay-per-request pricing, and zero server maintenance. If your application receives no traffic, you pay nothing for compute resources. If traffic spikes 100x during a launch, Vapor handles it automatically without intervention.

**The Vapor Request Lifecycle**

When a request arrives, API Gateway (or Application Load Balancer) receives it and invokes a Lambda function container. Vapor creates a lightweight Laravel bootstrap optimized for Lambda's execution model. The container processes the request and returns a response. If no requests arrive for a few minutes, the container is frozen; subsequent requests trigger a "cold start" of a few hundred milliseconds.

Vapor optimizes for the Lambda model. Sessions use DynamoDB instead of files. Cache uses DynamoDB or ElastiCache. File uploads go directly to S3 with pre-signed URLs, bypassing Lambda entirely for large uploads. Queue workers are separate Lambda functions that process SQS messages.

**Choosing Between Forge and Vapor**

Forge suits applications with predictable traffic, complex server requirements, or budget constraints. Traditional servers provide consistent performance, full control over the environment, and fixed monthly costs. If your application has steady traffic and doesn't need instant scaling, Forge is simpler and often more cost-effective.

Vapor excels for applications with variable traffic patterns, microservice architectures, or teams wanting to eliminate server management. Startup applications that might go viral, seasonal businesses, and event-driven architectures benefit from serverless scaling. However, Vapor requires AWS expertise and has variable costs based on usage.

**Database Strategy Comparison**

Forge typically uses server-based databases: MySQL, PostgreSQL, or managed services. Databases run continuously with fixed resources. Backups, replication, and failover follow traditional patterns.

Vapor encourages serverless database options: Aurora Serverless or DynamoDB. These scale automatically and pause during inactivity. Traditional RDS databases also work with Vapor but lack the full serverless scaling benefit.

**Deployment Process Comparison**

Forge deployments run on your servers. Composer installs dependencies, NPM builds assets, Artisan runs migrations. Server resources (CPU, memory) are shared between deployment and application serving.

Vapor deployments build a deployment artifact locally or in CI/CD. All dependencies are packaged, assets are uploaded to S3, and Lambda functions are updated atomically. Deployments have zero downtime—new Lambda versions receive traffic only after successful deployment.

### Production-Ready Code Snippets (30%)

**1. Forge Deployment Script**
```bash
#!/bin/bash
# .forge-deploy - Forge deployment script

# Navigate to project directory
cd /home/forge/example.com

# Pull latest code
git pull origin main

# Install PHP dependencies
$FORGE_PHP composer install --no-dev --optimize-autoloader

# Install and build frontend assets
npm ci
npm run build

# Run database migrations (with forced flag for safety)
$FORGE_PHP artisan migrate --force

# Clear and cache
$FORGE_PHP artisan optimize

# Restart queue workers (managed by Supervisor)
$FORGE_PHP artisan queue:restart

# Clear OPcache
$FORGE_PHP artisan opcache:clear

# Reload PHP-FPM
sudo service php8.3-fpm reload

# Notify monitoring services
curl -fsS --retry 3 https://healthchecks.io/ping/your-check-id

echo "Deployment completed successfully!"
```

**2. Vapor Configuration**
```yaml
# vapor.yml
id: 12345
name: my-laravel-app
environments:
  production:
    # Domain configuration
    domain: example.com
    domain-certificate: arn:aws:acm:us-east-1:123456789012:certificate/xxx
    
    # Storage for static assets
    storage: my-app-assets
    
    # Build configuration
    build:
      - 'COMPOSER_MIRROR_PATH_REPOS=1 composer install --no-dev --optimize-autoloader'
      - 'npm ci && npm run build'
      - 'php artisan event:cache'
      - 'php artisan config:cache'
      - 'php artisan route:cache'
      - 'php artisan view:cache'
    
    # Deployment hooks
    deploy:
      - 'php artisan migrate --force'
    
    # Environment-specific configuration
    environment:
      APP_ENV: production
      APP_DEBUG: false
      LOG_CHANNEL: stderr
      SESSION_DRIVER: dynamodb
      CACHE_DRIVER: dynamodb
      
      # Database
      DB_CONNECTION: mysql
      DATABASE_URL: '${DATABASE_URL}'
      
      # Queue
      QUEUE_CONNECTION: sqs
      SQS_QUEUE: '${SQS_QUEUE_URL}'
      
      # Storage
      FILESYSTEM_DISK: s3
    
    # Memory allocation for Lambda
    memory: 2048
    
    # Concurrency settings
    concurrency: 100
    
    # Scheduled tasks (runs via CloudWatch Events)
    schedule:
      - command: 'payments:process-scheduled'
        frequency: 'everyFiveMinutes'
        timeout: 300
      - command: 'report:daily'
        frequency: 'dailyAt:06:00'
        timeout: 600
    
    # Queue workers
    queues:
      - name: default
        memory: 1024
        timeout: 300
        concurrency: 10
    
    # Database proxy (for VPC databases)
    database-proxy: my-db-proxy
    
    # CloudFront CDN configuration
    cdn:
      cache-policy: Managed-CachingOptimized
      price-class: '100'
      aliases:
        - cdn.example.com

  staging:
    domain: staging.example.com
    storage: my-app-staging-assets
    memory: 1024
    concurrency: 50
    environment:
      APP_ENV: staging
      APP_DEBUG: true
```

**3. Vapor-Specific Service Provider**
```php
<?php
// app/Providers/VaporServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\URL;
use Aws\DynamoDb\DynamoDbClient;
use Aws\Sqs\SqsClient;

class VaporServiceProvider extends ServiceProvider
{
    /**
     * Register Vapor-specific services.
     */
    public function register(): void
    {
        if (! $this->app->environment('vapor')) {
            return;
        }

        // Configure DynamoDB cache
        $this->app['config']->set('cache.stores.dynamodb', [
            'driver' => 'dynamodb',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
            'endpoint' => env('DYNAMODB_ENDPOINT'),
        ]);

        // Configure SQS queue
        $this->app['config']->set('queue.connections.sqs', [
            'driver' => 'sqs',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'prefix' => env('SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
            'queue' => env('SQS_QUEUE', 'default'),
            'suffix' => env('SQS_SUFFIX'),
            'region' => env('AWS_DEFAULT_REGION'),
            'after_commit' => true,
        ]);
    }

    /**
     * Bootstrap Vapor-specific services.
     */
    public function boot(): void
    {
        if (! $this->app->environment('vapor')) {
            return;
        }

        // Force HTTPS in production
        URL::forceScheme('https');

        // Configure S3 for file storage
        Storage::extend('s3-vapor', function ($app, $config) {
            // Vapor automatically injects S3 client
            return $app['s3']->createS3Driver($config);
        });

        // Handle static assets through CloudFront
        if ($assetUrl = env('ASSET_URL')) {
            $this->app['config']->set('app.asset_url', $assetUrl);
        }

        // Configure trusted proxies for load balancers
        if ($proxyHeader = $_SERVER['HTTP_X_FORWARDED_FOR'] ?? null) {
            $this->app['request']->setTrustedProxies(
                [$proxyHeader],
                \Illuminate\Http\Request::HEADER_X_FORWARDED_ALL
            );
        }

        // Warm up services on cold start
        $this->warmUpServices();
    }

    /**
     * Warm up services to reduce cold start time.
     */
    protected function warmUpServices(): void
    {
        // Pre-resolve commonly used services
        app('cache.store');
        app('db.connection');
    }
}
```

**4. CI/CD Pipeline for Forge Deployment**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch: # Manual trigger

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, pdo_mysql, bcmath
          coverage: none

      - name: Install Dependencies
        run: |
          cp .env.ci .env
          composer install --no-interaction --prefer-dist

      - name: Run Tests
        run: php artisan test --parallel

  deploy:
    name: Deploy to Forge
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Trigger Forge Deployment
        run: |
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.FORGE_API_TOKEN }}" \
            -H "Accept: application/json" \
            "https://forge.laravel.com/api/v1/servers/${{ secrets.FORGE_SERVER_ID }}/sites/${{ secrets.FORGE_SITE_ID }}/deploy"
```

**Best Practices (Staff Engineer Tips)**

1. **Infrastructure as Code**: Never manually configure servers. Use Forge's API, CloudFormation templates for AWS, or infrastructure-as-code tools. Server configurations must be reproducible. Manual server tweaks lead to configuration drift and deployment failures.

2. **Zero-Downtime Deployments**: Design deployment processes that never drop traffic. Forge uses PHP-FPM graceful reload. Vapor uses Lambda versioning. Database migrations should be backward-compatible. Use feature flags for large changes.

3. **Environment Parity**: Keep staging and production environments as similar as possible. Use the same Laravel version, PHP version, extensions, and server configuration. Environment-specific differences should be minimal and clearly documented.

4. **Deployment Automation**: Automate everything that happens after code merges. Run tests, build assets, deploy, run migrations, restart workers, verify health checks. Manual deployments cause inconsistencies and errors. CI/CD pipelines provide repeatable, auditable deployments.

**Common Pitfalls and Solutions**

1. **Running Migrations During Peak Traffic**: Database migrations can lock tables, causing application errors during deployment. Schedule deployments during low-traffic periods. Use tools like `pt-online-schema-change` for zero-downtime migrations on large tables.

2. **Not Running `php artisan optimize`**: Forgetting to cache configuration, routes, and views means 50-100ms of additional overhead per request. Include `php artisan optimize` in every deployment script. Verify cache creation by checking `bootstrap/cache` directory.

3. **Hardcoded Paths in Vapor**: Vapor's filesystem is ephemeral—files written during one request may not exist in the next. Never use `storage_path()` for persistent files in Vapor. Always use S3 for file storage. Sessions, cache, and logs must use external services.

4. **Ignoring Lambda Cold Starts**: Vapor's first request after inactivity takes longer (cold start). Optimize service provider loading. Use deferred providers where possible. Pre-warm critical endpoints after deployment. Monitor cold start duration.

---
