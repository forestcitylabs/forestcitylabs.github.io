# Logger

`config/packages/logger.php` wires the Monolog logger and binds it to `LoggerInterface`.

## What it does

- Creates a `StreamHandler` writing to `logger.path`.
- Wraps the stream handler in a `FingersCrossedHandler` that buffers log records and only flushes them when a record at or above the configured level is emitted.
- Derives the minimum log `Level` from `app.debug`: `Level::Debug` in debug mode, `Level::Warning` in production.
- Creates a `Logger` named after `app.name` and binds it to `LoggerInterface`.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `logger.path` | `php://stderr` (from `services.php`) | Log output destination. Accepts any PHP stream wrapper or a file path. |
| `logger.handlers` | `[FingersCrossedHandler]` (from `services.php`) | Monolog handlers attached to the logger. Other packages or `services.php` can append additional handlers via `add([...])`. |

## Log levels by environment

| `app.debug` | Minimum level |
|-------------|---------------|
| `true` (dev) | `Debug` — all messages are written. |
| `false` (prod) | `Warning` — only warnings and above trigger a flush. |

## Writing logs to a file

```php
// services.php
'logger.path' => string('{app.project_root}/var/log/app.log'),
```

## Adding a handler

```php
// services.php
use Monolog\Handler\SlackWebhookHandler;
use function DI\add;
use function DI\create;

'logger.handlers' => add([
    create(SlackWebhookHandler::class)->constructor('https://hooks.slack.com/...'),
]),
```

## Dependencies

| Package | Description |
|---------|-------------|
| [`monolog/monolog`](https://packagist.org/packages/monolog/monolog) | The logging library. |
