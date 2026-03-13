# Code Generator

`config/packages/generator.php` registers the `GenerateEntityCommand`, a code-generation tool that scaffolds new Doctrine entity classes.

## What it does

- Binds `Nette\PhpGenerator\Printer` to `PsrPrinter` (PSR-2 compliant PHP code output).
- Wires `GenerateEntityCommand` with the entity output directory and namespace.
- Adds the command to the console.

## Configuration

The entity directory and namespace are hardcoded in this package file but can be overridden in `services.php`:

```php
// services.php
use ForestCityLabs\Framework\Command\GenerateEntityCommand;
use function DI\string;

GenerateEntityCommand::class => autowire()
    ->constructorParameter('directory', string('{app.project_root}/src/Domain/Entity'))
    ->constructorParameter('namespace', 'Application\\Domain\\Entity'),
```

## Default values

| Parameter | Default |
|-----------|---------|
| `directory` | `{app.project_root}/src/Entity` |
| `namespace` | `Application\Entity` |

## Console commands registered

| Command | Description |
|---------|-------------|
| `generate:entity` | Interactively scaffolds a new Doctrine entity class in the configured directory. |

## Dependencies

| Package | Description |
|---------|-------------|
| [`nette/php-generator`](https://packagist.org/packages/nette/php-generator) | Used to generate well-formatted PHP class files. |
