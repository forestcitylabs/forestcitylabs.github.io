# Environment Configuration

The standard install uses environment variables and PHP configuration for managing application settings across different environments.

## Environment Variables

### Basic Setup

The standard install uses environment variables for configuration. Copy `.env.dist` to `.env` and customize:

```bash
cp .env.dist .env
```

### Required Variables

Configure the three required environment variables:

```env
# Environment (development, testing, production)
ENVIRONMENT=development

# Database
DATABASE_URI="mysql://user:password@localhost:3306/database_name"
# Or: DATABASE_URI="postgresql://user:password@localhost:5432/database_name"
# Or: DATABASE_URI="sqlite:///path/to/database.db"

# Trusted host for security
TRUSTED_HOST=localhost
```

### Additional Variables

While the framework only requires the three environment variables above, developers may add additional environment variables as needed for their specific application requirements:

```env
# JWT Authentication (optional)
JWT_SECRET=your-secret-key-here

# Caching (optional)
REDIS_URI=redis://localhost:6379
CACHE_DRIVER=redis

# Email (optional)
SMTP_HOST=smtp.example.com
SMTP_USER=your-email@example.com
SMTP_PASS=your-password

# API Keys (optional)
STRIPE_SECRET_KEY=sk_test_...
SENDGRID_API_KEY=SG...
```

## Environment-Specific Settings

### Development Environment

```env
# Development settings
ENVIRONMENT=development
DATABASE_URI="mysql://root:password@localhost:3306/myapp_dev"
TRUSTED_HOST=localhost

# Development-specific features
DEBUG_MODE=true
LOG_LEVEL=debug
CACHE_DRIVER=array
```

### Testing Environment

```env
# Testing settings
ENVIRONMENT=testing
DATABASE_URI="sqlite:///:memory:"
TRUSTED_HOST=localhost

# Testing-specific features
LOG_LEVEL=error
CACHE_DRIVER=array
DISABLE_EMAILS=true
```

### Production Environment

```env
# Production settings
ENVIRONMENT=production
DATABASE_URI="mysql://user:secure_password@db-server:3306/myapp_prod"
TRUSTED_HOST=yourdomain.com

# Production-specific features
LOG_LEVEL=warning
CACHE_DRIVER=redis
REDIS_URI=redis://redis-server:6379
```

## PHP Configuration

### Session Configuration

Configure session settings through PHP configuration:

```php
// Configure session settings in your bootstrap or service configuration
ini_set('session.cookie_lifetime', 3600);
ini_set('session.cookie_secure', true);
ini_set('session.cookie_httponly', true);
ini_set('session.cookie_samesite', 'Strict');
```

### Development Settings

```php
// Development-specific PHP settings
if ($_ENV['ENVIRONMENT'] === 'development') {
    ini_set('display_errors', 1);
    ini_set('error_reporting', E_ALL);
    ini_set('log_errors', 1);
    ini_set('error_log', __DIR__ . '/var/log/php_errors.log');
}
```

### Production Settings

```php
// Production-specific PHP settings
if ($_ENV['ENVIRONMENT'] === 'production') {
    ini_set('display_errors', 0);
    ini_set('log_errors', 1);
    ini_set('error_log', '/var/log/php/errors.log');
    
    // OpCache settings
    ini_set('opcache.enable', 1);
    ini_set('opcache.memory_consumption', 128);
    ini_set('opcache.interned_strings_buffer', 8);
    ini_set('opcache.max_accelerated_files', 4000);
    ini_set('opcache.revalidate_freq', 60);
    ini_set('opcache.fast_shutdown', 1);
}
```

## Environment Detection

### Automatic Detection

The framework automatically detects the environment from the `ENVIRONMENT` variable:

```php
$environment = $_ENV['ENVIRONMENT'] ?? 'production';

switch ($environment) {
    case 'development':
        // Development configuration
        break;
    case 'testing':
        // Testing configuration
        break;
    default:
        // Production configuration (default)
        break;
}
```

### Manual Environment Files

Create separate environment files for different deployments:

```bash
# Create environment-specific files
cp .env.dist .env.development
cp .env.dist .env.staging
cp .env.dist .env.production
```

Load the appropriate file based on your deployment:

```php
// Load environment-specific file
$env_file = '.env.' . ($_SERVER['APP_ENV'] ?? 'production');
if (file_exists(__DIR__ . '/../' . $env_file)) {
    $dotenv = Dotenv\Dotenv::createImmutable(__DIR__ . '/..', $env_file);
} else {
    $dotenv = Dotenv\Dotenv::createImmutable(__DIR__ . '/..');
}
$dotenv->load();
```

## Security Considerations

### Environment Variable Security

1. **Never commit `.env` files** to version control
2. **Use strong secrets** in production
3. **Rotate secrets regularly**
4. **Limit environment variable access** to necessary processes

```bash
# Add .env to .gitignore
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore
echo "!.env.dist" >> .gitignore
```

### Secret Management

For production environments, consider using dedicated secret management:

```php
// Example: Using AWS Secrets Manager
if ($_ENV['ENVIRONMENT'] === 'production') {
    $secrets_client = new Aws\SecretsManager\SecretsManagerClient([
        'version' => 'latest',
        'region' => 'us-east-1'
    ]);
    
    $secret = $secrets_client->getSecretValue(['SecretId' => 'myapp/database']);
    $database_credentials = json_decode($secret['SecretString'], true);
    
    $_ENV['DATABASE_URI'] = sprintf(
        'mysql://%s:%s@%s:%d/%s',
        $database_credentials['username'],
        $database_credentials['password'],
        $database_credentials['host'],
        $database_credentials['port'],
        $database_credentials['database']
    );
}
```

## Validation

### Environment Variable Validation

Validate required environment variables at startup:

```php
class EnvironmentValidator
{
    public static function validate(): void
    {
        $required = ['ENVIRONMENT', 'DATABASE_URI', 'TRUSTED_HOST'];
        
        foreach ($required as $var) {
            if (empty($_ENV[$var])) {
                throw new \RuntimeException("Required environment variable {$var} is not set");
            }
        }
        
        // Validate environment value
        $valid_environments = ['development', 'testing', 'production'];
        if (!in_array($_ENV['ENVIRONMENT'], $valid_environments)) {
            throw new \RuntimeException("Invalid ENVIRONMENT value: " . $_ENV['ENVIRONMENT']);
        }
        
        // Validate database URI format
        if (!filter_var($_ENV['DATABASE_URI'], FILTER_VALIDATE_URL)) {
            throw new \RuntimeException("Invalid DATABASE_URI format");
        }
    }
}

// Validate on application start
EnvironmentValidator::validate();
```

## Troubleshooting

### Common Issues

#### Environment Variables Not Loading

```bash
# Check if .env file exists
ls -la .env

# Check file permissions
chmod 644 .env

# Verify dotenv is loading the file
var_dump($_ENV); // Should contain your variables
```

#### Database Connection Issues

```bash
# Test database connection with environment variable
./vendor/bin/console doctrine:dbal:run-sql "SELECT 1"
```

#### Trusted Host Mismatches

```bash
# Check current host configuration
echo "Trusted host: " . $_ENV['TRUSTED_HOST']
echo "Current host: " . $_SERVER['HTTP_HOST']
```