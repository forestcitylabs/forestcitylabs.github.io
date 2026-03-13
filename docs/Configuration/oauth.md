# OAuth 2.0

`config/packages/oauth.php` wires the OAuth 2.0 server, scope registry, and authentication middleware.

## What it does

- Provides `oauth.redirect_uri`, `oauth.scopes`, and `oauth.grants` as merge-lists.
- Wires `OAuthServer` with the registered grants.
- Wires `OAuthScopeRegistry` with the registered scopes.
- Adds `RequiresScope` to the security requirements registry so that routes and GraphQL fields can require specific OAuth scopes.
- Wires `OAuthMiddleware` (handles the OAuth flow and redirects) and `BearerTokenAuthenticationMiddleware` (validates `Authorization: Bearer` tokens on protected paths).

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `oauth.redirect_uri` | `''` | The path or URI the OAuth server redirects to after a completed authorization flow. |
| `oauth.scopes` | `[]` | Named scopes available on the server. Values are booleans indicating whether the scope is enabled by default. |
| `oauth.grants` | `[]` | OAuth 2.0 grant type instances to register with the server. |
| `security.token_auth_paths` | _(defined elsewhere)_ | Regex pattern matching paths that require bearer token authentication. |

## Registering a grant

```php
// services.php
use League\OAuth2\Server\Grant\ClientCredentialsGrant;
use function DI\add;
use function DI\get;

'oauth.grants' => add([
    get(ClientCredentialsGrant::class),
]),
```

## Registering a scope

```php
// services.php
use function DI\add;

'oauth.scopes' => add([
    'read:profile' => false,
    'write:profile' => false,
]),
```

## Dependencies

| Package | Description |
|---------|-------------|
| [`league/oauth2-server`](https://packagist.org/packages/league/oauth2-server) | OAuth 2.0 server implementation. |

> **Note:** The OIDC package (`oidc.php`) builds on top of this package. If you need OpenID Connect support, load `oidc.php` instead of (or in addition to) this file.
