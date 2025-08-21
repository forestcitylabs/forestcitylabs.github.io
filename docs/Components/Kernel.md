# Kernel

The Kernel is the heart of the FCL Framework, responsible for handling HTTP requests through a configurable middleware pipeline. It implements the [PSR-15](https://www.php-fig.org/psr/psr-15/) HTTP Server Request Handler pattern and coordinates the execution of middleware.

## Overview

The Kernel:

- Receives HTTP requests (PSR-7 ServerRequestInterface)
- Executes middleware in a sequential pipeline
- Dispatches events before and after each middleware
- Returns HTTP responses (PSR-7 ResponseInterface)
- Handles exceptions and errors gracefully

## Basic Usage

### Creating a Kernel

```php
use ForestCityLabs\Framework\Kernel;
use Psr\EventDispatcher\EventDispatcherInterface;

$kernel = new Kernel($event_dispatcher);
```

### Adding Middleware

Add middleware to the kernel in order of execution:

```php
// Add middleware in execution order
$kernel->addMiddleware($cors_middleware);
$kernel->addMiddleware($session_middleware);
$kernel->addMiddleware($auth_middleware);  
$kernel->addMiddleware($routing_middleware);
$kernel->addMiddleware($error_middleware);
```

### Handling Requests

Process requests through the middleware pipeline:

```php
use GuzzleHttp\Psr7\ServerRequest;

// Create request from globals (or use your preferred PSR-7 implementation)
$request = ServerRequest::fromGlobals();

// Process request through kernel
$response = $kernel->handle($request);

// Send response to browser
$emitter = new ResponseEmitter();
$emitter->emit($response);
```

## Kernel Events

The Kernel dispatches events during request processing:

### PreMiddlewareHandleEvent

Dispatched before each middleware is executed:

```php
use ForestCityLabs\Framework\Event\Attribute\EventListener;
use ForestCityLabs\Framework\Events\PreMiddlewareHandleEvent;

#[EventListener(PreMiddlewareHandleEvent::class)]
class PreMiddlewareListener
{
    public function __invoke(PreMiddlewareHandleEvent $event): void
    {
        $middleware = $event->getMiddleware();
        $request = $event->getRequest();
        
        // Log middleware execution
        $this->logger->debug('Executing middleware', [
            'middleware' => get_class($middleware),
            'path' => $request->getUri()->getPath()
        ]);
        
        // Modify request before middleware execution
        $modified_request = $request->withAttribute('start_time', microtime(true));
        $event->setRequest($modified_request);
    }
}
```

### PostMiddlewareHandleEvent

Dispatched after each middleware completes:

```php
use ForestCityLabs\Framework\Events\PostMiddlewareHandleEvent;

#[EventListener(PostMiddlewareHandleEvent::class)]
class PostMiddlewareListener
{
    public function __invoke(PostMiddlewareHandleEvent $event): void
    {
        $middleware = $event->getMiddleware();
        $request = $event->getRequest();
        $response = $event->getResponse();
        
        // Calculate middleware execution time
        $start_time = $request->getAttribute('start_time');
        if ($start_time) {
            $execution_time = microtime(true) - $start_time;
            $this->logger->debug('Middleware completed', [
                'middleware' => get_class($middleware),
                'execution_time' => $execution_time,
                'status_code' => $response->getStatusCode()
            ]);
        }
        
        // Modify response after middleware execution
        $modified_response = $response->withHeader('X-Middleware', get_class($middleware));
        $event->setResponse($modified_response);
    }
}
```

## Complete Application Setup

Here's how to set up a complete application with the Kernel:

### Basic Application Structure

```php
<?php
// public/index.php

use ForestCityLabs\Framework\Kernel;
use GuzzleHttp\Psr7\ServerRequest;
use Http\Response\Sender\ResponseSender;

// Load dependencies
require_once __DIR__ . '/../vendor/autoload.php';

// Create container (PSR-11)
$container = new DI\Container();

// Create event dispatcher (PSR-14)
$event_dispatcher = new League\Event\EventDispatcher(
    new ForestCityLabs\Framework\Event\ListenerProvider(
        // Event listener discovery
        new ForestCityLabs\Framework\Utility\ClassDiscovery\ScanDirectoryDiscovery(
            __DIR__ . '/../src/EventListener'
        ),
        $cache_pool,
        $container
    )
);

// Create kernel
$kernel = new Kernel($event_dispatcher);

// Add middleware in order
$kernel->addMiddleware($container->get(CorsMiddleware::class));
$kernel->addMiddleware($container->get(SessionMiddleware::class));
$kernel->addMiddleware($container->get(AuthenticationMiddleware::class));
$kernel->addMiddleware($container->get(RoutingMiddleware::class));
$kernel->addMiddleware($container->get(GraphQLMiddleware::class));
$kernel->addMiddleware($container->get(NotFoundMiddleware::class));

// Handle request
$request = ServerRequest::fromGlobals();
$response = $kernel->handle($request);

// Send response
$sender = new ResponseSender();
$sender->send($response);
```

### With Error Handling

```php
<?php
// public/index.php with error handling

try {
    // Application setup...
    $response = $kernel->handle($request);
} catch (\Throwable $e) {
    // Log the error
    $logger->error('Unhandled exception', [
        'exception' => $e->getMessage(),
        'file' => $e->getFile(),
        'line' => $e->getLine(),
        'trace' => $e->getTraceAsString()
    ]);
    
    // Return error response
    if ($is_development) {
        // Development: Show detailed error
        $whoops = new Whoops\Run();
        $whoops->pushHandler(new Whoops\Handler\PrettyPageHandler());
        $response = new Response(
            500,
            ['Content-Type' => 'text/html'],
            $whoops->handleException($e)
        );
    } else {
        // Production: Generic error page
        $response = new Response(
            500,
            ['Content-Type' => 'text/html'],
            '<h1>Internal Server Error</h1>'
        );
    }
}

$sender->send($response);
```

## Kernel Configuration

### Environment-Specific Setup

```php
class KernelFactory
{
    public static function create(string $environment): Kernel
    {
        $container = self::createContainer($environment);
        $event_dispatcher = $container->get(EventDispatcherInterface::class);
        
        $kernel = new Kernel($event_dispatcher);
        
        // Add environment-specific middleware
        if ($environment === 'development') {
            $kernel->addMiddleware($container->get(WhoopsMiddleware::class));
        }
        
        // Common middleware
        $kernel->addMiddleware($container->get(CorsMiddleware::class));
        $kernel->addMiddleware($container->get(SessionMiddleware::class));
        
        if ($environment === 'production') {
            $kernel->addMiddleware($container->get(CacheMiddleware::class));
        }
        
        $kernel->addMiddleware($container->get(RoutingMiddleware::class));
        $kernel->addMiddleware($container->get(NotFoundMiddleware::class));
        
        return $kernel;
    }
}

// Usage
$kernel = KernelFactory::create($_ENV['APP_ENV'] ?? 'production');
```

### Middleware Prioritization

Order middleware by priority to ensure proper execution:

```php
class MiddlewareConfig
{
    public static function configure(Kernel $kernel, ContainerInterface $container): void
    {
        // Priority 1: Security and validation
        $kernel->addMiddleware($container->get(AllowedHostMiddleware::class));
        $kernel->addMiddleware($container->get(CorsMiddleware::class));
        
        // Priority 2: Session and authentication  
        $kernel->addMiddleware($container->get(SessionMiddleware::class));
        $kernel->addMiddleware($container->get(AuthenticationMiddleware::class));
        
        // Priority 3: Request processing
        $kernel->addMiddleware($container->get(RoutingMiddleware::class));
        $kernel->addMiddleware($container->get(GraphQLMiddleware::class));
        
        // Priority 4: Error handling (last resort)
        $kernel->addMiddleware($container->get(NotFoundMiddleware::class));
        $kernel->addMiddleware($container->get(ErrorHandlerMiddleware::class));
    }
}
```

## Advanced Usage

### Conditional Middleware

Add middleware conditionally based on request or environment:

```php
class ConditionalKernel extends Kernel
{
    public function addConditionalMiddleware(
        MiddlewareInterface $middleware,
        callable $condition
    ): void {
        $this->addMiddleware(new class($middleware, $condition) implements MiddlewareInterface {
            public function __construct(
                private MiddlewareInterface $middleware,
                private callable $condition
            ) {}
            
            public function process(
                ServerRequestInterface $request,
                RequestHandlerInterface $handler
            ): ResponseInterface {
                if (($this->condition)($request)) {
                    return $this->middleware->process($request, $handler);
                }
                
                return $handler->handle($request);
            }
        });
    }
}

// Usage
$kernel = new ConditionalKernel($event_dispatcher);

$kernel->addConditionalMiddleware(
    $api_middleware,
    fn($request) => str_starts_with($request->getUri()->getPath(), '/api/')
);

$kernel->addConditionalMiddleware(
    $admin_middleware,
    fn($request) => str_starts_with($request->getUri()->getPath(), '/admin/')
);
```

### Middleware Groups

Organize related middleware into groups:

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
$api_group->add($cors_middleware)
          ->add($bearer_auth_middleware)
          ->add($rate_limit_middleware);

$web_group = new MiddlewareGroup();  
$web_group->add($session_middleware)
          ->add($csrf_middleware)
          ->add($flash_middleware);

// Apply groups conditionally
if (str_starts_with($request->getUri()->getPath(), '/api/')) {
    $api_group->applyTo($kernel);
} else {
    $web_group->applyTo($kernel);
}
```

## Performance Considerations

### Middleware Caching

Cache expensive middleware operations:

```php
class CachedKernel extends Kernel
{
    public function __construct(
        EventDispatcherInterface $event_dispatcher,
        private CacheItemPoolInterface $cache
    ) {
        parent::__construct($event_dispatcher);
    }
    
    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        // Generate cache key from request
        $cache_key = $this->generateCacheKey($request);
        
        // Check cache for GET requests only
        if ($request->getMethod() === 'GET') {
            $item = $this->cache->getItem($cache_key);
            if ($item->isHit()) {
                return $item->get();
            }
        }
        
        $response = parent::handle($request);
        
        // Cache successful GET responses
        if ($request->getMethod() === 'GET' && $response->getStatusCode() === 200) {
            $item->set($response);
            $item->expiresAfter(300); // 5 minutes
            $this->cache->save($item);
        }
        
        return $response;
    }
}
```

### Profiling and Monitoring

Add profiling to identify bottlenecks:

```php
#[EventListener(PreMiddlewareHandleEvent::class)]
class ProfilingListener
{
    private array $timings = [];
    
    public function onPreMiddleware(PreMiddlewareHandleEvent $event): void
    {
        $middleware_class = get_class($event->getMiddleware());
        $this->timings[$middleware_class] = microtime(true);
    }
    
    #[EventListener(PostMiddlewareHandleEvent::class)]
    public function onPostMiddleware(PostMiddlewareHandleEvent $event): void
    {
        $middleware_class = get_class($event->getMiddleware());
        $start_time = $this->timings[$middleware_class] ?? microtime(true);
        $duration = microtime(true) - $start_time;
        
        // Log slow middleware
        if ($duration > 0.1) { // 100ms threshold
            $this->logger->warning('Slow middleware detected', [
                'middleware' => $middleware_class,
                'duration' => $duration
            ]);
        }
    }
}
```

## Testing the Kernel

### Unit Testing

Test kernel behavior in isolation:

```php
use PHPUnit\Framework\TestCase;

class KernelTest extends TestCase
{
    public function testKernelExecutesMiddleware(): void
    {
        $event_dispatcher = $this->createMock(EventDispatcherInterface::class);
        $kernel = new Kernel($event_dispatcher);
        
        $middleware = $this->createMock(MiddlewareInterface::class);
        $middleware->expects($this->once())
                   ->method('process')
                   ->willReturn(new Response(200));
        
        $kernel->addMiddleware($middleware);
        
        $request = new ServerRequest('GET', '/');
        $response = $kernel->handle($request);
        
        $this->assertEquals(200, $response->getStatusCode());
    }
}
```

### Integration Testing

Test complete request flow:

```php
class KernelIntegrationTest extends TestCase
{
    private Kernel $kernel;
    
    protected function setUp(): void
    {
        $container = $this->createContainer();
        $this->kernel = $container->get(Kernel::class);
    }
    
    public function testCompleteRequestFlow(): void
    {
        $request = new ServerRequest('GET', '/api/users');
        $response = $this->kernel->handle($request);
        
        $this->assertEquals(200, $response->getStatusCode());
        $this->assertStringContainsString('application/json', 
                                        $response->getHeaderLine('Content-Type'));
    }
}
```

## Best Practices

1. **Order middleware carefully** - Security and validation first, error handling last
2. **Use events** - Leverage the event system for cross-cutting concerns  
3. **Handle errors gracefully** - Always have error handling middleware
4. **Profile performance** - Monitor middleware execution times
5. **Test thoroughly** - Unit and integration test your kernel configuration
6. **Environment configuration** - Use different middleware stacks for different environments
7. **Keep it simple** - Don't add unnecessary middleware layers
8. **Document middleware order** - Make the execution order clear to other developers