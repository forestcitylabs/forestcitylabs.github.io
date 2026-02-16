# Routing

The standard install provides automatic route discovery using PHP attributes. Controllers are placed in the `src/Controller/` directory and routes are defined using the `#[Route]` and `#[RoutePrefix]` attributes.

## Basic Routing

### Simple Routes

```php
<?php

namespace Application\Controller;

use ForestCityLabs\Framework\Routing\Attribute\Route;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ResponseFactoryInterface;

class HomeController
{
    public function __construct(
        private ResponseFactoryInterface $responseFactory
    ) {}
    
    #[Route('/')]
    public function index(): ResponseInterface
    {
        $response = $this->responseFactory->createResponse();
        $response->getBody()->write('<h1>Welcome to FCL Framework!</h1>');
        return $response;
    }
    
    #[Route('/about')]
    public function about(): ResponseInterface
    {
        $response = $this->responseFactory->createResponse();
        $response->getBody()->write('<h1>About Us</h1>');
        return $response;
    }
}
```

### HTTP Methods

Specify HTTP methods for routes:

```php
#[Route('/users', methods: ['GET'])]
public function listUsers(): ResponseInterface
{
    // Handle GET /users
}

#[Route('/users', methods: ['POST'])]
public function createUser(): ResponseInterface  
{
    // Handle POST /users
}

#[Route('/users/{id}', methods: ['PUT', 'PATCH'])]
public function updateUser(int $id): ResponseInterface
{
    // Handle PUT or PATCH /users/123
}

#[Route('/users/{id}', methods: ['DELETE'])]
public function deleteUser(int $id): ResponseInterface
{
    // Handle DELETE /users/123
}
```

## Route Prefixes

Group related routes using `#[RoutePrefix]`:

```php
<?php

namespace Application\Controller;

use ForestCityLabs\Framework\Routing\Attribute\Route;
use ForestCityLabs\Framework\Routing\Attribute\RoutePrefix;

#[RoutePrefix('/api/v1')]
class ApiController
{
    #[Route('/users')]  // Results in: /api/v1/users
    public function getUsers(): ResponseInterface
    {
        // Return JSON response with users
    }
    
    #[Route('/users/{id}')]  // Results in: /api/v1/users/{id}
    public function getUser(int $id): ResponseInterface
    {
        // Return JSON response with specific user
    }
    
    #[Route('/posts')]  // Results in: /api/v1/posts
    public function getPosts(): ResponseInterface
    {
        // Return JSON response with posts
    }
}
```

## Route Parameters

### Path Parameters

Extract parameters from the URL:

```php
#[Route('/users/{id}')]
public function getUser(int $id): ResponseInterface
{
    // $id is automatically extracted and type-converted
}

#[Route('/users/{id}/posts/{postId}')]
public function getUserPost(int $id, int $postId): ResponseInterface
{
    // Multiple parameters
}

#[Route('/blog/{slug}')]
public function getBlogPost(string $slug): ResponseInterface
{
    // String parameters
}
```

### Optional Parameters

Define optional route parameters:

```php
#[Route('/search/{query?}')]
public function search(string $query = ''): ResponseInterface
{
    // Query parameter is optional
    if (empty($query)) {
        // Handle empty search
    }
}

#[Route('/posts/{page?}')]  
public function listPosts(int $page = 1): ResponseInterface
{
    // Page parameter defaults to 1
}
```

### Parameter Constraints

Use regex constraints for parameters:

```php
#[Route('/users/{id}', requirements: ['id' => '\d+'])]
public function getUser(int $id): ResponseInterface
{
    // ID must be numeric
}

#[Route('/posts/{year}/{month}', requirements: [
    'year' => '\d{4}',
    'month' => '\d{1,2}'
])]
public function getPostsByDate(int $year, int $month): ResponseInterface
{
    // Year must be 4 digits, month 1-2 digits
}
```

## Request Handling

### Accessing Request Data

```php
use Psr\Http\Message\ServerRequestInterface;

#[Route('/contact', methods: ['POST'])]
public function contact(ServerRequestInterface $request): ResponseInterface
{
    // Get POST data
    $data = $request->getParsedBody();
    $name = $data['name'] ?? '';
    $email = $data['email'] ?? '';
    
    // Get query parameters
    $params = $request->getQueryParams();
    $source = $params['source'] ?? '';
    
    // Get headers
    $userAgent = $request->getHeaderLine('User-Agent');
    
    // Get uploaded files
    $files = $request->getUploadedFiles();
    $avatar = $files['avatar'] ?? null;
    
    // Process contact form...
}
```

### JSON API Endpoints

Handle JSON requests and responses:

```php
#[Route('/api/users', methods: ['POST'])]
public function createUser(ServerRequestInterface $request): ResponseInterface
{
    // Parse JSON input
    $input = json_decode($request->getBody()->getContents(), true);
    
    // Validate input
    if (empty($input['name']) || empty($input['email'])) {
        return $this->jsonError('Name and email are required', 400);
    }
    
    // Create user logic...
    $user = $this->userService->create($input);
    
    // Return JSON response
    return $this->jsonResponse(['user' => $user], 201);
}

private function jsonResponse(array $data, int $status = 200): ResponseInterface
{
    $response = $this->responseFactory->createResponse($status);
    $response->getBody()->write(json_encode($data));
    return $response->withHeader('Content-Type', 'application/json');
}

private function jsonError(string $message, int $status = 400): ResponseInterface  
{
    return $this->jsonResponse(['error' => $message], $status);
}
```

## Dependency Injection

### Controller Dependencies

Inject services into controller constructors:

```php
<?php

namespace Application\Controller;

use Application\Service\UserService;
use Application\Service\EmailService;
use Psr\Log\LoggerInterface;

class UserController
{
    public function __construct(
        private UserService $userService,
        private EmailService $emailService,
        private LoggerInterface $logger,
        private ResponseFactoryInterface $responseFactory
    ) {}
    
    #[Route('/users/{id}')]
    public function getUser(int $id): ResponseInterface
    {
        $user = $this->userService->findById($id);
        
        if (!$user) {
            return $this->responseFactory->createResponse(404);
        }
        
        return $this->jsonResponse(['user' => $user]);
    }
}
```

### Method-Level Injection

Inject services directly into route methods:

```php
#[Route('/users/{id}/send-welcome')]
public function sendWelcome(
    int $id,
    UserService $userService,
    EmailService $emailService
): ResponseInterface {
    $user = $userService->findById($id);
    $emailService->sendWelcomeEmail($user);
    
    return $this->jsonResponse(['message' => 'Welcome email sent']);
}
```

## Response Types

### HTML Responses

Return HTML using Twig templates:

```php
use Twig\Environment;

class PageController
{
    public function __construct(
        private Environment $twig,
        private ResponseFactoryInterface $responseFactory
    ) {}
    
    #[Route('/profile/{id}')]
    public function profile(int $id, UserService $userService): ResponseInterface
    {
        $user = $userService->findById($id);
        
        $html = $this->twig->render('profile.html.twig', [
            'user' => $user
        ]);
        
        $response = $this->responseFactory->createResponse();
        $response->getBody()->write($html);
        return $response->withHeader('Content-Type', 'text/html');
    }
}
```

### File Downloads

Return file downloads:

```php
#[Route('/download/{file}')]
public function download(string $file): ResponseInterface
{
    $filePath = '/path/to/files/' . $file;
    
    if (!file_exists($filePath)) {
        return $this->responseFactory->createResponse(404);
    }
    
    $response = $this->responseFactory->createResponse();
    $response->getBody()->write(file_get_contents($filePath));
    
    return $response
        ->withHeader('Content-Type', 'application/octet-stream')
        ->withHeader('Content-Disposition', 'attachment; filename="' . $file . '"');
}
```

### Redirects

Redirect to other routes:

```php
#[Route('/login', methods: ['POST'])]
public function login(ServerRequestInterface $request): ResponseInterface
{
    // Authentication logic...
    
    if ($authenticated) {
        // Redirect to dashboard
        return $this->responseFactory
            ->createResponse(302)
            ->withHeader('Location', '/dashboard');
    }
    
    // Redirect back to login form
    return $this->responseFactory
        ->createResponse(302)
        ->withHeader('Location', '/login?error=invalid_credentials');
}
```

## Error Handling

### Custom Error Responses

```php
#[Route('/users/{id}')]
public function getUser(int $id): ResponseInterface
{
    try {
        $user = $this->userService->findById($id);
        
        if (!$user) {
            throw new UserNotFoundException("User {$id} not found");
        }
        
        return $this->jsonResponse(['user' => $user]);
        
    } catch (UserNotFoundException $e) {
        return $this->jsonError($e->getMessage(), 404);
    } catch (\Exception $e) {
        $this->logger->error('Unexpected error in getUser', [
            'user_id' => $id,
            'error' => $e->getMessage()
        ]);
        
        return $this->jsonError('Internal server error', 500);
    }
}
```

## Route Caching

In production, route metadata is automatically cached for performance. To clear the route cache:

```bash
./vendor/bin/console cache:clear
```

## Security

### Authentication Required

```php
use ForestCityLabs\Framework\Security\Attribute\RequiresRole;

class AdminController
{
    #[Route('/admin/users')]
    #[RequiresRole('admin')]
    public function listUsers(): ResponseInterface
    {
        // Only accessible to authenticated users with 'admin' role
    }
}
```

### CSRF Protection

For form submissions, implement CSRF protection:

```php
#[Route('/profile', methods: ['POST'])]
public function updateProfile(ServerRequestInterface $request): ResponseInterface
{
    // Validate CSRF token
    $token = $request->getParsedBody()['_token'] ?? '';
    if (!$this->csrfService->isValid($token)) {
        return $this->jsonError('Invalid CSRF token', 403);
    }
    
    // Process form...
}
```

## Testing Routes

Test your routes using PHPUnit:

```php
use PHPUnit\Framework\TestCase;
use GuzzleHttp\Psr7\ServerRequest;

class UserControllerTest extends TestCase
{
    public function testGetUser(): void
    {
        $kernel = $this->createKernel();
        $request = new ServerRequest('GET', '/users/123');
        
        $response = $kernel->handle($request);
        
        $this->assertEquals(200, $response->getStatusCode());
        
        $data = json_decode($response->getBody()->getContents(), true);
        $this->assertArrayHasKey('user', $data);
    }
    
    public function testGetUserNotFound(): void
    {
        $kernel = $this->createKernel();
        $request = new ServerRequest('GET', '/users/999');
        
        $response = $kernel->handle($request);
        
        $this->assertEquals(404, $response->getStatusCode());
    }
}
