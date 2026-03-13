# OpenID Connect (OIDC)

`config/packages/oidc.php` extends the OAuth package with OpenID Connect (OIDC) support, including RSA key management, JWT signing, and the `openid` scope.

## What it does

- Overrides `OAuthServer::class` to use `OidcServer`, which adds OIDC-specific grant handling.
- Wires a `Keystore` with the configured RSA key files.
- Wires an `OidcClaimRegistry` with the configured claims and groups.
- Builds an `lcobucci/jwt` `Configuration` using the active key pair (RSA-SHA256).
- Registers the `openid` scope (not enabled by default).
- Wires `OidcMiddleware` with the redirect URI and application base URI.

## Configuration parameters

| Key | Default | Description |
|-----|---------|-------------|
| `oidc.keys` | `['default' => '{app.project_root}/var/keys/oidc_private.pem']` | Map of key name to PEM private key file path. The framework expects both a private and public key at the given path. |
| `oidc.active_key` | `'default'` | The name of the key from `oidc.keys` to use for signing JWTs. |
| `oidc.claims` | `[]` | Custom OIDC claims to include in the ID token. |
| `oidc.groups` | `[]` | Claim groups to register with the claim registry. |
| `oauth.redirect_uri` | `''` (from `oauth.php`) | The path the OIDC middleware redirects to after a completed authorization. |

## Generating keys

```bash
# Generate a private key
openssl genrsa -out var/keys/oidc_private.pem 4096

# Extract the public key
openssl rsa -in var/keys/oidc_private.pem -pubout -out var/keys/oidc_public.pem
```

## Using multiple key pairs

```php
// services.php
use function DI\add;
use function DI\string;

'oidc.keys' => add([
    'default' => string('{app.project_root}/var/keys/oidc_private.pem'),
    'v2'      => string('{app.project_root}/var/keys/oidc_private_v2.pem'),
]),
'oidc.active_key' => 'v2',
```

## Adding custom claims

```php
// services.php
use function DI\add;
use function DI\get;

'oidc.claims' => add([
    get(\Application\Security\Oidc\Claim\RolesClaim::class),
]),
```

## Dependencies

| Package | Description |
|---------|-------------|
| [`lcobucci/jwt`](https://packagist.org/packages/lcobucci/jwt) | JWT creation and validation. |
| [`league/oauth2-server`](https://packagist.org/packages/league/oauth2-server) | Underlying OAuth 2.0 server (required via `oauth.php`). |
