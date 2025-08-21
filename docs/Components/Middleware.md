# Middleware

The FCL Framework is built around a [PSR-15](https://www.php-fig.org/psr/psr-15/) compliant middleware pipeline that handles HTTP requests. Each middleware can process the request, modify it, delegate to the next middleware, and process the response.

## Overview

Middleware in the FCL Framework follows the standard PSR-15 pattern where each middleware:

1. Receives a request and request handler
2. Can modify the request before passing it on
3. Delegates to the next handler in the pipeline
4. Can modify the response before returning it

## Available Middleware

The framework provides many built-in middleware components:

### Core Middleware

- **RoutingMiddleware** - Routes requests to controllers
- **GraphQLMiddleware** - Handles GraphQL requests
- **GraphiQLMiddleware** - Provides GraphiQL interface
- **SessionMiddleware** - Manages sessions
- **FlashMiddleware** - Handles flash messages

### Security Middleware

- **BearerTokenAuthenticationMiddleware** - OAuth bearer token authentication
- **SessionAuthenticationMiddleware** - Session-based authentication
- **OAuthMiddleware** - OAuth 2.0 server endpoints
- **OidcMiddleware** - OpenID Connect server endpoints

### Utility Middleware

- **CorsMiddleware** - CORS headers handling
- **AllowedHostMiddleware** - Host validation
- **DefaultCacheMiddleware** - HTTP caching headers
- **GeneratedByMiddleware** - Adds framework identification headers
- **TrailingSlashMiddleware** - Handles trailing slash redirects

### Error Handling Middleware

- **HttpNotFoundMiddleware** - Returns 404 responses
- **HttpUnauthorizedMiddleware** - Returns 401 responses  
- **HttpForbiddenMiddleware** - Returns 403 responses
- **WhoopsMiddleware** - Development error pages
- **ReturnNotFoundMiddleware** - Fallback 404 handler
- **FallbackControllerMiddleware** - Fallback response handler

## Using Middleware

### Basic Usage

Add middleware to your kernel in order of execution:

```php
use ForestCityLabs\Framework\Kernel;

$kernel = new Kernel($event_dispatcher);

// Add middleware in execution order
$kernel->addMiddleware($cors_middleware);
$kernel->addMiddleware($session_middleware);
$kernel->addMiddleware($auth_middleware);
$kernel->addMiddleware($routing_middleware);
$kernel->addMiddleware($not_found_middleware);
```

### Middleware Order

The order of middleware is important. Here's a typical middleware stack:

1. **CORS** - Handle preflight requests early
2. **Allowed Host** - Validate allowed hosts
3. **Session** - Initialize sessions
4. **Authentication** - Authenticate users
5. **Routing** - Route to controllers
6. **GraphQL** - Handle GraphQL requests
7. **Error Handlers** - Handle various HTTP errors

## Built-in Middleware Details

### CORS Middleware

Handles Cross-Origin Resource Sharing:

```php
use ForestCityLabs\Framework\Middleware\CorsMiddleware;

$cors_middleware = new CorsMiddleware([
    'allowed_origins' => ['https://example.com', 'https://app.example.com'],
    'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    'allowed_headers' => ['Content-Type', 'Authorization'],
    'exposed_headers' => ['X-Total-Count'],
    'allow_credentials' => true,
    'max_age' => 86400, // 24 hours
]);
```

### Session Middleware

Manages HTTP sessions:

```php
use ForestCityLabs\Framework\Middleware\SessionMiddleware;
use ForestCityLabs\Framework\Session\Driver\FilesystemSessionDriver;

$session_driver = new FilesystemSessionDriver('/tmp/sessions');

$session_middleware = new SessionMiddleware(
    $session_driver,
    [
        'name' => 'PHPSESSID',
        'lifetime' => 3600, // 1 hour
        'path' => '/',
        'domain' => '',
        'secure' => true,    // HTTPS only
        'httponly' => true,  // No JavaScript access
        'samesite' => 'Lax'  // CSRF protection
    ]
);
```

### Authentication Middleware

#### Bearer Token Authentication

For API authentication using OAuth access tokens:

```php
use ForestCityLabs\Framework\Middleware\BearerTokenAuthenticationMiddleware;

$bearer_middleware = new BearerTokenAuthenticationMiddleware(
    $access_token_manager,  // AccessTokenManagerInterface
    $user_repository,       // UserRepositoryInterface
    $logger                 // LoggerInterface
);
```

The middleware extracts tokens from the `Authorization: Bearer <token>` header.

#### Session Authentication

For web application authentication using sessions:

```php
use ForestCityLabs\Framework\Middleware\SessionAuthenticationMiddleware;

$session_auth_middleware = new SessionAuthenticationMiddleware(
    $user_repository,       // UserRepositoryInterface
    $logger                 // LoggerInterface
);
```

### Cache Middleware

Adds HTTP caching headers:

```php
use ForestCityLabs\Framework\Middleware\DefaultCacheMiddleware;

$cache_middleware = new DefaultCacheMiddleware([
    'max_age' => 3600,           // Cache for 1 hour
    'must_revalidate' => true,   // Force revalidation
    'no_cache' => false,         // Allow caching
    'no_store' => false,         // Allow storage
    'private' => false,          // Allow public caching
]);
```

### Host Validation

Validates incoming requests against allowed hosts:

```php
use ForestCityLabs\Framework\Middleware\AllowedHostMiddleware;

$host_middleware = new AllowedHostMiddleware(
    ['example.com', 'www.example.com', 'api.example.com'],
    $response_factory
);
```

### Trailing Slash Handling

Redirects or normalizes trailing slashes:

```php
use ForestCityLabs\Framework\Middleware\TrailingSlashMiddleware;

$trailing_slash_middleware = new TrailingSlashMiddleware(
    $response_factory,
    'redirect',  // 'redirect', 'remove', or 'add'
    301          // Redirect status code
);
```

## Error Handling Middleware

### Development Error Pages

For development environments, use Whoops for detailed error pages:

```php
use ForestCityLabs\Framework\Middleware\WhoopsMiddleware;

if ($is_development) {
    $whoops_middleware = new WhoopsMiddleware();
    $kernel->addMiddleware($whoops_middleware);
}
```

### HTTP Error Responses

#### 404 Not Found

```php
use ForestCityLabs\Framework\Middleware\HttpNotFoundMiddleware;

$not_found_middleware = new HttpNotFoundMiddleware(
    $response_factory,
    $twig_environment,    // Optional: for custom 404 page
    '404.html.twig'       // Template name
);
```

#### 401 Unauthorized

```php
use ForestCityLabs\Framework\Middleware\HttpUnauthorizedMiddleware;

$unauthorized_middleware = new HttpUnauthorizedMiddleware(
    $response_factory,
    $twig_environment,    // Optional
    '401.html.twig'       // Template name
);
```

#### 403 Forbidden

```php
use ForestCityLabs\Framework\Middleware\HttpForbiddenMiddleware;

$forbidden_middleware = new HttpForbiddenMiddleware(
    $response_factory,
    $twig_environment,    // Optional
    '403.html.twig'       // Template name
);
```

## Custom Middleware

Create your own middleware by implementing `MiddlewareInterface`:

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class CustomMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        // Process request before delegation
        $request = $request->withAttribute('custom_data', 'value');
        
        // Delegate to next middleware
        $response = $handler->handle($request);
        
        // Process response before returning
        return $response->withHeader('X-Custom-Header', 'Custom Value');
    }
}
```

### Middleware with Dependencies

Use dependency injection for services:

```php
class LoggingMiddleware implements MiddlewareInterface
{
    public function __construct(
        private LoggerInterface $logger
    ) {}
    
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler  
    ): ResponseInterface {
        $start_time = microtime(true);
        
        $this->logger->info('Request started', [
            'method' => $request->getMethod(),
            'uri' => (string) $request->getUri()
        ]);
        
        $response = $handler->handle($request);
        
        $duration = microtime(true) - $start_time;
        
        $this->logger->info('Request completed', [
            'status' => $response->getStatusCode(),
            'duration' => $duration
        ]);
        
        return $response;
    }
}
```

## Conditional Middleware

Apply middleware conditionally based on request properties:

```php
class ConditionalMiddleware implements MiddlewareInterface
{
    public function __construct(
        private MiddlewareInterface $middleware
    ) {}
    
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        // Only apply middleware to API routes
        if (str_starts_with($request->getUri()->getPath(), '/api/')) {
            return $this->middleware->process($request, $handler);
        }
        
        return $handler->handle($request);
    }
}
```

## Middleware Events

The framework dispatches events before and after each middleware:

```php
use ForestCityLabs\Framework\Event\Attribute\EventListener;
use ForestCityLabs\Framework\Events\PreMiddlewareHandleEvent;
use ForestCityLabs\Framework\Events\PostMiddlewareHandleEvent;

#[EventListener(PreMiddlewareHandleEvent::class)]
class MiddlewareEventListener
{
    public function onPreHandle(PreMiddlewareHandleEvent $event): void
    {
        $middleware = $event->getMiddleware();
        $request = $event->getRequest();
        
        // Log or monitor middleware execution
    }
    
    #[EventListener(PostMiddlewareHandleEvent::class)]
    public function onPostHandle(PostMiddlewareHandleEvent $event): void
    {
        $middleware = $event->getMiddleware();
        $response = $event->getResponse();
        
        // Post-process response or log completion
    }
}
```

## Performance Considerations

### Middleware Ordering

- Place lightweight validation middleware early (CORS, Host validation)
- Place expensive middleware later (Database authentication, complex routing)
- Place error handlers last to catch all exceptions

### Caching Middleware Output

For expensive middleware operations, consider caching:

```php
class CachedMiddleware implements MiddlewareInterface
{
    public function __construct(
        private CacheItemPoolInterface $cache,
        private MiddlewareInterface $middleware
    ) {}
    
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $cache_key = $this->generateCacheKey($request);
        
        $item = $this->cache->getItem($cache_key);
        if ($item->isHit()) {
            return $item->get();
        }
        
        $response = $this->middleware->process($request, $handler);
        
        $item->set($response);
        $item->expiresAfter(300); // 5 minutes
        $this->cache->save($item);
        
        return $response;
    }
}
```

## Testing Middleware

Test middleware in isolation:

```php
use PHPUnit\Framework\TestCase;
use Psr\Http\Server\RequestHandlerInterface;

class CustomMiddlewareTest extends TestCase
{
    public function testMiddlewareAddsHeader(): void
    {
        $middleware = new CustomMiddleware();
        
        $request = $this->createRequest('GET', '/');
        $handler = $this->createMock(RequestHandlerInterface::class);
        $handler->method('handle')
                ->willReturn($this->createResponse(200));
        
        $response = $middleware->process($request, $handler);
        
        $this->assertTrue($response->hasHeader('X-Custom-Header'));
        $this->assertEquals('Custom Value', $response->getHeaderLine('X-Custom-Header'));
    }
}
```

## Best Practices

1. **Keep middleware focused** - Each middleware should have a single responsibility
2. **Use events** - Leverage the event system for cross-cutting concerns
3. **Handle exceptions** - Wrap operations in try-catch blocks
4. **Log appropriately** - Log errors and important operations
5. **Test thoroughly** - Unit test middleware behavior
6. **Consider order** - Think about middleware execution order
7. **Use caching** - Cache expensive operations when possible
8. **Validate input** - Always validate and sanitize request data