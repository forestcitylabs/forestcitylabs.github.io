# Cache

`config/packages/cache.php` wires the PSR-6 cache subsystem and registers the `cache:clear` console command.

## What it does

- Initialises `cache.paths` and `cache.pools` as empty merge-lists so other packages can append to them.
- Binds `CacheItemPoolInterface` to `cache.pool.default`, making the default pool available for autowiring throughout the container.
- Constructs `CacheClearCommand` with all registered pools and paths, then adds it to the console.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `cache.paths` | `[]` | File paths that should be deleted when the cache is cleared. Populated by other packages (e.g. the compiled container path in `services.php`). |
| `cache.pools` | `[cache.pool.default]` | PSR-6 pool instances that `CacheClearCommand` will flush. Add additional pools here if your application uses more than one backend. |
| `cache.pool.default` | Defined in `services.php` | The default `CacheItemPoolInterface` implementation. The standard install uses `PredisCachePool`. |

## Console commands registered

| Command | Description |
|---------|-------------|
| `cache:clear` | Flushes all registered cache pools and deletes all registered cache file paths. |

## Customising the cache backend

Swap the default pool in `services.php`:

```php
// services.php
use ForestCityLabs\Framework\Cache\Pool\FilesystemCachePool;
use League\Flysystem\Filesystem;
use League\Flysystem\Local\LocalFilesystemAdapter;
use function DI\create;

'cache.pool.default' => create(FilesystemCachePool::class)
    ->constructor(
        new Filesystem(new LocalFilesystemAdapter(__DIR__ . '/../var/cache/pool'))
    ),
```

## Adding a second cache pool

```php
// services.php
use function DI\add;
use function DI\get;

'cache.pools' => add([
    get('cache.pool.sessions'),
]),
'cache.pool.sessions' => create(\ForestCityLabs\Framework\Cache\Pool\PredisCachePool::class)
    ->constructor(get('redis.session_client')),
```
