# Router

`config/packages/router.php` wires the FastRoute-based HTTP router.

## What it does

- Scans `src/Controller/` for classes annotated with `#[Route]` and `#[RoutePrefix]` attributes using `ScanDirectoryDiscovery`.
- Binds `RouteParserInterface` to FastRoute's `Std` parser.
- Binds `RouteDataGeneratorInterface` to FastRoute's `GroupCountBased` generator.
- Builds a `RouteDispatcherInterface` via a factory that calls `DataLoader::loadRoutes()` to compile all discovered routes.
- Wires the routing `MetadataProvider` with the class discovery instance.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `router.paths` | `[{app.project_root}/src/Controller]` | Directories scanned for controller classes with route attributes. |

## Adding a controller directory

```php
// services.php
use function DI\add;
use function DI\string;

'router.paths' => add([
    string('{app.project_root}/src/Module/Controller'),
]),
```

## Defining routes

Routes are declared directly on controller methods using attributes. See the [Routing](../Standard Install/Routing.md) documentation for full details.

```php
<?php

namespace Application\Controller;

use ForestCityLabs\Framework\Routing\Attribute\Route;
use ForestCityLabs\Framework\Routing\Attribute\RoutePrefix;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;

#[RoutePrefix('/api')]
class ApiController
{
    #[Route('/status', methods: ['GET'])]
    public function status(ResponseFactoryInterface $rf): ResponseInterface
    {
        return $rf->createResponse(200);
    }
}
```

## Dependencies

| Package | Description |
|---------|-------------|
| [`nikic/fast-route`](https://packagist.org/packages/nikic/fast-route) | The underlying route matching library. |
