# GraphQL

`config/packages/graphql.php` wires the GraphQL engine, including type/controller discovery, schema construction, scalar transformers, and all GraphQL-related console commands.

## What it does

- Scans `src/Entity/` for PHP classes annotated as GraphQL types.
- Scans `src/Controller/GraphQL/` for PHP classes that define queries and mutations.
- Builds a `GraphQL\Type\Schema` from the discovered types via `TypeRegistry`.
- Wires `MetadataProvider` with both discovery instances.
- Wires `TransformerManager` with the `graphql.transformers` list.
- Configures code-generation commands with the correct project paths and namespaces.
- Registers four console commands.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `graphql.type_paths` | `[{app.project_root}/src/Entity]` | Directories scanned for GraphQL type classes. Add paths here if your types live in other modules. |
| `graphql.controller_paths` | `[{app.project_root}/src/Controller/GraphQL]` | Directories scanned for GraphQL controller classes that define resolvers. |
| `graphql.transformers` | Defined in `services.php` | List of scalar transformer instances registered with `TransformerManager`. |

## Console commands registered

| Command | Description |
|---------|-------------|
| `graphql:validate-schema` | Validates the GraphQL schema for errors. |
| `graphql:dump-schema` | Dumps the current schema to SDL (Schema Definition Language). |
| `graphql:generate-from-schema` | Generates entity and controller stubs from a `.graphql` schema file. |
| `graphql:schema-diff` | Diffs the current code-generated schema against a reference `.graphql` file. |

## Code generation paths

The `graphql:generate-from-schema` and `graphql:schema-diff` commands use `config/schema.graphql` as their reference file. This path is set in the package and can be overridden:

```php
// services.php
use ForestCityLabs\Framework\Command\GraphQLGenerateFromSchema;
use ForestCityLabs\Framework\Command\GraphQLSchemaDiffCommand;
use function DI\string;

GraphQLSchemaDiffCommand::class => autowire()
    ->constructorParameter('schema_file', string('{app.project_root}/config/my-schema.graphql')),
```

## Adding a type scan directory

```php
// services.php
use function DI\add;
use function DI\string;

'graphql.type_paths' => add([
    string('{app.project_root}/src/Module/Entity'),
]),
```

## Adding a scalar transformer

```php
// services.php
use function DI\add;
use function DI\get;

'graphql.transformers' => add([
    get(\Application\GraphQL\Transformer\MoneyTransformer::class),
]),
```
