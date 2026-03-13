# ORM

`config/packages/orm.php` configures Doctrine ORM and registers the full suite of ORM console commands.

## What it does

- Configures a Doctrine `Configuration` with:
  - `AttributeDriver` reading entity paths from `orm.entity_paths`.
  - `UnderscoreNamingStrategy` (converts camelCase property names to `snake_case` columns).
  - Metadata, query, and result caches pointed at `cache.pool.doctrine`.
  - Proxy classes stored in `orm.proxy_directory`; auto-generation enabled only in debug mode.
  - Native lazy objects enabled.
- Creates the `EntityManagerInterface` from the DBAL connection and configuration.
- Wires `EntityManagerProvider` (used by console commands) to `SingleManagerProvider`.
- Registers ORM console commands.
- Aliases `cache.pool.doctrine` to `cache.pool.default` (override in `services.php` to use a separate pool).

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `orm.entity_paths` | `[{app.project_root}/src/Entity]` (from `services.php`) | Directories scanned by `AttributeDriver` for entity classes. |
| `orm.proxy_directory` | `{app.project_root}/var/cache/doctrine` | Where Doctrine writes generated proxy class files. |
| `cache.pool.doctrine` | Aliased to `cache.pool.default` | PSR-6 cache pool used for Doctrine metadata, query, and result caches. Override to use a dedicated pool. |

## Using a dedicated Doctrine cache pool

```php
// services.php
use ForestCityLabs\Framework\Cache\Pool\PredisCachePool;
use function DI\create;
use function DI\get;

'cache.pool.doctrine' => create(PredisCachePool::class)
    ->constructor(get('redis.doctrine_client')),
```

## Adding entity paths

```php
// services.php
use function DI\add;
use function DI\string;

'orm.entity_paths' => add([
    string('{app.project_root}/src/Entity'),
    string('{app.project_root}/src/Module/Entity'),
]),
```

## Console commands registered

| Command | Description |
|---------|-------------|
| `orm:schema-tool:create` | Creates the database schema from entity mappings. |
| `orm:schema-tool:update` | Updates the database schema to match current mappings. |
| `orm:schema-tool:drop` | Drops the database schema. |
| `orm:generate-proxies` | Pre-generates proxy classes for all entities. |
| `orm:run-dql` | Executes a DQL query and prints results. |
| `orm:validate-schema` | Validates the mapping against the database schema. |
| `orm:info` | Lists all mapped entities. |
| `orm:mapping:describe` | Describes the mapping metadata for a given entity. |

## Dependencies

| Package | Description |
|---------|-------------|
| [`doctrine/orm`](https://packagist.org/packages/doctrine/orm) | Doctrine Object Relational Mapper. |
