# HTTP Factories

`config/packages/http.php` binds the PSR-17 HTTP factory interfaces to their Guzzle-based implementations.

## What it does

Maps the three PSR-17 factory interfaces to concrete classes from `php-http/guzzle7-factory`:

| Interface | Implementation |
|-----------|----------------|
| `ResponseFactoryInterface` | `Http\Factory\Guzzle\ResponseFactory` |
| `StreamFactoryInterface` | `Http\Factory\Guzzle\StreamFactory` |
| `UriFactoryInterface` | `Http\Factory\Guzzle\UriFactory` |

These bindings allow any service that type-hints a PSR-17 interface to receive the correct factory via autowiring.

## Dependencies

| Package | Description |
|---------|-------------|
| [`php-http/guzzle7-factory`](https://packagist.org/packages/php-http/guzzle7-factory) | PSR-17 factory implementations backed by Guzzle 7. |

## Swapping the HTTP factory

To use a different PSR-17 implementation (e.g. `nyholm/psr7`), replace the bindings in `services.php`:

```php
// services.php
use Nyholm\Psr7\Factory\Psr17Factory;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\StreamFactoryInterface;
use Psr\Http\Message\UriFactoryInterface;
use function DI\autowire;

ResponseFactoryInterface::class => autowire(Psr17Factory::class),
StreamFactoryInterface::class  => autowire(Psr17Factory::class),
UriFactoryInterface::class     => autowire(Psr17Factory::class),
```
