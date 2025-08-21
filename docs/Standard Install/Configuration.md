# Configuration

The standard install provides a pre-configured setup with sensible defaults. This guide explains how to customize the configuration for your needs.

## Environment Configuration

### Environment Variables

The standard install uses environment variables for configuration. Copy `.env.dist` to `.env` and customize:

```bash
cp .env.dist .env
```

### Database Configuration

Configure your database connection:

```env
# Database
DATABASE_URL="mysql://user:password@localhost:3306/database_name"

# Or for PostgreSQL
DATABASE_URL="postgresql://user:password@localhost:5432/database_name"

# Or for SQLite
DATABASE_URL="sqlite:///path/to/database.db"
```

### Application Settings

```env
# Application
APP_ENV=development
APP_DEBUG=true
APP_NAME="My FCL App"

# Security
APP_SECRET=your-secret-key-here
JWT_SECRET=your-jwt-secret-here

# Cache
CACHE_DRIVER=filesystem
CACHE_PATH=/tmp/cache

# Sessions  
SESSION_DRIVER=filesystem
SESSION_PATH=/tmp/sessions

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:3000,https://yourdomain.com
```

## Service Configuration

The standard install uses PHP-DI for dependency injection. Services are configured in `config/services.php`:

```php
<?php

use DI\ContainerBuilder;
use Psr\Container\ContainerInterface;

return [
    // PSR-7 HTTP Message factories
    'response_factory' => DI\factory(function () {
        return new \GuzzleHttp\Psr7\HttpFactory();
    }),
    
    // Database connection
    'doctrine.connection' => DI\factory(function (ContainerInterface $c) {
        return \Doctrine\DBAL\DriverManager::getConnection([
            'url' => $_ENV['DATABASE_URL']
        ]);
    }),
    
    // Event dispatcher
    'event_dispatcher' => DI\factory(function (ContainerInterface $c) {
        return new \League\Event\EventDispatcher(
            $c->get('listener_provider')
        );
    }),
    
    // Custom services
    MyService::class => DI\autowire(),
];
```

## Database Setup

### Running Migrations

The standard install includes Doctrine Migrations:

```bash
# Create migration
./vendor/bin/doctrine-migrations generate

# Run migrations  
./vendor/bin/doctrine-migrations migrate

# Check status
./vendor/bin/doctrine-migrations status
```

### Entity Configuration

Entities are stored in `src/Entity/` and use Doctrine attributes:

```php
<?php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'users')]
class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;
    
    #[ORM\Column(type: 'string', length: 255)]
    private string $name;
    
    #[ORM\Column(type: 'string', length: 255, unique: true)]
    private string $email;
    
    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;
    
    // Getters and setters...
}
```

## Security Configuration  

### OAuth Setup

Configure OAuth clients and scopes:

```php
// In your bootstrap file
use ForestCityLabs\Framework\Security\OAuth\OAuthScopeRegistry;

$scope_registry = new OAuthScopeRegistry();
$scope_registry->addScope('read', 'Read access to resources');  
$scope_registry->addScope('write', 'Write access to resources');
$scope_registry->addScope('admin', 'Administrative access');
```

### User Authentication

Implement the required interfaces:

```php
<?php

namespace Application\Repository;

use ForestCityLabs\Framework\Security\Repository\UserRepositoryInterface;
use ForestCityLabs\Framework\Security\Model\UserInterface;

class UserRepository implements UserRepositoryInterface
{
    public function find(string $identifier): ?UserInterface
    {
        // Find user by ID or username
        return $this->entityManager
                    ->getRepository(User::class)
                    ->findOneBy(['id' => $identifier]);
    }
    
    public function findByUsername(string $username): ?UserInterface  
    {
        return $this->entityManager
                    ->getRepository(User::class)
                    ->findOneBy(['username' => $username]);
    }
}
```

## GraphQL Configuration

### Type Discovery

GraphQL types are automatically discovered from the `src/GraphQL/` directory:

```php
// config/services.php
use ForestCityLabs\Framework\GraphQL\MetadataProvider;
use ForestCityLabs\Framework\Utility\ClassDiscovery\ScanDirectoryDiscovery;

return [
    'graphql.metadata_provider' => DI\factory(function (ContainerInterface $c) {
        return new MetadataProvider(
            new ScanDirectoryDiscovery(__DIR__ . '/../src/GraphQL'),
            $c->get('cache'),
            $c->get('logger')
        );
    }),
];
```

### Schema Caching

Enable schema caching in production:

```env
GRAPHQL_SCHEMA_CACHE=true
GRAPHQL_INTROSPECTION=false  # Disable in production
```

## Caching Configuration

### Cache Drivers

Configure cache storage:

```php
// For Redis
return [
    'cache' => DI\factory(function () {
        $redis = new \Predis\Client($_ENV['REDIS_URL'] ?? 'tcp://localhost:6379');
        return new \ForestCityLabs\Framework\Cache\Pool\PredisCachePool($redis);
    }),
];

// For Database
return [
    'cache' => DI\factory(function (ContainerInterface $c) {
        return new \ForestCityLabs\Framework\Cache\Pool\DbalCachePool(
            $c->get('doctrine.connection'),
            'cache_items'  // Table name
        );
    }),
];
```

### Create Cache Tables

```bash
# Create cache table for database caching
./vendor/bin/fcl cache:table:create
```

## Session Configuration

### Session Storage

Configure session storage:

```php
// For database sessions
return [
    'session_driver' => DI\factory(function (ContainerInterface $c) {
        return new \ForestCityLabs\Framework\Session\Driver\DbalSessionDriver(
            $c->get('doctrine.connection'),
            'sessions'  // Table name
        );
    }),
];

// For Redis sessions  
return [
    'session_driver' => DI\factory(function () {
        $redis = new \Predis\Client($_ENV['REDIS_URL'] ?? 'tcp://localhost:6379');
        return new \ForestCityLabs\Framework\Session\Driver\PredisSessionDriver($redis);
    }),
];
```

### Create Session Tables

```bash
# Create session table for database storage
./vendor/bin/fcl session:table:create
```

## Logging Configuration

Configure logging with Monolog:

```php
// config/services.php
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Handler\RotatingFileHandler;

return [
    'logger' => DI\factory(function () {
        $logger = new Logger('app');
        
        if ($_ENV['APP_ENV'] === 'development') {
            $logger->pushHandler(new StreamHandler('php://stdout', Logger::DEBUG));
        } else {
            $logger->pushHandler(new RotatingFileHandler(
                __DIR__ . '/../var/log/app.log',
                0,  // Keep all files
                Logger::INFO
            ));
        }
        
        return $logger;
    }),
];
```

## Docker Configuration

The standard install includes Docker support:

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:80"
    environment:
      - APP_ENV=development
      - DATABASE_URL=mysql://root:password@db:3306/app
    volumes:
      - .:/var/www/html
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: app
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  db_data:
```

### Running with Docker

```bash
# Start services
docker compose up -d

# Run migrations
docker compose exec app ./vendor/bin/doctrine-migrations migrate

# View logs
docker compose logs -f app

# Access container shell
docker compose exec app bash
```

## Performance Optimization

### Production Settings

For production environments:

```env
# Production environment
APP_ENV=production
APP_DEBUG=false

# Enable OpCache
PHP_OPCACHE_ENABLE=1
PHP_OPCACHE_VALIDATE_TIMESTAMPS=0

# Cache configuration
CACHE_DRIVER=redis
REDIS_URL=tcp://redis:6379

# Session configuration  
SESSION_DRIVER=redis

# GraphQL optimizations
GRAPHQL_SCHEMA_CACHE=true
GRAPHQL_INTROSPECTION=false
GRAPHQL_QUERY_COMPLEXITY_LIMIT=100
```

### Cache Warming

Warm caches after deployment:

```bash
# Clear and warm caches
./vendor/bin/fcl cache:clear
./vendor/bin/doctrine orm:generate-proxies
./vendor/bin/fcl graphql:validate-schema
```

## Development Tools

### Debugging

Enable debugging in development:

```env
APP_DEBUG=true
WHOOPS_ENABLED=true
```

### Code Quality

The standard install includes code quality tools:

```bash
# PHP CodeSniffer
./vendor/bin/phpcs

# Fix coding standards
./vendor/bin/phpcbf

# PHPUnit tests
./vendor/bin/phpunit

# PHPStan static analysis (if installed)
./vendor/bin/phpstan analyse
```

## Deployment

### Environment-Specific Builds

Create environment-specific configurations:

```bash
# Staging environment
cp .env.dist .env.staging

# Production environment  
cp .env.dist .env.production
```

### Build Process

```bash
#!/bin/bash
# deploy.sh

# Install dependencies (production only)
composer install --no-dev --optimize-autoloader

# Clear caches
./vendor/bin/fcl cache:clear

# Run database migrations
./vendor/bin/doctrine-migrations migrate --no-interaction

# Generate Doctrine proxies
./vendor/bin/doctrine orm:generate-proxies

# Set proper permissions
chmod -R 755 var/
chmod -R 777 var/cache/
chmod -R 777 var/log/
chmod -R 777 var/sessions/
```

## Troubleshooting

### Common Issues

#### Database Connection

```bash
# Test database connection
./vendor/bin/doctrine dbal:run-sql "SELECT 1"
```

#### Cache Issues  

```bash
# Clear all caches
./vendor/bin/fcl cache:clear
rm -rf var/cache/*
```

#### Permission Issues

```bash
# Fix file permissions
sudo chown -R www-data:www-data var/
sudo chmod -R 755 var/
```

#### GraphQL Schema Issues

```bash  
# Validate schema
./vendor/bin/fcl graphql:validate-schema

# Dump current schema
./vendor/bin/fcl graphql:dump-schema > current-schema.graphql
```