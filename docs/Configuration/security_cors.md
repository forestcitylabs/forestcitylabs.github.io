# CORS

`config/packages/security_cors.php` wires the `CorsMiddleware` that handles Cross-Origin Resource Sharing (CORS) preflight and response headers.

## What it does

- Initialises `security.cors.allow_origins`, `security.cors.allow_methods`, and `security.cors.allow_headers` as empty merge-lists.
- Sets `security.cors.max_age` to `3600` seconds.
- Wires `CorsMiddleware` with all four parameters.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `security.cors.allow_origins` | `[]` | Permitted origins for cross-origin requests. Populated in `services.php` with `BASE_URI`. |
| `security.cors.allow_methods` | `[]` | HTTP methods allowed in cross-origin requests. Populated in `services.php`. |
| `security.cors.allow_headers` | `[]` | HTTP headers allowed in cross-origin requests. Populated in `services.php`. |
| `security.cors.max_age` | `3600` | How long (in seconds) browsers may cache preflight responses. |

The standard install sets initial values in `services.php`:

```php
// services.php
'security.cors.allow_origins' => add([env('BASE_URI', 'http://localhost:8080')]),
'security.cors.allow_methods' => add(['GET', 'POST', 'OPTIONS']),
'security.cors.allow_headers' => add(['Content-Type', 'Authorization']),
```

## Adding additional origins

```php
// services.php
use function DI\add;

'security.cors.allow_origins' => add([
    env('BASE_URI', 'http://localhost:8080'),
    'https://my-frontend.example.com',
]),
```

## Changing the max-age

```php
// services.php
'security.cors.max_age' => 86400, // 24 hours
```
