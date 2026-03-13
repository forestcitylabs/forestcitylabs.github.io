---
sidebar_position: 3
---

# Application Kernel

The `src/KernelFactory.php` file is how you will modify the behaviour of your application. This file is responsible for creating the application container and the base kernel for the application that will process requests using middleware and return a response.

`KernelFactory` has three static methods, each building on the previous:

| Method | Returns | Purpose |
|--------|---------|---------|
| `getPackages()` | `array` | Returns the list of package config files to load into the container. |
| `createApplicationContainer()` | `ContainerInterface` | Builds and compiles the PHP-DI container from all packages and `services.php`. |
| `createHttpKernel()` | `Kernel` | Creates the container, then constructs the HTTP kernel and registers all middleware. |


## Adding a package

`getPackages()` returns the ordered list of `config/packages/*.php` files that are loaded into the container. To enable an optional package, add its path to the array:

```php
public static function getPackages(): array
{
    $packages = [
        // ... existing packages ...
        __DIR__ . '/../config/packages/security.php',
        __DIR__ . '/../config/packages/oauth.php',
    ];

    // ...

    return $packages;
}
```

### Environment-specific packages

Packages that should only be active in development are added conditionally. The standard install already does this for `development.php` (Whoops error pages) and `generator.php` (code generation commands):

```php
if ((bool) ('dev' === $_ENV['ENVIRONMENT'])) {
    $packages[] = __DIR__ . '/../config/packages/development.php';
    $packages[] = __DIR__ . '/../config/packages/generator.php';
}
```

Apply the same pattern for any package that should not be loaded in production.

### Package load order

Packages are loaded in the order they appear in the array. Because each file can add values to shared lists (using PHP-DI's `add()`), later packages can depend on services defined by earlier ones. `config/services.php` is always loaded last and takes precedence over everything — use it for application-level overrides.

---

## Adding middleware

`createHttpKernel()` builds the HTTP kernel and registers middleware by calling `$kernel->addMiddleware()`. Middleware is executed in the order it is added for incoming requests, and in reverse order for outgoing responses.

```php
public static function createHttpKernel(): Kernel
{
    $container = self::createApplicationContainer();
    $kernel = new Kernel(
        $container,
        $container->get(EventDispatcherInterface::class),
        $container->get(LoggerInterface::class)
    );

    $kernel->addMiddleware(MyMiddleware::class);

    return $kernel;
}
```

The class name passed to `addMiddleware()` is resolved from the container, so any middleware that declares constructor dependencies will have them autowired automatically.

### Default middleware stack

The standard install registers the following middleware in this order:

| Middleware | Condition | Description |
|------------|-----------|-------------|
| `WhoopsMiddleware` | `app.debug` only | Catches unhandled exceptions and renders a Whoops error page. |
| `DefaultCacheMiddleware` | Always | Adds default `Cache-Control` headers to responses. |
| `GeneratedByMiddleware` | Always | Appends an `X-Generated-By` header identifying the framework. |
| `AllowedHostMiddleware` | Always | Rejects requests with an untrusted `Host` header (400). |
| `CorsMiddleware` | Always | Handles CORS preflight requests and injects CORS response headers. |
| `GraphiQLMiddleware` | `app.debug` only | Serves the GraphiQL in-browser IDE at a well-known path. |
| `TrailingSlashMiddleware` | Always | Redirects requests with a trailing slash to the canonical path. |
| `GraphQLMiddleware` | Always | Handles requests to the GraphQL endpoint. |
| `RoutingMiddleware` | Always | Dispatches all other requests to the matched HTTP controller. |

### Adding custom middleware

Insert your middleware at the appropriate position in the stack. For example, to add a session middleware that must run after security but before routing:

```php
// Security middlewares.
$kernel->addMiddleware(AllowedHostMiddleware::class);
$kernel->addMiddleware(CorsMiddleware::class);

// Session handling.
$kernel->addMiddleware(\Application\Middleware\SessionMiddleware::class);

// The main routing middlewares.
$kernel->addMiddleware(TrailingSlashMiddleware::class);
$kernel->addMiddleware(GraphQLMiddleware::class);
$kernel->addMiddleware(RoutingMiddleware::class);
```

---

## Container compilation

In non-development environments `createApplicationContainer()` enables two PHP-DI optimisations:

```php
if (!(bool) ('dev' === $_ENV['ENVIRONMENT'])) {
    $builder->writeProxiesToFile(true, __DIR__ . '/../var/cache/proxies');
    $builder->enableCompilation(__DIR__ . '/../var/cache');
}
```

| Optimisation | Path | Description |
|--------------|------|-------------|
| Proxy generation | `var/cache/proxies/` | Writes lazy-loading proxy classes to disk instead of generating them at runtime. |
| Container compilation | `var/cache/` | Compiles the full container definition into a single `CompiledContainer.php` file for maximum performance. |

These are disabled in `dev` so that container changes (new services, changed definitions) are picked up immediately without a cache-clear step. Run `cache:clear` after any deployment to production to regenerate the compiled container.
