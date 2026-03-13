# Twig

`config/packages/twig.php` wires the Twig templating engine and its built-in `ManifestExtension`.

## What it does

- Initialises `twig.extensions`, `twig.template_directories`, and `twig.cache_directory` as overridable parameters.
- Creates the `Twig\Environment` via a factory that:
  - Loads templates from all directories in `twig.template_directories`.
  - Enables the cache directory and debug mode based on `app.debug`.
  - Registers all extensions from `twig.extensions`.
- Wires `ManifestExtension` with the configured manifest path.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `twig.template_directories` | `[{app.project_root}/templates]` (from `services.php`) | Directories Twig searches for template files. |
| `twig.cache_directory` | `{app.project_root}/var/cache/twig` (from `services.php`) | Directory where compiled templates are cached. Falls back to `sys_get_temp_dir()` if not set. |
| `twig.extensions` | `[]` | Twig extensions to register. Other packages and `services.php` can add extensions via `add([...])`. |
| `twig.extensions.manifest.path` | `null` | Absolute path to an asset manifest JSON file for the `ManifestExtension`. Set to `null` to disable manifest support. |

## Registering a Twig extension

```php
// services.php
use function DI\add;
use function DI\get;

'twig.extensions' => add([
    get(\Application\Twig\AppExtension::class),
]),
```

## Enabling the manifest extension

The `ManifestExtension` provides a `asset()` function in templates that resolves hashed asset filenames from a Vite or Webpack manifest:

```php
// services.php
use function DI\string;

'twig.extensions.manifest.path' => string('{app.web_root}/build/manifest.json'),
```

Then in a Twig template:

```twig
<script src="{{ asset('src/main.js') }}"></script>
```

## Adding a template directory

```php
// services.php
use function DI\add;
use function DI\string;

'twig.template_directories' => add([
    string('{app.project_root}/templates'),
    string('{app.project_root}/src/Module/templates'),
]),
```

## Dependencies

| Package | Description |
|---------|-------------|
| [`twig/twig`](https://packagist.org/packages/twig/twig) | The Twig template engine. |
