# Service Configuration

The FCL Framework uses PHP-DI for dependency injection and service configuration. Services are defined in `config/services.php` and can be customized for your application needs.

## Basic Service Configuration

### Service Definition File

Services are configured in `config/services.php`:

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
            'url' => $_ENV['DATABASE_URI']
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

### Service Types

#### Factory Services

Factory services are created each time they're requested:

```php
'service.name' => DI\factory(function (ContainerInterface $c) {
    return new MyService(
        $c->get('dependency1'),
        $c->get('dependency2')
    );
}),
```

#### Singleton Services

Singleton services are created once and reused:

```php
'service.name' => DI\factory(function (ContainerInterface $c) {
    return new MyService();
})->singleton(),
```

#### Autowired Services

Autowired services automatically resolve constructor dependencies:

```php
MyService::class => DI\autowire(),
```

#### Value Services

Value services store static values:

```php
'config.timeout' => 30,
'config.api_key' => $_ENV['API_KEY'],
```

## Core Framework Services

### Event System

Configure the event dispatcher and listener provider:

```php
return [
    'listener_provider' => DI\factory(function (ContainerInterface $c) {
        return new ForestCityLabs\Framework\Event\ListenerProvider(
            new ForestCityLabs\Framework\Utility\ClassDiscovery\ScanDirectoryDiscovery(
                __DIR__ . '/../src/EventListener'
            ),
            $c->get('cache'),
            $c
        );
    }),
    
    'event_dispatcher' => DI\factory(function (ContainerInterface $c) {
        return new \League\Event\EventDispatcher(
            $c->get('listener_provider')
        );
    }),
];
```

### Caching Services

Configure different cache drivers:

```php
// Redis Cache
'cache' => DI\factory(function () {
    $redis = new \Predis\Client($_ENV['REDIS_URI'] ?? 'tcp://localhost:6379');
    return new \ForestCityLabs\Framework\Cache\Pool\PredisCachePool($redis);
}),

// Database Cache
'cache' => DI\factory(function (ContainerInterface $c) {
    return new \ForestCityLabs\Framework\Cache\Pool\DbalCachePool(
        $c->get('doctrine.connection'),
        'cache_items'
    );
}),

// File Cache
'cache' => DI\factory(function () {
    return new \ForestCityLabs\Framework\Cache\Pool\FileCachePool(
        __DIR__ . '/../var/cache'
    );
}),

// Array Cache (development/testing)
'cache' => DI\factory(function () {
    return new \ForestCityLabs\Framework\Cache\Pool\ArrayCachePool();
}),
```

### Logging Services

Configure logging with Monolog:

```php
'logger' => DI\factory(function () {
    $logger = new \Monolog\Logger('app');
    
    if ($_ENV['ENVIRONMENT'] === 'development') {
        $logger->pushHandler(new \Monolog\Handler\StreamHandler('php://stdout', \Monolog\Logger::DEBUG));
    } else {
        $logger->pushHandler(new \Monolog\Handler\RotatingFileHandler(
            __DIR__ . '/../var/log/app.log',
            0,  // Keep all files
            \Monolog\Logger::INFO
        ));
    }
    
    return $logger;
}),
```

## Application Services

### Repository Services

Configure repository classes with dependencies:

```php
return [
    UserRepository::class => DI\factory(function (ContainerInterface $c) {
        return new UserRepository(
            $c->get('doctrine.connection'),
            $c->get('cache'),
            $c->get('logger')
        );
    }),
    
    ProductRepository::class => DI\autowire(),
];
```

### Business Logic Services

Configure service classes:

```php
return [
    UserService::class => DI\factory(function (ContainerInterface $c) {
        return new UserService(
            $c->get(UserRepository::class),
            $c->get('event_dispatcher'),
            $c->get('logger')
        );
    }),
    
    EmailService::class => DI\factory(function (ContainerInterface $c) {
        return new EmailService(
            $_ENV['SMTP_HOST'],
            $_ENV['SMTP_USER'],
            $_ENV['SMTP_PASS'],
            $c->get('logger')
        );
    }),
];
```

### External API Services

Configure third-party API clients:

```php
return [
    'stripe.client' => DI\factory(function () {
        \Stripe\Stripe::setApiKey($_ENV['STRIPE_SECRET_KEY']);
        return new \Stripe\StripeClient($_ENV['STRIPE_SECRET_KEY']);
    }),
    
    'sendgrid.client' => DI\factory(function () {
        return new \SendGrid($_ENV['SENDGRID_API_KEY']);
    }),
    
    'aws.s3' => DI\factory(function () {
        return new \Aws\S3\S3Client([
            'version' => 'latest',
            'region' => $_ENV['AWS_REGION'] ?? 'us-east-1',
            'credentials' => [
                'key' => $_ENV['AWS_ACCESS_KEY_ID'],
                'secret' => $_ENV['AWS_SECRET_ACCESS_KEY'],
            ],
        ]);
    }),
];
```

## Environment-Specific Services

### Development Services

Services specific to development environment:

```php
if ($_ENV['ENVIRONMENT'] === 'development') {
    $services['debug.toolbar'] = DI\factory(function () {
        return new DebugToolbar();
    });
    
    $services['profiler'] = DI\factory(function () {
        return new Profiler();
    });
}
```

### Production Services

Services specific to production environment:

```php
if ($_ENV['ENVIRONMENT'] === 'production') {
    $services['monitoring.client'] = DI\factory(function () {
        return new MonitoringClient($_ENV['MONITORING_API_KEY']);
    });
    
    $services['cache'] = DI\factory(function () {
        // Use Redis in production
        $redis = new \Predis\Client($_ENV['REDIS_URI']);
        return new \ForestCityLabs\Framework\Cache\Pool\PredisCachePool($redis);
    });
}
```

## Advanced Configuration

### Conditional Services

Register services conditionally:

```php
return [
    'payment.processor' => DI\factory(function (ContainerInterface $c) {
        $provider = $_ENV['PAYMENT_PROVIDER'] ?? 'stripe';
        
        switch ($provider) {
            case 'stripe':
                return new StripePaymentProcessor($c->get('stripe.client'));
            case 'paypal':
                return new PayPalPaymentProcessor($c->get('paypal.client'));
            default:
                throw new \InvalidArgumentException("Unknown payment provider: {$provider}");
        }
    }),
];
```

### Service Decorators

Decorate existing services with additional functionality:

```php
return [
    'user.service.base' => DI\factory(function (ContainerInterface $c) {
        return new UserService($c->get(UserRepository::class));
    }),
    
    UserService::class => DI\factory(function (ContainerInterface $c) {
        $base_service = $c->get('user.service.base');
        
        // Add caching decorator
        $cached_service = new CachedUserService($base_service, $c->get('cache'));
        
        // Add logging decorator
        return new LoggedUserService($cached_service, $c->get('logger'));
    }),
];
```

### Service Tags

Group related services using tags:

```php
return [
    'event.listeners' => DI\factory(function (ContainerInterface $c) {
        return [
            $c->get(UserCreatedListener::class),
            $c->get(EmailNotificationListener::class),
            $c->get(LoggingListener::class),
        ];
    }),
    
    UserCreatedListener::class => DI\autowire()->addTag('event.listener'),
    EmailNotificationListener::class => DI\autowire()->addTag('event.listener'),
    LoggingListener::class => DI\autowire()->addTag('event.listener'),
];
```

## Testing Configuration

### Test Services

Override services for testing:

```php
// config/services.test.php
return [
    'cache' => DI\factory(function () {
        return new \ForestCityLabs\Framework\Cache\Pool\ArrayCachePool();
    }),
    
    'logger' => DI\factory(function () {
        return new \Monolog\Logger('test', [
            new \Monolog\Handler\NullHandler()
        ]);
    }),
    
    EmailService::class => DI\factory(function () {
        return new MockEmailService();
    }),
];
```

### Container Builder for Tests

Create test-specific container:

```php
public function createTestContainer(): ContainerInterface
{
    $builder = new ContainerBuilder();
    $builder->addDefinitions(__DIR__ . '/../config/services.php');
    $builder->addDefinitions(__DIR__ . '/../config/services.test.php');
    
    return $builder->build();
}
```

## Performance Optimization

### Container Compilation

Compile the container for production:

```php
// config/container.php
$builder = new ContainerBuilder();
$builder->addDefinitions(__DIR__ . '/services.php');

if ($_ENV['ENVIRONMENT'] === 'production') {
    $builder->enableCompilation(__DIR__ . '/../var/cache/container');
    $builder->writeProxiesToFile(true, __DIR__ . '/../var/cache/proxies');
}

return $builder->build();
```

### Lazy Services

Use lazy loading for expensive services:

```php
return [
    ExpensiveService::class => DI\factory(function (ContainerInterface $c) {
        return new ExpensiveService(
            $c->get('heavy.dependency')
        );
    })->lazy(),
];
```

## Debugging Services

### Service Inspection

Debug service configuration:

```php
// Check if service is registered
if ($container->has('service.name')) {
    $service = $container->get('service.name');
    var_dump($service);
}

// List all services (PHP-DI specific)
if ($container instanceof \DI\Container) {
    $definitions = $container->getKnownEntryNames();
    foreach ($definitions as $name) {
        echo "Service: {$name}\n";
    }
}
```

### Service Profiling

Profile service creation times:

```php
$start = microtime(true);
$service = $container->get('expensive.service');
$end = microtime(true);

echo "Service creation took: " . ($end - $start) . " seconds\n";
```

## Best Practices

1. **Use interfaces** - Define services using interfaces for better testability
2. **Minimize dependencies** - Keep service constructors focused
3. **Environment-specific config** - Use different configurations per environment
4. **Lazy loading** - Use lazy services for expensive operations
5. **Service compilation** - Compile containers in production
6. **Testing overrides** - Provide test doubles for external services
7. **Documentation** - Document complex service configurations
8. **Validation** - Validate required configuration at startup