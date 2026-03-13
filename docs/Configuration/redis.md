# Redis

`config/packages/redis.php` wires the Predis client and creates named client instances used by other packages.

## What it does

- Binds `ClientInterface` to a default `Predis\Client` connected to `redis.uri`.
- Creates a dedicated `redis.cache_client` instance with a key prefix of `{redis.prefix}:cache:`, used by `cache.pool.default` in `services.php`.

## Configuration parameters

| Key | Environment variable | Default | Description |
|-----|----------------------|---------|-------------|
| `redis.uri` | `REDIS_URI` | — | Predis connection URI. Must be set in your `.env` file. |
| `redis.prefix` | `REDIS_PREFIX` | `'app'` | Prefix applied to all cache keys to avoid collisions when multiple applications share a Redis instance. |

### `.env` example

```
REDIS_URI=tcp://cache:6379
REDIS_PREFIX=myapp
```

### URI format

Predis accepts TCP and Unix socket URIs:

```
tcp://host:6379
tcp://host:6379?database=1
unix:///var/run/redis/redis.sock
```

## Adding a custom client

To add a dedicated client (e.g. for sessions), define it in `services.php`:

```php
// services.php
use Predis\Client;
use function DI\factory;
use function DI\get;

'redis.session_client' => factory(function (string $uri, string $prefix) {
    return new Client($uri, ['prefix' => $prefix . ':session:']);
})
    ->parameter('uri', get('redis.uri'))
    ->parameter('prefix', get('redis.prefix')),
```

## Dependencies

| Package | Description |
|---------|-------------|
| [`predis/predis`](https://packagist.org/packages/predis/predis) | Pure PHP Redis client. |
