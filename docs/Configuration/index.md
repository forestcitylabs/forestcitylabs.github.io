---
sidebar_position: 4
---
# Container Configuration

Most configuration for your application is stored in the `config/` folder. The `services.php` file contains the most important settings you need to set up for most applications to function correctly.

## `config/services.php`

This file returns a PHP-DI container definitions array. Each key is a service or parameter name, and its value defines how that service or parameter is resolved. The file uses [PHP-DI helper functions](https://php-di.org/doc/php-definitions.html) (`env`, `get`, `create`, `factory`, `add`, `string`) to wire everything together.

```php
use function DI\add;
use function DI\create;
use function DI\env;
use function DI\factory;
use function DI\get;
use function DI\string;
```

## Application

| Key | Default | Description |
|-----|---------|-------------|
| `app.name` | `'Application'` | Human-readable name for your application. |
| `app.version` | `'dev-main'` | Current version string. |
| `app.environment` | `ENVIRONMENT` env var | Runtime environment (`dev`, `prod`, etc.). |
| `app.project_root` | Resolved from `__DIR__/..` | Absolute path to the project root directory. |
| `app.web_root` | `{app.project_root}/public` | Absolute path to the public web root. |
| `app.base_uri` | `BASE_URI` env var (`http://localhost:8080`) | Base URI of the application as a `UriInterface` instance. |
| `app.debug` | Derived from `app.environment` | `true` when `app.environment` is `dev`, `false` otherwise. |

`app.base_uri` is built by a factory that creates a `UriInterface` from `BASE_URI`. Set `BASE_URI` in your `.env` file to the public URL of your application.

`app.debug` is automatically derived — you do not need to set it manually. Change the `ENVIRONMENT` variable to toggle debug mode.

---

## Security

### CORS

| Key | Default | Description |
|-----|---------|-------------|
| `security.cors.allow_origins` | `[BASE_URI]` | List of allowed CORS origins. Defaults to your application's base URI. |
| `security.cors.allow_methods` | `['GET', 'POST', 'OPTIONS']` | HTTP methods allowed in cross-origin requests. |
| `security.cors.allow_headers` | `['Content-Type', 'Authorization']` | HTTP headers allowed in cross-origin requests. |

All three keys use `add([...])`, meaning additional values can be merged in from other config files. To allow additional origins, add them to the array:

```php
'security.cors.allow_origins' => add([
    env('BASE_URI', 'http://localhost:8080'),
    'https://my-other-domain.com',
]),
```

### Trusted Hosts

| Key | Default | Description |
|-----|---------|-------------|
| `security.allowed_hosts` | `[TRUSTED_HOST]` | Hostnames the application will respond to. Defaults to the `TRUSTED_HOST` env var (`localhost`). |

Set `TRUSTED_HOST` in your `.env` file to match your production domain to prevent host header injection attacks.

---

## GraphQL

| Key | Default | Description |
|-----|---------|-------------|
| `graphql.transformers` | `[DateTimeImmutableTransformer]` | List of scalar transformers registered with the GraphQL engine. |

Transformers convert PHP types to and from GraphQL scalar values. The `DateTimeImmutableTransformer` is included by default to handle `DateTimeImmutable` objects. To add your own transformer, append it to the array:

```php
'graphql.transformers' => add([
    get(\ForestCityLabs\Framework\GraphQL\Transformer\DateTimeImmutableTransformer::class),
    get(\Application\GraphQL\Transformer\MoneyTransformer::class),
]),
```

---

## Cache

| Key | Default | Description |
|-----|---------|-------------|
| `cache.paths` | `[{app.project_root}/var/cache/CompiledContainer.php]` | Paths to files that should be invalidated when the cache is cleared. |
| `cache.pool.default` | `PredisCachePool` using `redis.cache_client` | The default PSR-6 cache pool used by the application. |

The standard install uses Redis via Predis as the default cache backend. The Redis connection is configured separately under the `redis.cache_client` service. Set `REDIS_URI` in your `.env` file to point to your Redis instance (e.g. `tcp://cache:6379`).

To swap out the cache backend, replace `cache.pool.default` with a different PSR-6 pool implementation. See the [Caching](../Components/Caching.md) documentation for available drivers.

---

## Logger

| Key | Default | Description |
|-----|---------|-------------|
| `logger.path` | `php://stderr` | Log output destination. Can be a file path or a PHP stream wrapper. |
| `logger.handlers` | `[FingersCrossedHandler]` | Monolog handlers attached to the application logger. |

By default, logs are written to `stderr` through a `FingersCrossedHandler`, which buffers log records and only flushes them when an error-level message is emitted. To write logs to a file instead:

```php
'logger.path' => string('{app.project_root}/var/log/app.log'),
```

---

## ORM

| Key | Default | Description |
|-----|---------|-------------|
| `orm.entity_paths` | `[{app.project_root}/src/Entity]` | Directories scanned by Doctrine ORM for entity classes. |

Add additional directories if your entities are spread across multiple namespaces or packages:

```php
'orm.entity_paths' => add([
    string('{app.project_root}/src/Entity'),
    string('{app.project_root}/src/AnotherModule/Entity'),
]),
```

The database connection is configured via the `DATABASE_URI` environment variable (e.g. `pdo-mysql://root:root@db:3306/app`).

---

## Migrations

| Key | Default | Description |
|-----|---------|-------------|
| `migration.paths` | `['Application\\Migrations' => {app.project_root}/migrations]` | Map of namespace to directory for Doctrine Migrations. |

The key is the PHP namespace for generated migration classes and the value is the filesystem path where those classes are stored. To add a second migration set:

```php
'migration.paths' => add([
    'Application\\Migrations' => string('{app.project_root}/migrations'),
    'Application\\AnotherModule\\Migrations' => string('{app.project_root}/src/AnotherModule/migrations'),
]),
```

---

## Twig

| Key | Default | Description |
|-----|---------|-------------|
| `twig.cache_directory` | `{app.project_root}/var/cache/twig` | Directory where Twig stores compiled template files. |
| `twig.extensions` | `[]` | Additional Twig extensions to register. |
| `twig.template_directories` | `[{app.project_root}/templates]` | Directories scanned for Twig template files. |

To register a Twig extension:

```php
'twig.extensions' => add([
    get(\Application\Twig\MyExtension::class),
]),
```

---

## Console

| Key | Default | Description |
|-----|---------|-------------|
| `console.commands` | `[]` | Additional console commands to register with the application. |

Register your own Symfony Console commands here:

```php
'console.commands' => add([
    get(\Application\Command\MyCommand::class),
]),
```

---

## Environment Variables Reference

All environment variables are read from your `.env` file. Copy `.env.dist` to `.env` to get started:

```bash
$ cp .env.dist .env
```

| Variable | Description | Example |
|----------|-------------|---------|
| `BASE_URI` | Public base URL of the application | `http://localhost:8080` |
| `ENVIRONMENT` | Runtime environment | `dev` or `prod` |
| `DATABASE_URI` | Doctrine DBAL DSN for the database | `pdo-mysql://root:root@db:3306/app` |
| `TRUSTED_HOST` | Hostname the app accepts requests for | `localhost` |
| `REDIS_URI` | Predis connection URI for cache and sessions | `tcp://cache:6379` |
