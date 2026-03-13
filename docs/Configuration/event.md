# Events

`config/packages/event.php` wires the PSR-14 event dispatcher and its listener discovery pipeline.

## What it does

- Creates a `ChainedDiscovery` composed of:
  - `ScanDirectoryDiscovery` — automatically discovers listener classes in `src/EventListener/`.
  - `ManualDiscovery` — resolves listeners explicitly listed in `event.listeners`.
- Binds `ListenerProviderInterface` to the framework's `ListenerProvider`, backed by the chained discovery.
- Binds `EventDispatcherInterface` to `League\Event\EventDispatcher`.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `event.listeners` | `[]` | Explicit list of listener class names. Use this for listeners that live outside the auto-scanned directories. |
| `event.listener_paths` | `[{app.project_root}/src/EventListener]` | Directories scanned for event listener classes. |

## Creating a listener

Place your listener class in `src/EventListener/`. The framework will discover and register it automatically. Listeners should implement the appropriate interface or use PHP attributes to declare which events they handle.

## Adding a scan directory

```php
// services.php
use function DI\add;
use function DI\string;

'event.listener_paths' => add([
    string('{app.project_root}/src/Module/EventListener'),
]),
```

## Registering a listener manually

```php
// services.php
use function DI\add;

'event.listeners' => add([
    \Application\EventListener\MySpecialListener::class,
]),
```
