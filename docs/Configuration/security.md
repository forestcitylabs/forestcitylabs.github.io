# Security

`config/packages/security.php` wires the core security subsystem: roles, role hierarchy, and the requirement registry.

## What it does

- Provides `security.roles` and `security.hierarchy` as merge-lists for defining application roles and their inheritance relationships.
- Registers `RequiresRole` as a built-in security requirement (other packages such as `oauth.php` add further requirements like `RequiresScope`).
- Wires `RoleRegistry` with the role list and hierarchy.
- Wires `RequirementRegistry` with all registered requirement classes.
- Registers two event listeners:
  - `SecurityPreRouteDispatchListener` — enforces security requirements before an HTTP route is dispatched.
  - `SecurityPreGraphQLFieldResolveListener` — enforces security requirements before a GraphQL field resolver is invoked.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `security.roles` | `[]` | List of role names available in the application (e.g. `ROLE_USER`, `ROLE_ADMIN`). |
| `security.hierarchy` | `[]` | Role inheritance map. A role listed as a value inherits all permissions of the key role. |
| `security.requirements` | `[RequiresRole::class]` | Requirement attribute classes registered with `RequirementRegistry`. Other packages add to this list. |

## Defining roles

```php
// services.php
use function DI\add;

'security.roles' => add([
    'ROLE_USER',
    'ROLE_EDITOR',
    'ROLE_ADMIN',
]),
```

## Defining a role hierarchy

```php
// services.php
use function DI\add;

'security.hierarchy' => add([
    'ROLE_ADMIN'  => ['ROLE_EDITOR'],
    'ROLE_EDITOR' => ['ROLE_USER'],
]),
```

In this example `ROLE_ADMIN` inherits `ROLE_EDITOR`, which in turn inherits `ROLE_USER`.

## Protecting routes and GraphQL fields

Use the `#[RequiresRole]` attribute on controller methods or GraphQL resolver methods:

```php
use ForestCityLabs\Framework\Security\Attribute\RequiresRole;

#[RequiresRole('ROLE_ADMIN')]
public function adminAction(): ResponseInterface { ... }
```
