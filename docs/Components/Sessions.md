Sessions
========

The Forest City Labs Framework uses standard PHP sessions for session management, providing a simple and reliable session handling approach.

Requirements
------------

The session component uses PHP's built-in session functionality and requires the [`dflydev/fig-cookies`](https://packagist.org/packages/dflydev/fig-cookies) library for PSR-7 compatible cookie handling.

Configuration
-------------

Sessions are configured using PHP's standard session configuration. You can set session parameters through `php.ini` or programmatically:

```php
<?php

// Configure session settings
ini_set('session.cookie_lifetime', 3600);
ini_set('session.cookie_path', '/');
ini_set('session.cookie_domain', '');
ini_set('session.cookie_secure', true);
ini_set('session.cookie_httponly', true);
ini_set('session.cookie_samesite', 'Strict');
```

Usage
-----

To use sessions, add the session middleware to your kernel. The middleware will automatically handle session management, creating a `_session` attribute on your request object:

```php
<?php

use ForestCityLabs\Framework\Middleware\SessionMiddleware;

$middleware = new SessionMiddleware();
$kernel->addMiddleware($middleware);
$kernel->handle($request);
```

The `ForestCityLabs\Framework\Session\Session` class provides the API for interacting with the session:

```php
<?php

$session = $request->getAttribute('_session');
$session->setValue('hello', 'there');
```

Session data is automatically saved when the response is sent:

```php
<?php

$session = $request->getAttribute('_session');
print_r($session->getValue('hello'));
/**
 * Will output "there".
 */
```

### Session Security

For production environments, ensure proper session security:

```php
<?php

// Secure session configuration
ini_set('session.cookie_secure', true);     // HTTPS only
ini_set('session.cookie_httponly', true);  // No JavaScript access
ini_set('session.cookie_samesite', 'Strict'); // CSRF protection
ini_set('session.use_strict_mode', true);  // Prevent session fixation
```