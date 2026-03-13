# Database Migrations

`config/packages/migrations.php` wires Doctrine Migrations and registers the full suite of migration console commands.

## What it does

- Creates a `ConfigurationLoader` from the `migration.paths` map (namespace → directory).
- Wires `DependencyFactory` using the existing `EntityManagerInterface` (via `ExistingEntityManager`).
- Registers all standard Doctrine Migrations console commands.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `migration.paths` | Defined in `services.php` | Map of PHP namespace to filesystem directory for migration classes. |

The `migration.paths` key is an associative array where each entry maps the namespace used for generated migration class names to the directory where those files are stored.

### Default value (from `services.php`)

```php
'migration.paths' => add([
    'Application\\Migrations' => string('{app.project_root}/migrations'),
]),
```

### Adding a second migration directory

```php
// services.php
use function DI\add;
use function DI\string;

'migration.paths' => add([
    'Application\\Migrations' => string('{app.project_root}/migrations'),
    'Application\\Module\\Migrations' => string('{app.project_root}/src/Module/migrations'),
]),
```

## Console commands registered

| Command | Description |
|---------|-------------|
| `migrations:diff` | Generates a migration by comparing the current schema to the database. |
| `migrations:current` | Shows the current migration version. |
| `migrations:dump-schema` | Dumps the schema to a migration file. |
| `migrations:execute` | Executes a specific migration version up or down. |
| `migrations:generate` | Generates a blank migration class. |
| `migrations:latest` | Shows the latest available migration version. |
| `migrations:migrate` | Migrates the database to a specified version (default: latest). |
| `migrations:rollup` | Rolls up all migrations into a single baseline migration. |
| `migrations:status` | Shows the migration status of the database. |
| `migrations:version` | Manually marks a version as migrated or not migrated. |
| `migrations:up-to-date` | Checks whether the database is up to date. |
| `migrations:sync-metadata-storage` | Syncs the metadata storage with the database. |
| `migrations:list` | Lists all available migrations and their status. |

## Dependencies

| Package | Description |
|---------|-------------|
| [`doctrine/migrations`](https://packagist.org/packages/doctrine/migrations) | The Doctrine Migrations library. |
