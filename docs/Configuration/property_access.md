# Property Access

`config/packages/property_access.php` binds the Symfony PropertyAccess component.

## What it does

Binds `PropertyAccessorInterface` to an autowired instance of `Symfony\Component\PropertyAccess\PropertyAccessor`.

The `PropertyAccessor` is used internally by the framework (e.g. the parameter converter pipeline) to read and write object properties via property paths like `user.profile.name`.

## Dependencies

| Package | Description |
|---------|-------------|
| [`symfony/property-access`](https://packagist.org/packages/symfony/property-access) | The Symfony PropertyAccess component. |

## Customising the accessor

If you need to configure the `PropertyAccessor` (e.g. to enable magic methods or throw exceptions on invalid property paths), override the binding in `services.php`:

```php
// services.php
use Symfony\Component\PropertyAccess\PropertyAccessor;
use Symfony\Component\PropertyAccess\PropertyAccessorInterface;
use function DI\factory;

PropertyAccessorInterface::class => factory(function () {
    return new PropertyAccessor(
        magicMethods: PropertyAccessor::MAGIC_GET | PropertyAccessor::MAGIC_SET,
        throwExceptionOnInvalidIndex: true,
    );
}),
```
