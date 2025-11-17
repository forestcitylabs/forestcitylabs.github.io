# Configuration Overview

The standard install provides a pre-configured setup with sensible defaults. This guide provides an overview of the configuration system and links to detailed configuration topics.

## Quick Start

The FCL Framework requires only three environment variables to get started:

```env
# Copy the template and customize
cp .env.dist .env

# Required configuration
ENVIRONMENT=development
DATABASE_URI="mysql://user:password@localhost:3306/database_name"
TRUSTED_HOST=localhost
```

After setting these variables, you can start the development server and begin building your application.

## Configuration Topics

### [Environment Configuration](./Environment)

Learn how to configure environment variables, PHP settings, and environment-specific configurations for development, testing, and production.

**Topics covered:**
- Environment variables setup
- Development vs production settings  
- PHP configuration
- Security considerations
- Environment validation

### [Service Configuration](./Services)

Understand how to configure services using PHP-DI dependency injection, including core framework services and custom application services.

**Topics covered:**
- Service definition and registration
- Factory services and singletons
- Core framework services (caching, logging, events)
- Third-party API integration
- Environment-specific services

### [Kernel and Middleware](./Kernel)

Configure the HTTP request/response pipeline using the KernelFactory pattern and middleware configuration.

**Topics covered:**
- KernelFactory setup and usage
- Middleware configuration and ordering
- Environment-specific middleware stacks
- Custom middleware creation
- Advanced middleware patterns

### [Database Configuration](./Database)

Set up and manage database connections, entities, migrations, and repository patterns using Doctrine DBAL and ORM.

**Topics covered:**
- Database connection setup
- Entity configuration and relationships
- Migration management
- Repository patterns
- Performance optimization

### [Docker and Deployment](./Docker)

Use Docker for local development and production deployment, including container orchestration and monitoring.

**Topics covered:**
- Docker Compose setup
- Multi-stage Dockerfile
- Production deployment strategies
- Container orchestration (Swarm, Kubernetes)
- Monitoring and health checks

## Configuration Architecture

### Configuration Layers

The framework uses a layered configuration approach:

1. **Environment Variables** - Runtime configuration via `.env` files
2. **Service Definitions** - Dependency injection configuration in `config/services.php`
3. **Application Factory** - Kernel and middleware setup via `KernelFactory`
4. **Runtime Settings** - PHP configuration and feature flags

### Best Practices

1. **Environment-based Configuration** - Use different configurations for development, testing, and production
2. **Security First** - Never commit secrets, use secure defaults
3. **Validation** - Validate configuration at application startup
4. **Documentation** - Document configuration requirements clearly
5. **Defaults** - Provide sensible defaults for optional settings

## Common Patterns

### Factory Pattern

Use factory classes for complex service configuration:

```php
class DatabaseFactory
{
    public static function create(array $config): Connection
    {
        return DriverManager::getConnection($config);
    }
}
```

### Configuration Objects

Create configuration value objects for complex settings:

```php
class CacheConfig
{
    public function __construct(
        public readonly string $driver,
        public readonly int $ttl,
        public readonly array $options
    ) {}
}
```

### Environment Detection

Use environment detection for conditional configuration:

```php
$environment = $_ENV['ENVIRONMENT'] ?? 'production';

return match($environment) {
    'development' => $developmentConfig,
    'testing' => $testingConfig,
    default => $productionConfig
};
```

## Next Steps

1. **Choose your configuration approach** - Start with [Environment Configuration](./Environment)
2. **Set up services** - Configure your application services in [Service Configuration](./Services)
3. **Configure middleware** - Set up request processing in [Kernel and Middleware](./Kernel)
4. **Set up the database** - Configure data persistence in [Database Configuration](./Database)
5. **Prepare for deployment** - Set up containers in [Docker and Deployment](./Docker)

## Troubleshooting

### Configuration Issues

- **Environment variables not loading** - Check `.env` file location and permissions
- **Service resolution errors** - Verify service definitions in `config/services.php`
- **Database connection failures** - Validate `DATABASE_URI` format and connectivity
- **Middleware not executing** - Check middleware order in `KernelFactory`

### Getting Help

- Check the specific configuration topic pages for detailed guidance
- Review the troubleshooting sections in each configuration area
- Validate your configuration using the framework's built-in validation tools
- Ensure environment variables match the expected format and values

Each configuration topic page provides comprehensive examples, best practices, and troubleshooting guidance for that specific area.