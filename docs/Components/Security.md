# Security

The FCL Framework provides comprehensive security features including OAuth 2.0, OpenID Connect (OIDC), role-based access control, and session management.

## Overview

The security system is built around several key components:

- **OAuth 2.0 Server** - Full OAuth 2.0 authorization server implementation
- **OpenID Connect (OIDC)** - Identity layer on top of OAuth 2.0
- **Role & Scope-Based Access Control** - Fine-grained permission system
- **Session Management** - Secure session handling with multiple storage options
- **Authentication Middleware** - Bearer token and session-based authentication

## OAuth 2.0 Server

### Supported Grant Types

The framework supports the following OAuth 2.0 grant types:

- **Authorization Code Grant** - For web applications
- **Refresh Token Grant** - For token renewal

### Basic Setup

```php
use ForestCityLabs\Framework\Security\OAuth\OAuthServer;
use ForestCityLabs\Framework\Security\OAuth\OAuthScopeRegistry;

// Create scope registry
$scope_registry = new OAuthScopeRegistry();
$scope_registry->addScope('read', 'Read access to resources');
$scope_registry->addScope('write', 'Write access to resources');

// Create OAuth server
$oauth_server = new OAuthServer(
    $client_manager,        // ClientManagerInterface
    $access_token_manager,  // AccessTokenManagerInterface  
    $auth_code_manager,     // AuthCodeManagerInterface
    $refresh_token_manager, // RefreshTokenManagerInterface
    $scope_registry,
    $encryption_service,    // EncryptionService
    $logger                 // LoggerInterface
);
```

### Required Interfaces

You must implement the following interfaces for your data layer:

#### Client Management

```php
use ForestCityLabs\Framework\Security\Manager\ClientManagerInterface;
use ForestCityLabs\Framework\Security\Model\ClientInterface;

class MyClientManager implements ClientManagerInterface
{
    public function find(string $identifier): ?ClientInterface 
    {
        // Find client by ID
    }
    
    public function isCredentialsValid(string $identifier, string $secret): bool
    {
        // Validate client credentials
    }
}
```

#### Access Token Management

```php
use ForestCityLabs\Framework\Security\Manager\AccessTokenManagerInterface;
use ForestCityLabs\Framework\Security\Model\AccessTokenInterface;

class MyAccessTokenManager implements AccessTokenManagerInterface 
{
    public function create(AccessTokenInterface $token): void
    {
        // Persist access token
    }
    
    public function find(string $identifier): ?AccessTokenInterface
    {
        // Find access token by ID
    }
    
    public function revoke(string $identifier): void
    {
        // Revoke/delete access token
    }
}
```

### Using OAuth Middleware

```php
use ForestCityLabs\Framework\Middleware\OAuthMiddleware;

$oauth_middleware = new OAuthMiddleware(
    $oauth_server,
    $response_factory,
    $logger
);

$kernel->addMiddleware($oauth_middleware);
```

## OpenID Connect (OIDC)

OIDC extends OAuth 2.0 to provide identity information through ID tokens.

### OIDC Server Setup

```php
use ForestCityLabs\Framework\Security\Oidc\OidcServer;
use ForestCityLabs\Framework\Security\Oidc\Keystore;
use ForestCityLabs\Framework\Security\Oidc\OidcClaimRegistry;

// Create keystore for JWT signing
$keystore = new Keystore();
$keystore->loadPrivateKey('/path/to/private.pem');
$keystore->loadPublicKey('/path/to/public.pem');

// Create claim registry
$claim_registry = new OidcClaimRegistry();
$claim_registry->addClaim('email', fn($user) => $user->getEmail());
$claim_registry->addClaim('name', fn($user) => $user->getName());

// Create OIDC server
$oidc_server = new OidcServer(
    $client_manager,
    $access_token_manager,
    $auth_code_manager, 
    $refresh_token_manager,
    $user_repository,      // UserRepositoryInterface
    $scope_registry,
    $claim_registry,
    $keystore,
    $encryption_service,
    $logger
);
```

### Using OIDC Middleware

```php
use ForestCityLabs\Framework\Middleware\OidcMiddleware;

$oidc_middleware = new OidcMiddleware(
    $oidc_server,
    $response_factory,
    $logger
);

$kernel->addMiddleware($oidc_middleware);
```

## Access Control

### Role-Based Access Control

Use the `#[RequiresRole]` attribute to protect routes:

```php
use ForestCityLabs\Framework\Security\Attribute\RequiresRole;
use ForestCityLabs\Framework\Routing\Attribute\Route;

class AdminController 
{
    #[Route('/admin/users')]
    #[RequiresRole('admin')]
    public function listUsers(): ResponseInterface
    {
        // Only accessible to users with 'admin' role
    }
    
    #[Route('/admin/settings')]  
    #[RequiresRole(['admin', 'superuser'])] // Multiple roles
    public function settings(): ResponseInterface
    {
        // Accessible to users with 'admin' OR 'superuser' role
    }
}
```

### Scope-Based Access Control

Use the `#[RequiresScope]` attribute for OAuth scopes:

```php
use ForestCityLabs\Framework\Security\Attribute\RequiresScope;

class APIController
{
    #[Route('/api/users')]
    #[RequiresScope('read')]
    public function getUsers(): ResponseInterface
    {
        // Requires 'read' scope
    }
    
    #[Route('/api/users', methods: ['POST'])]
    #[RequiresScope(['write', 'admin'])]
    public function createUser(): ResponseInterface  
    {
        // Requires 'write' AND 'admin' scopes
    }
}
```

### Role Registry

Configure available roles in your application:

```php
use ForestCityLabs\Framework\Security\RoleRegistry;

$role_registry = new RoleRegistry();
$role_registry->addRole('user', 'Basic User');
$role_registry->addRole('admin', 'Administrator'); 
$role_registry->addRole('superuser', 'Super User');
```

## Authentication Middleware

### Bearer Token Authentication

For API endpoints using OAuth access tokens:

```php
use ForestCityLabs\Framework\Middleware\BearerTokenAuthenticationMiddleware;

$bearer_middleware = new BearerTokenAuthenticationMiddleware(
    $access_token_manager,
    $user_repository,
    $logger
);

$kernel->addMiddleware($bearer_middleware);
```

### Session Authentication

For traditional web applications:

```php
use ForestCityLabs\Framework\Middleware\SessionAuthenticationMiddleware;

$session_middleware = new SessionAuthenticationMiddleware(
    $user_repository,
    $logger
);

$kernel->addMiddleware($session_middleware);
```

## Session Management

The framework provides secure session management using PHP's built-in session functionality.

### Session Configuration

```php
// Configure secure session settings
ini_set('session.cookie_lifetime', 3600);
ini_set('session.cookie_secure', true);     // HTTPS only
ini_set('session.cookie_httponly', true);  // No JavaScript access
ini_set('session.cookie_samesite', 'Strict'); // CSRF protection
ini_set('session.use_strict_mode', true);  // Prevent session fixation
```

### Using Sessions

```php
// In a controller
public function login(Session $session): ResponseInterface
{
    $session->set('user_id', 123);
    $session->set('authenticated', true);
    
    return $this->redirect('/dashboard');
}

public function dashboard(Session $session): ResponseInterface  
{
    $user_id = $session->get('user_id');
    $is_authenticated = $session->get('authenticated', false);
    
    // ...
}
```

## Flash Messages

Temporary messages that persist across redirects:

```php
use ForestCityLabs\Framework\Session\Flash;

public function saveUser(Flash $flash): ResponseInterface
{
    // ... save user logic
    
    $flash->set('success', 'User saved successfully!');
    return $this->redirect('/users');
}

public function listUsers(Flash $flash): ResponseInterface
{
    $success_message = $flash->get('success'); // Retrieved once, then deleted
    
    // ...
}
```

## Security Events

The framework dispatches security-related events:

- **PreRouteDispatchEvent** - Before route execution
- **PreGraphQLFieldResolveEvent** - Before GraphQL field resolution

### Security Event Listeners

```php
use ForestCityLabs\Framework\Event\Attribute\EventListener;
use ForestCityLabs\Framework\Events\PreRouteDispatchEvent;

#[EventListener(PreRouteDispatchEvent::class)]
class SecurityEventListener
{
    public function __invoke(PreRouteDispatchEvent $event): void
    {
        $request = $event->getRequest();
        
        // Perform security checks
        $this->checkRateLimit($request);
        $this->logSecurityEvent($request);
    }
}
```

## Encryption Services

The framework provides encryption utilities:

```php
use ForestCityLabs\Framework\Utility\EncryptionService;

$encryption = new EncryptionService('your-secret-key');

// Encrypt data
$encrypted = $encryption->encrypt('sensitive data');

// Decrypt data  
$decrypted = $encryption->decrypt($encrypted);
```

## Security Best Practices

1. **Always use HTTPS** in production
2. **Rotate encryption keys** regularly
3. **Implement rate limiting** for authentication endpoints
4. **Log security events** for monitoring
5. **Use strong, unique passwords** for OAuth clients
6. **Implement proper CORS policies**
7. **Validate all input** data
8. **Use secure session settings** (httpOnly, secure, sameSite)