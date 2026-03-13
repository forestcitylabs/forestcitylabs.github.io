# Allowed Hosts

`config/packages/security_allowed_hosts.php` wires the `AllowedHostMiddleware` that rejects requests with unrecognised `Host` headers.

## What it does

- Initialises `security.allowed_hosts` as an empty merge-list (values are added in `services.php` via the `TRUSTED_HOST` environment variable).
- Wires `AllowedHostMiddleware` with the list of allowed hostnames.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `security.allowed_hosts` | `[]` | Hostnames the middleware will accept. Requests with a `Host` header not in this list receive a `400 Bad Request` response. |

The standard install populates this list in `services.php`:

```php
// services.php
'security.allowed_hosts' => add([
    env('TRUSTED_HOST', 'localhost'),
]),
```

## Adding allowed hosts

```php
// services.php
use function DI\add;
use function DI\env;

'security.allowed_hosts' => add([
    env('TRUSTED_HOST', 'localhost'),
    'api.example.com',
]),
```

## `.env` example

```
TRUSTED_HOST=example.com
```
