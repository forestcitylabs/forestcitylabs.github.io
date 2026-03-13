# Console

`config/packages/console.php` bootstraps the Symfony Console application and its class-discovery pipeline.

## What it does

- Creates the `Application` (the top-level Symfony Console runner) named "Forest City Labs Framework Console".
- Wires a `CommandLoader` backed by a `ChainedDiscovery` that combines:
  - `ManualDiscovery` — commands registered via the `console.commands` list.
  - `ScanDirectoryDiscovery` — commands found by scanning directories in `console.paths`.
- Attaches a `HelperSet` with the four standard Symfony helpers.
- Sets `setCatchErrors` to the value of `app.debug` so exceptions surface cleanly in development.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `console.commands` | `[]` | Explicit list of command class names to register. Other packages append their commands here via `add([...])`. |
| `console.paths` | `[{app.project_root}/src/Command]` | Directories scanned for command classes. Add directories here to auto-discover commands without manually listing them. |
| `console.helpers` | Four standard helpers | `FormatterHelper`, `DebugFormatterHelper`, `ProcessHelper`, `QuestionHelper`. Extend with `add([...])` to register additional helpers. |

## Registering a command manually

Commands in `src/Command/` are discovered automatically. To register a command from elsewhere, add it to `console.commands` in `services.php`:

```php
// services.php
use function DI\add;
use function DI\get;

'console.commands' => add([
    get(\Application\Command\MySpecialCommand::class),
]),
```

## Adding a command scan directory

```php
// services.php
use function DI\add;
use function DI\string;

'console.paths' => add([
    string('{app.project_root}/src/Module/Command'),
]),
```
