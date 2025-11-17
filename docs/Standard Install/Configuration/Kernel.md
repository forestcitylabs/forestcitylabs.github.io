# Kernel and Middleware Configuration

The Kernel manages the HTTP request/response lifecycle through a configurable middleware pipeline. This guide shows how to configure the kernel and middleware using the KernelFactory pattern.

## KernelFactory Setup

### Creating a KernelFactory

The KernelFactory provides a centralized way to configure middleware and services for your application. Create a `KernelFactory.php` file to manage your application setup:

```php
<?php

namespace Application;

use ForestCityLabs\Framework\Kernel;
use Psr\EventDispatcher\EventDispatcherInterface;
use Psr\Container\ContainerInterface;

class KernelFactory
{
    public static function create(ContainerInterface $container): Kernel
    {
        $event_dispatcher = $container->get(EventDispatcherInterface::class);
        $kernel = new Kernel($event_dispatcher);
        
        // Configure middleware pipeline
        self::configureMiddleware($kernel, $container);
        
        return $kernel;
    }
    
    private static function configureMiddleware(Kernel $kernel, ContainerInterface $container): void
    {
        $environment = $_ENV['ENVIRONMENT'] ?? 'production';
        
        // Security middleware (first priority)
        $kernel->addMiddleware($container->get('middleware.allowed_host'));
        $kernel->addMiddleware($container->get('middleware.cors'));
        
        // Development-specific middleware
        if ($environment === 'development') {
            $kernel->addMiddleware($container->get('middleware.whoops'));
        }
        
        // Session and authentication
        $kernel->addMiddleware($container->get('middleware.session'));
        $kernel->addMiddleware($container->get('middleware.authentication'));
        
        // Request processing
        $kernel->addMiddleware($container->get('middleware.routing'));
        $kernel->addMiddleware($container->get('middleware.graphql'));
        
        // Error handling (last priority)
        $kernel->addMiddleware($container->get('middleware.not_found'));
    }
}
```

### Using the KernelFactory

Update your `public/index.php` to use the KernelFactory:

```php
<?php

require_once __DIR__ . '/../vendor/autoload.php';

use Application\KernelFactory;
use GuzzleHttp\Psr7\ServerRequest;
use Http\Response\Sender\ResponseSender;

// Load environment variables
$dotenv = Dotenv\Dotenv::createImmutable(__DIR__ . '/..');
$dotenv->load();

// Create dependency injection container
$container_builder = new DI\ContainerBuilder();
$container_builder->addDefinitions(__DIR__ . '/../config/services.php');
$container = $container_builder->build();

// Create kernel using factory
$kernel = KernelFactory::create($container);

// Handle request
$request = ServerRequest::fromGlobals();
$response = $kernel->handle($request);

// Send response
$sender = new ResponseSender();
$sender->send($response);
```

## Middleware Configuration

### Core Middleware Services

Configure middleware services in your `config/services.php` file:

```php
<?php

use ForestCityLabs\Framework\Middleware\AllowedHostMiddleware;
use ForestCityLabs\Framework\Middleware\CorsMiddleware;
use ForestCityLabs\Framework\Middleware\SessionMiddleware;
use ForestCityLabs\Framework\Middleware\RoutingMiddleware;
use ForestCityLabs\Framework\Middleware\GraphQLMiddleware;
use ForestCityLabs\Framework\Middleware\NotFoundMiddleware;

return [
    // Security middleware
    'middleware.allowed_host' => DI\factory(function (ContainerInterface $c) {
        return new AllowedHostMiddleware($_ENV['TRUSTED_HOST']);
    }),
    
    'middleware.cors' => DI\factory(function (ContainerInterface $c) {
        return new CorsMiddleware([
            'allowed_origins' => ['*'],
            'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
            'allowed_headers' => ['Content-Type', 'Authorization'],
            'max_age' => 3600
        ]);
    }),
    
    // Session middleware
    'middleware.session' => DI\factory(function (ContainerInterface $c) {
        return new SessionMiddleware([
            'name' => 'FCLS_SESSION',
            'lifetime' => 3600,
            'path' => '/',
            'domain' => null,
            'secure' => $_ENV['ENVIRONMENT'] === 'production',
            'httponly' => true,
            'samesite' => 'Lax'
        ]);
    }),
    
    // Request processing middleware
    'middleware.routing' => DI\factory(function (ContainerInterface $c) {
        return new RoutingMiddleware(
            $c->get('router'),
            $c->get('response_factory')
        );
    }),
    
    'middleware.graphql' => DI\factory(function (ContainerInterface $c) {
        return new GraphQLMiddleware(
            $c->get('graphql.schema'),
            $c->get('graphql.executor'),
            $c->get('response_factory')
        );
    }),
    
    // Error handling middleware
    'middleware.not_found' => DI\factory(function (ContainerInterface $c) {
        return new NotFoundMiddleware($c->get('response_factory'));
    }),
];
```

### Authentication Middleware

Configure authentication middleware with dependencies:

```php
return [
    'middleware.authentication' => DI\factory(function (ContainerInterface $c) {
        return new AuthenticationMiddleware(
            $c->get('auth.user_repository'),
            $c->get('auth.token_validator'),
            $c->get('logger')
        );
    }),
    
    'auth.user_repository' => DI\factory(function (ContainerInterface $c) {
        return new UserRepository(
            $c->get('doctrine.connection'),
            $c->get('cache')
        );
    }),
    
    'auth.token_validator' => DI\factory(function (ContainerInterface $c) {
        return new JwtTokenValidator([
            'secret' => $_ENV['JWT_SECRET'] ?? 'development-secret',
            'algorithm' => 'HS256',
            'leeway' => 60
        ]);
    }),
];
```

### Custom Middleware

Create and configure custom middleware:

```php
// src/Middleware/RateLimitMiddleware.php
class RateLimitMiddleware implements MiddlewareInterface
{
    public function __construct(
        private CacheItemPoolInterface $cache,
        private int $limit = 100,
        private int $window = 3600
    ) {}
    
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $client_ip = $request->getServerParams()['REMOTE_ADDR'] ?? 'unknown';
        $cache_key = "rate_limit_{$client_ip}";
        
        $item = $this->cache->getItem($cache_key);
        $requests = $item->isHit() ? $item->get() : 0;
        
        if ($requests >= $this->limit) {
            return new Response(429, [], 'Rate limit exceeded');
        }
        
        $item->set($requests + 1);
        $item->expiresAfter($this->window);
        $this->cache->save($item);
        
        return $handler->handle($request);
    }
}

// Configure in services.php
'middleware.rate_limit' => DI\factory(function (ContainerInterface $c) {
    return new RateLimitMiddleware(
        $c->get('cache'),
        $_ENV['RATE_LIMIT_REQUESTS'] ?? 100,
        $_ENV['RATE_LIMIT_WINDOW'] ?? 3600
    );
}),
```

## Environment-Specific Configuration

### Development Configuration

Create development-specific kernel configuration:

```php
private static function createDevelopmentKernel(ContainerInterface $container): Kernel
{
    $kernel = new Kernel($container->get(EventDispatcherInterface::class));
    
    // Development middleware stack
    $kernel->addMiddleware($container->get('middleware.allowed_host'));
    $kernel->addMiddleware($container->get('middleware.whoops')); // Detailed errors
    $kernel->addMiddleware($container->get('middleware.cors'));
    $kernel->addMiddleware($container->get('middleware.session'));
    $kernel->addMiddleware($container->get('middleware.routing'));
    $kernel->addMiddleware($container->get('middleware.graphql'));
    $kernel->addMiddleware($container->get('middleware.not_found'));
    
    return $kernel;
}
```

### Production Configuration

Create production-specific kernel configuration:

```php
private static function createProductionKernel(ContainerInterface $container): Kernel
{
    $kernel = new Kernel($container->get(EventDispatcherInterface::class));
    
    // Production middleware stack
    $kernel->addMiddleware($container->get('middleware.allowed_host'));
    $kernel->addMiddleware($container->get('middleware.cors'));
    $kernel->addMiddleware($container->get('middleware.cache')); // Response caching
    $kernel->addMiddleware($container->get('middleware.rate_limit')); // Rate limiting
    $kernel->addMiddleware($container->get('middleware.session'));
    $kernel->addMiddleware($container->get('middleware.authentication'));
    $kernel->addMiddleware($container->get('middleware.routing'));
    $kernel->addMiddleware($container->get('middleware.graphql'));
    $kernel->addMiddleware($container->get('middleware.error_handler')); // Generic errors
    $kernel->addMiddleware($container->get('middleware.not_found'));
    
    return $kernel;
}
```

### Environment Selection

Choose configuration based on environment:

```php
class KernelFactory
{
    public static function create(ContainerInterface $container): Kernel
    {
        $environment = $_ENV['ENVIRONMENT'] ?? 'production';
        
        switch ($environment) {
            case 'development':
                return self::createDevelopmentKernel($container);
            case 'testing':
                return self::createTestingKernel($container);
            default:
                return self::createProductionKernel($container);
        }
    }
}
```

## Advanced Middleware Configuration

### Conditional Middleware

Add middleware conditionally based on request properties:

```php
class ConditionalMiddlewareFactory
{
    public static function addApiMiddleware(Kernel $kernel, ContainerInterface $container): void
    {
        $kernel->addMiddleware(new ConditionalMiddleware(
            $container->get('middleware.api_auth'),
            fn($request) => str_starts_with($request->getUri()->getPath(), '/api/')
        ));
    }
    
    public static function addAdminMiddleware(Kernel $kernel, ContainerInterface $container): void
    {
        $kernel->addMiddleware(new ConditionalMiddleware(
            $container->get('middleware.admin_auth'),
            fn($request) => str_starts_with($request->getUri()->getPath(), '/admin/')
        ));
    }
}
```

### Middleware Groups

Organize middleware into reusable groups:

```php
class MiddlewareGroup
{
    private array $middleware = [];
    
    public function add(MiddlewareInterface $middleware): self
    {
        $this->middleware[] = $middleware;
        return $this;
    }
    
    public function applyTo(Kernel $kernel): void
    {
        foreach ($this->middleware as $middleware) {
            $kernel->addMiddleware($middleware);
        }
    }
}

// Usage
$api_group = new MiddlewareGroup();
$api_group->add($container->get('middleware.cors'))
          ->add($container->get('middleware.bearer_auth'))
          ->add($container->get('middleware.rate_limit'));

$web_group = new MiddlewareGroup();
$web_group->add($container->get('middleware.session'))
          ->add($container->get('middleware.csrf'))
          ->add($container->get('middleware.flash'));

// Apply based on request
if (str_starts_with($request->getUri()->getPath(), '/api/')) {
    $api_group->applyTo($kernel);
} else {
    $web_group->applyTo($kernel);
}
```

### Middleware Prioritization

Configure middleware with explicit priorities:

```php
class PriorityMiddlewareConfig
{
    private const PRIORITIES = [
        'security' => 1000,
        'authentication' => 800,
        'routing' => 600,
        'caching' => 400,
        'error_handling' => 200,
    ];
    
    public static function configure(Kernel $kernel, ContainerInterface $container): void
    {
        $middleware_configs = [
            ['middleware' => 'middleware.allowed_host', 'priority' => self::PRIORITIES['security']],
            ['middleware' => 'middleware.cors', 'priority' => self::PRIORITIES['security']],
            ['middleware' => 'middleware.session', 'priority' => self::PRIORITIES['authentication']],
            ['middleware' => 'middleware.authentication', 'priority' => self::PRIORITIES['authentication']],
            ['middleware' => 'middleware.routing', 'priority' => self::PRIORITIES['routing']],
            ['middleware' => 'middleware.cache', 'priority' => self::PRIORITIES['caching']],
            ['middleware' => 'middleware.not_found', 'priority' => self::PRIORITIES['error_handling']],
        ];
        
        // Sort by priority (highest first)
        usort($middleware_configs, fn($a, $b) => $b['priority'] <=> $a['priority']);
        
        foreach ($middleware_configs as $config) {
            $kernel->addMiddleware($container->get($config['middleware']));
        }
    }
}
```

## Router Configuration

### Route Registration

Configure routing with multiple route files:

```php
'router' => DI\factory(function (ContainerInterface $c) {
    $router = new \FastRoute\RouterDefinition();
    
    // Load routes from multiple files
    $route_files = [
        __DIR__ . '/../routes/api.php',
        __DIR__ . '/../routes/web.php',
        __DIR__ . '/../routes/admin.php',
    ];
    
    foreach ($route_files as $file) {
        if (file_exists($file)) {
            $routes = require $file;
            foreach ($routes as $route) {
                $router->addRoute(
                    $route['method'],
                    $route['pattern'],
                    $route['handler']
                );
            }
        }
    }
    
    return $router;
}),
```

### Route Caching

Enable route caching for production:

```php
'router' => DI\factory(function (ContainerInterface $c) {
    $cache_enabled = $_ENV['ENVIRONMENT'] === 'production';
    $cache_file = __DIR__ . '/../var/cache/routes.cache';
    
    if ($cache_enabled && file_exists($cache_file)) {
        return require $cache_file;
    }
    
    $router = new \FastRoute\RouterDefinition();
    // ... configure routes
    
    if ($cache_enabled) {
        file_put_contents($cache_file, '<?php return ' . var_export($router, true) . ';');
    }
    
    return $router;
}),
```

## Error Handling Configuration

### Development Error Handler

Configure detailed error handling for development:

```php
'middleware.whoops' => DI\factory(function (ContainerInterface $c) {
    $whoops = new \Whoops\Run();
    
    if (php_sapi_name() === 'cli') {
        $whoops->pushHandler(new \Whoops\Handler\PlainTextHandler());
    } else {
        $whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler());
    }
    
    return new WhoopsMiddleware($whoops);
}),
```

### Production Error Handler

Configure generic error handling for production:

```php
'middleware.error_handler' => DI\factory(function (ContainerInterface $c) {
    return new ErrorHandlerMiddleware(
        $c->get('response_factory'),
        $c->get('logger'),
        $_ENV['ENVIRONMENT'] === 'development'
    );
}),
```

## Performance Optimization

### Middleware Caching

Cache expensive middleware operations:

```php
'middleware.cached_auth' => DI\factory(function (ContainerInterface $c) {
    $base_auth = $c->get('middleware.authentication');
    return new CachedAuthenticationMiddleware(
        $base_auth,
        $c->get('cache'),
        300 // 5 minutes
    );
}),
```

### Lazy Middleware Loading

Load middleware lazily for better performance:

```php
'middleware.expensive' => DI\factory(function (ContainerInterface $c) {
    return new ExpensiveMiddleware(
        $c->get('heavy.dependency')
    );
})->lazy(),
```

## Testing Configuration

### Test Kernel Factory

Create a simplified kernel for testing:

```php
class TestKernelFactory
{
    public static function create(ContainerInterface $container): Kernel
    {
        $kernel = new Kernel($container->get(EventDispatcherInterface::class));
        
        // Minimal middleware for testing
        $kernel->addMiddleware($container->get('middleware.routing'));
        $kernel->addMiddleware($container->get('middleware.not_found'));
        
        return $kernel;
    }
}
```

### Mock Middleware

Create mock middleware for testing:

```php
'middleware.mock_auth' => DI\factory(function (ContainerInterface $c) {
    return new MockAuthenticationMiddleware([
        'user_id' => 123,
        'roles' => ['user', 'admin']
    ]);
}),
```

## Best Practices

1. **Order matters** - Security middleware first, error handling last
2. **Environment-specific** - Different middleware stacks for different environments
3. **Use factories** - Centralize kernel configuration in factory classes
4. **Test thoroughly** - Test middleware configurations with integration tests
5. **Cache in production** - Enable caching for routes and expensive operations
6. **Monitor performance** - Profile middleware execution times
7. **Keep it simple** - Avoid unnecessary middleware layers
8. **Document order** - Clearly document middleware execution order