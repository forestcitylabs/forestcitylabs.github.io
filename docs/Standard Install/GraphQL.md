# GraphQL

This framework is primarily interested in creating GraphQL APIs and that is easy to start doing either using a **schema-first** or **code-first** approach.

## Overview

The FCL Framework provides comprehensive GraphQL support built on top of [webonyx/graphql-php](https://github.com/webonyx/graphql-php) with additional features for:

- Attribute-based type definitions
- Automatic schema generation
- Field resolvers and transformers
- Schema comparison and migration tools
- Built-in GraphiQL interface

## Getting Started

### Schema-First Approach

Start by defining your GraphQL schema in a `.graphql` file:

```graphql title="schema.graphql"
type Query {
  users: [User!]!
  user(id: ID!): User
}

type Mutation {
  createUser(input: UserInput!): User!
  updateUser(id: ID!, input: UserInput!): User!
}

type User {
  id: ID!
  name: String!
  email: String!
  createdAt: String!
}

input UserInput {
  name: String!
  email: String!
}
```

Then generate PHP classes from your schema:

```bash
./vendor/bin/fcl graphql:generate-from-schema schema.graphql src/GraphQL
```

This creates type classes with attributes that you can customize:

```php title="Generated User type"
<?php

namespace Application\GraphQL\Type;

use ForestCityLabs\Framework\GraphQL\Attribute\ObjectType;
use ForestCityLabs\Framework\GraphQL\Attribute\Field;

#[ObjectType]
class User
{
    #[Field(type: 'ID!')]
    public function id(): string
    {
        return $this->user->getId();
    }
    
    #[Field(type: 'String!')]
    public function name(): string  
    {
        return $this->user->getName();
    }
    
    #[Field(type: 'String!')]
    public function email(): string
    {
        return $this->user->getEmail();
    }
    
    #[Field(type: 'String!')]
    public function createdAt(): string
    {
        return $this->user->getCreatedAt()->format('c');
    }
}
```

### Code-First Approach

Define types directly using PHP attributes:

```php title="User GraphQL Type"
<?php

namespace Application\GraphQL\Type;

use ForestCityLabs\Framework\GraphQL\Attribute\ObjectType;
use ForestCityLabs\Framework\GraphQL\Attribute\Field;
use Application\Entity\User as UserEntity;

#[ObjectType(name: 'User')]
class User  
{
    public function __construct(
        private UserEntity $user
    ) {}
    
    #[Field(type: 'ID!')]
    public function id(): string
    {
        return $this->user->getId();
    }
    
    #[Field(type: 'String!')]
    public function name(): string
    {
        return $this->user->getName();
    }
    
    #[Field(type: 'String!')]  
    public function email(): string
    {
        return $this->user->getEmail();
    }
    
    #[Field(type: 'DateTime!')]
    public function createdAt(): \DateTimeInterface
    {
        return $this->user->getCreatedAt();
    }
}
```

### Query Types

Define your root Query type:

```php title="Query Type"
<?php

namespace Application\GraphQL\Type;

use ForestCityLabs\Framework\GraphQL\Attribute\Query;
use ForestCityLabs\Framework\GraphQL\Attribute\Field;
use ForestCityLabs\Framework\GraphQL\Attribute\Argument;
use Application\Repository\UserRepository;

class Query
{
    public function __construct(
        private UserRepository $userRepository
    ) {}
    
    #[Query]
    #[Field(type: '[User!]!')]
    public function users(): array
    {
        $users = $this->userRepository->findAll();
        return array_map(fn($user) => new User($user), $users);
    }
    
    #[Query]  
    #[Field(type: 'User')]
    #[Argument('id', type: 'ID!')]
    public function user(string $id): ?User
    {
        $user = $this->userRepository->find($id);
        return $user ? new User($user) : null;
    }
}
```

### Mutation Types

Define mutations for data modification:

```php title="Mutation Type"
<?php

namespace Application\GraphQL\Type;

use ForestCityLabs\Framework\GraphQL\Attribute\Mutation;
use ForestCityLabs\Framework\GraphQL\Attribute\Field;
use ForestCityLabs\Framework\GraphQL\Attribute\Argument;
use Application\Repository\UserRepository;
use Application\Entity\User as UserEntity;

class Mutation
{
    public function __construct(
        private UserRepository $userRepository
    ) {}
    
    #[Mutation]
    #[Field(type: 'User!')]
    #[Argument('input', type: 'UserInput!')]
    public function createUser(array $input): User
    {
        $user = new UserEntity();
        $user->setName($input['name']);
        $user->setEmail($input['email']);
        
        $this->userRepository->save($user);
        
        return new User($user);
    }
    
    #[Mutation]
    #[Field(type: 'User!')]
    #[Argument('id', type: 'ID!')]
    #[Argument('input', type: 'UserInput!')]
    public function updateUser(string $id, array $input): User
    {
        $user = $this->userRepository->find($id);
        if (!$user) {
            throw new \Exception('User not found');
        }
        
        $user->setName($input['name']);
        $user->setEmail($input['email']);
        
        $this->userRepository->save($user);
        
        return new User($user);
    }
}
```

### Input Types

Define input types for mutations:

```php title="UserInput Type"
<?php

namespace Application\GraphQL\Input;

use ForestCityLabs\Framework\GraphQL\Attribute\InputType;
use ForestCityLabs\Framework\GraphQL\Attribute\Field;

#[InputType(name: 'UserInput')]
class UserInput
{
    #[Field(type: 'String!')]
    public string $name;
    
    #[Field(type: 'String!')]
    public string $email;
}
```

## Advanced Features

### Enum Types

Define GraphQL enums:

```php title="Status Enum"
<?php

namespace Application\GraphQL\Enum;

use ForestCityLabs\Framework\GraphQL\Attribute\EnumType;
use ForestCityLabs\Framework\GraphQL\Attribute\Value;

#[EnumType(name: 'UserStatus')]
enum UserStatus: string
{
    #[Value(value: 'ACTIVE')]
    case ACTIVE = 'active';
    
    #[Value(value: 'INACTIVE')]  
    case INACTIVE = 'inactive';
    
    #[Value(value: 'SUSPENDED')]
    case SUSPENDED = 'suspended';
}
```

### Interface Types

Define GraphQL interfaces:

```php title="Node Interface"
<?php

namespace Application\GraphQL\Interface;

use ForestCityLabs\Framework\GraphQL\Attribute\InterfaceType;
use ForestCityLabs\Framework\GraphQL\Attribute\Field;

#[InterfaceType(name: 'Node')]
interface Node
{
    #[Field(type: 'ID!')]
    public function id(): string;
}
```

### Field Resolvers

The framework supports multiple field resolution strategies:

#### Method-Based Resolution (Default)

Fields are resolved by calling methods on the type class:

```php
#[Field(type: 'String!')]
public function fullName(): string
{
    return $this->user->getFirstName() . ' ' . $this->user->getLastName();
}
```

#### Property-Based Resolution

Fields can resolve directly to object properties:

```php
#[ObjectType(name: 'User')]
class User
{
    #[Field(type: 'String!', resolver: 'property')]
    public string $name;
    
    #[Field(type: 'String!', resolver: 'property')]  
    public string $email;
}
```

### Transformers

Transform PHP values to GraphQL types:

```php title="DateTime Transformer"
use ForestCityLabs\Framework\GraphQL\Transformer\DateTimeImmutableTransformer;

// Automatically transforms DateTimeInterface to ISO string
#[Field(type: 'DateTime!')]
public function createdAt(): \DateTimeInterface
{
    return $this->user->getCreatedAt(); // Transformed to ISO string
}
```

## Schema Management

### Dump Schema

Export your schema to a `.graphql` file:

```bash
./vendor/bin/fcl graphql:dump-schema > schema.graphql
```

### Schema Validation

Validate your schema for errors:

```bash
./vendor/bin/fcl graphql:validate-schema
```

### Schema Comparison

Compare schemas for breaking changes:

```bash
./vendor/bin/fcl graphql:schema-diff schema-old.graphql schema-new.graphql
```

## GraphQL Middleware Setup

Add GraphQL middleware to your kernel:

```php title="Kernel configuration"
use ForestCityLabs\Framework\Middleware\GraphQLMiddleware;
use ForestCityLabs\Framework\GraphQL\TypeRegistry;

// Create type registry with your types
$type_registry = new TypeRegistry();
$type_registry->addType(Query::class);
$type_registry->addType(Mutation::class);
$type_registry->addType(User::class);

// Create GraphQL middleware
$graphql_middleware = new GraphQLMiddleware(
    $type_registry,
    $container,
    $logger
);

$kernel->addMiddleware($graphql_middleware);
```

## GraphiQL Interface

The framework includes GraphiQL for testing queries. Add the GraphiQL middleware:

```php
use ForestCityLabs\Framework\Middleware\GraphiQLMiddleware;

$graphiql_middleware = new GraphiQLMiddleware(
    '/graphql',  // GraphQL endpoint
    '/graphiql', // GraphiQL interface URL
    $response_factory
);

$kernel->addMiddleware($graphiql_middleware);
```

Visit `/graphiql` to access the interactive query interface.

## Security Integration

GraphQL integrates with the framework's security system:

```php
use ForestCityLabs\Framework\Security\Attribute\RequiresRole;
use ForestCityLabs\Framework\Security\Attribute\RequiresScope;

class Mutation
{
    #[Mutation]
    #[Field(type: 'User!')]
    #[RequiresRole('admin')]
    public function createUser(array $input): User
    {
        // Only accessible to admin users
    }
    
    #[Mutation]
    #[Field(type: 'User!')]  
    #[RequiresScope('write')]
    public function updateUser(string $id, array $input): User
    {
        // Requires 'write' OAuth scope
    }
}
```

## Error Handling

GraphQL errors are automatically formatted:

```php
#[Query]
public function user(string $id): User
{
    $user = $this->userRepository->find($id);
    if (!$user) {
        throw new \Exception('User not found'); // Becomes GraphQL error
    }
    
    return new User($user);
}
```

## Performance Considerations

### Query Complexity Analysis

The framework supports query complexity analysis to prevent expensive queries:

```php
$graphql_middleware = new GraphQLMiddleware(
    $type_registry,
    $container,
    $logger,
    [
        'queryComplexity' => 100,  // Maximum query complexity
        'introspection' => false,  // Disable introspection in production
    ]
);
```

### DataLoader Pattern

Implement the DataLoader pattern to solve N+1 query problems:

```php
use ForestCityLabs\Framework\Routing\DataLoader;

#[Field(type: 'User')]
public function author(DataLoader $userLoader): Promise  
{
    return $userLoader->load($this->post->getAuthorId());
}
```

## Testing GraphQL APIs

Test your GraphQL API using PHPUnit:

```php
use PHPUnit\Framework\TestCase;

class GraphQLTest extends TestCase
{
    public function testUserQuery(): void
    {
        $query = '
            query GetUser($id: ID!) {
                user(id: $id) {
                    id
                    name
                    email
                }
            }
        ';
        
        $result = $this->executeQuery($query, ['id' => '123']);
        
        $this->assertArrayHasKey('data', $result);
        $this->assertArrayHasKey('user', $result['data']);
    }
}
