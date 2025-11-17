Routing
=======

Routes can be created directly in controllers using the `Route` and `RoutePrefix` attributes. Controllers live in the `src/Controller/` directory and are automatically scanned for the routing attribute information.

```php
<?php

namespace Application\Controller;

use ForestCityLabs\Framework\Routing\Attribute\Route;
use ForestCityLabs\Framework\Routing\Attribute\RoutePrefix;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

#[RoutePrefix('/example')]
class ExampleController
{
  #[Route('/home')] # This would correspond to the uri "/example/home" in your application.
  public function home(
    ResponseFactoryInterface $rf, # You can autowire services in your controllers.
    ServerRequestInterface $r # If you need access to the request simply type-hint it here.
  ): ResponseInterface {
    return $rf->createResponse();
  }
}
```
