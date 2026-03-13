# Utilities

`config/packages/utility.php` wires the parameter resolver and parameter converter pipelines used throughout the framework, along with the Doctrine `Inflector`.

## What it does

- Registers default `ParameterResolverInterface` implementations in a chained resolver.
- Registers default `ParameterConverterInterface` implementations in a chained converter.
- Binds `ParameterConverterInterface` to `ChainedParameterConverter`.
- Binds `ParameterResolverInterface` to `ChainedParameterResolver`.
- Creates a Doctrine `Inflector` using the default English factory.

## Parameter resolvers

Resolvers determine how method/function parameters are populated when the framework invokes a controller, listener, or command.

| Resolver | Description |
|----------|-------------|
| `IndexedParameterResolver` | Resolves parameters by their positional index from a provided argument list. |
| `ContainerParameterResolver` | Resolves parameters by type-hinting them against the DI container. |

Resolvers are tried in order. Add custom resolvers in `services.php`:

```php
// services.php
use function DI\add;
use function DI\get;

'utility.parameter_resolvers' => add([
    get(\Application\Utility\RequestAttributeResolver::class),
    get(\ForestCityLabs\Framework\Utility\ParameterResolver\IndexedParameterResolver::class),
    get(\ForestCityLabs\Framework\Utility\ParameterResolver\ContainerParameterResolver::class),
]),
```

## Parameter converters

Converters transform raw scalar values (e.g. route path segments, request body fields) into typed PHP objects before they are passed to a resolver.

| Converter | Description |
|-----------|-------------|
| `UuidParameterConverter` | Converts UUID strings into `Ramsey\Uuid\UuidInterface` instances. |
| `DateTimeParameterConverter` | Converts date/time strings into `DateTimeInterface` instances. |
| `InputTypeConverter` | Converts associative arrays into GraphQL input type objects. |

Add custom converters in `services.php`:

```php
// services.php
use function DI\add;
use function DI\get;

'utility.parameter_converters' => add([
    get(\Application\Utility\MoneyParameterConverter::class),
]),
```

## Inflector

The `Doctrine\Inflector\Inflector` service is available for autowiring wherever string inflection (pluralisation, singularisation, camelCase ↔ snake_case) is needed.

## Dependencies

| Package | Description |
|---------|-------------|
| [`doctrine/inflector`](https://packagist.org/packages/doctrine/inflector) | String inflection utility. |
