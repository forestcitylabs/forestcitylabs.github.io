# Development Tools

`config/packages/development.php` configures the [Whoops](https://github.com/filp/whoops) error handler used in development environments.

## What it does

Registers a `Whoops\Run` instance using a factory that selects the appropriate handler based on the current SAPI:

- **CLI** — attaches a `PlainTextHandler` that writes stack traces to the application logger (`LoggerInterface`).
- **Web** — attaches a `PrettyPageHandler` that renders a rich, interactive HTML error page. Whoops is configured *not* to send HTTP codes, exit, or write directly to output so that the error response can be handled by an HTTP middleware instead.

## When to load this file

This package should only be loaded in non-production environments. The standard install loads it conditionally based on the `ENVIRONMENT` variable. Do not include this package in production as the `PrettyPageHandler` can expose sensitive server information.

## Dependencies

| Package | Description |
|---------|-------------|
| [`filp/whoops`](https://packagist.org/packages/filp/whoops) | The error handler library. |

## Customising the error page

You can push additional Whoops handlers before or after the defaults by overriding the `Run` service in `services.php`:

```php
// services.php
use Whoops\Handler\JsonResponseHandler;
use Whoops\Run;
use function DI\decorate;

Run::class => decorate(function (Run $whoops) {
    $whoops->pushHandler(new JsonResponseHandler());
    return $whoops;
}),
```
