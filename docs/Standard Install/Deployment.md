# Deployment

This guide covers deploying FCL Framework applications to production environments with best practices for security, performance, and reliability.

## Pre-Deployment Checklist

Before deploying to production, ensure:

- [ ] Environment variables configured for production
- [ ] Database migrations are ready
- [ ] GraphQL schema is validated  
- [ ] Security settings are properly configured
- [ ] Caching is enabled and configured
- [ ] Logging is properly set up
- [ ] SSL/TLS certificates are configured
- [ ] Error pages are customized
- [ ] Dependencies are optimized for production

## Environment Configuration

### Production Environment File

Create a production `.env` file:

```env
# Application
APP_ENV=production
APP_DEBUG=false
APP_NAME="Your Production App"
APP_SECRET=your-very-secure-secret-key-here

# Database
DATABASE_URL="mysql://user:secure_password@localhost:3306/production_db"

# Cache (Redis recommended for production)
CACHE_DRIVER=redis
REDIS_URL=tcp://redis-server:6379

# Sessions
SESSION_DRIVER=redis
SESSION_SECURE=true
SESSION_HTTPONLY=true
SESSION_SAMESITE=strict

# Security
JWT_SECRET=your-very-secure-jwt-secret-here
OAUTH_ENCRYPTION_KEY=base64:your-encryption-key-here

# CORS (restrict to your domains)
CORS_ALLOWED_ORIGINS=https://yourdomain.com,https://app.yourdomain.com

# GraphQL
GRAPHQL_INTROSPECTION=false
GRAPHQL_SCHEMA_CACHE=true
GRAPHQL_QUERY_COMPLEXITY_LIMIT=100

# Logging
LOG_LEVEL=warning
LOG_CHANNEL=production
```

### Security Configuration

#### JWT Key Generation

Generate secure JWT keys:

```bash
# Generate private key
openssl genrsa -out var/keys/jwt_private.pem 2048

# Generate public key
openssl rsa -in var/keys/jwt_private.pem -pubout -out var/keys/jwt_public.pem

# Set proper permissions
chmod 600 var/keys/jwt_private.pem
chmod 644 var/keys/jwt_public.pem
```

#### OAuth Key Generation

```bash
# Generate OAuth encryption key
php -r "echo 'OAUTH_ENCRYPTION_KEY=base64:' . base64_encode(random_bytes(32)) . PHP_EOL;"
```

## Docker Deployment

### Production Dockerfile

```dockerfile
# Dockerfile
FROM php:8.2-fpm-alpine

# Install system dependencies
RUN apk add --no-cache \
    nginx \
    supervisor \
    mysql-client \
    postgresql-client

# Install PHP extensions
RUN docker-php-ext-install \
    pdo \
    pdo_mysql \
    pdo_pgsql \
    opcache \
    bcmath

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Configure PHP for production
COPY docker/php/php.ini /usr/local/etc/php/
COPY docker/php/opcache.ini /usr/local/etc/php/conf.d/

# Configure Nginx
COPY docker/nginx/nginx.conf /etc/nginx/
COPY docker/nginx/default.conf /etc/nginx/conf.d/

# Configure Supervisor
COPY docker/supervisor/supervisord.conf /etc/supervisor/conf.d/

# Install Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy application files
COPY . .

# Install PHP dependencies (production)
RUN composer install --no-dev --optimize-autoloader --no-scripts

# Set permissions
RUN chown -R www-data:www-data var/ \
    && chmod -R 755 var/ \
    && chmod -R 777 var/cache/ var/log/ var/sessions/

# Run post-install scripts
RUN composer run-script post-install-cmd

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### Docker Compose for Production

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build: 
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      - APP_ENV=production
    volumes:
      - ./var/log:/var/www/html/var/log
      - ./var/keys:/var/www/html/var/keys:ro
    depends_on:
      - db
      - redis
    networks:
      - app-network

  db:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/ssl:/etc/nginx/ssl:ro
      - ./public:/var/www/html/public:ro
    depends_on:
      - app
    networks:
      - app-network

volumes:
  db_data:
  redis_data:

networks:
  app-network:
```

## Traditional Server Deployment

### System Requirements

- PHP 8.1 or higher
- Web server (Apache/Nginx)
- MySQL 8.0+ or PostgreSQL 13+
- Redis (recommended for caching/sessions)
- Composer

### Apache Configuration

```apache
# /etc/apache2/sites-available/your-app.conf
<VirtualHost *:80>
    ServerName yourdomain.com
    DocumentRoot /var/www/your-app/public
    
    <Directory /var/www/your-app/public>
        AllowOverride All
        Require all granted
        
        # Enable rewrite module
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d  
        RewriteRule ^ index.php [QSA,L]
    </Directory>
    
    # Security headers
    Header always set X-Content-Type-Options nosniff
    Header always set X-Frame-Options DENY
    Header always set X-XSS-Protection "1; mode=block"
    
    ErrorLog ${APACHE_LOG_DIR}/your-app_error.log
    CustomLog ${APACHE_LOG_DIR}/your-app_access.log combined
</VirtualHost>

# SSL version
<VirtualHost *:443>
    ServerName yourdomain.com
    DocumentRoot /var/www/your-app/public
    
    SSLEngine on
    SSLCertificateFile /path/to/certificate.crt
    SSLCertificateKeyFile /path/to/private.key
    SSLCertificateChainFile /path/to/chain.crt
    
    # Same directory and header configuration as above
</VirtualHost>
```

### Nginx Configuration

```nginx
# /etc/nginx/sites-available/your-app
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com;
    
    root /var/www/your-app/public;
    index index.php;
    
    # SSL configuration
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    
    # Security headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
    
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Security
        fastcgi_hide_header X-Powered-By;
        fastcgi_param HTTP_PROXY "";
    }
    
    # Deny access to sensitive files
    location ~ /\. {
        deny all;
    }
    
    location ~ /(composer|package)\.json$ {
        deny all;
    }
    
    location ~ /var/ {
        deny all;
    }
    
    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

## Deployment Scripts

### Automated Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -e

APP_DIR="/var/www/your-app"
BACKUP_DIR="/var/backups/your-app"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "Starting deployment..."

# Create backup
echo "Creating backup..."
mkdir -p $BACKUP_DIR
rsync -av --exclude 'var/cache' --exclude 'var/log' $APP_DIR/ $BACKUP_DIR/backup_$TIMESTAMP/

# Pull latest code
echo "Pulling latest code..."
cd $APP_DIR
git pull origin main

# Install/update dependencies
echo "Installing dependencies..."
composer install --no-dev --optimize-autoloader

# Clear and warm caches
echo "Clearing caches..."
./vendor/bin/fcl cache:clear

# Run database migrations
echo "Running migrations..."
./vendor/bin/doctrine-migrations migrate --no-interaction

# Generate Doctrine proxies
echo "Generating Doctrine proxies..."
./vendor/bin/doctrine orm:generate-proxies

# Validate GraphQL schema
echo "Validating GraphQL schema..."
./vendor/bin/fcl graphql:validate-schema

# Set permissions
echo "Setting permissions..."
chown -R www-data:www-data var/
chmod -R 755 var/
chmod -R 777 var/cache/ var/log/ var/sessions/

# Reload web server
echo "Reloading web server..."
systemctl reload nginx
# or: systemctl reload apache2

echo "Deployment completed successfully!"

# Test deployment
echo "Testing deployment..."
curl -f https://yourdomain.com/health || {
    echo "Health check failed! Rolling back..."
    rsync -av --delete $BACKUP_DIR/backup_$TIMESTAMP/ $APP_DIR/
    exit 1
}

echo "Deployment verified!"
```

### Zero-Downtime Deployment

```bash
#!/bin/bash
# zero-downtime-deploy.sh

APP_DIR="/var/www/your-app"
RELEASES_DIR="$APP_DIR/releases"
SHARED_DIR="$APP_DIR/shared"
CURRENT_DIR="$APP_DIR/current"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RELEASE_DIR="$RELEASES_DIR/$TIMESTAMP"

echo "Starting zero-downtime deployment..."

# Create directory structure
mkdir -p $RELEASES_DIR $SHARED_DIR

# Clone repository to new release directory
echo "Creating new release..."
git clone --depth 1 https://github.com/yourusername/your-app.git $RELEASE_DIR
cd $RELEASE_DIR

# Link shared directories
echo "Linking shared directories..."
rm -rf var/log var/sessions var/keys
ln -s $SHARED_DIR/log var/log
ln -s $SHARED_DIR/sessions var/sessions  
ln -s $SHARED_DIR/keys var/keys

# Copy environment file
cp $SHARED_DIR/.env .env

# Install dependencies
echo "Installing dependencies..."
composer install --no-dev --optimize-autoloader

# Run migrations (on shared database)
echo "Running migrations..."
./vendor/bin/doctrine-migrations migrate --no-interaction

# Validate application
echo "Validating application..."
./vendor/bin/fcl graphql:validate-schema

# Atomic switch to new release
echo "Switching to new release..."
ln -sfn $RELEASE_DIR $CURRENT_DIR

# Reload web server
systemctl reload nginx

# Cleanup old releases (keep last 3)
echo "Cleaning up old releases..."
cd $RELEASES_DIR
ls -1t | tail -n +4 | xargs rm -rf

echo "Zero-downtime deployment completed!"
```

## Health Checks

### Health Check Endpoint

Create a health check endpoint:

```php
<?php
// src/Controller/HealthController.php

namespace Application\Controller;

use ForestCityLabs\Framework\Routing\Attribute\Route;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ResponseFactoryInterface;
use Doctrine\DBAL\Connection;

class HealthController
{
    public function __construct(
        private ResponseFactoryInterface $responseFactory,
        private Connection $connection
    ) {}
    
    #[Route('/health')]
    public function health(): ResponseInterface
    {
        $checks = [
            'database' => $this->checkDatabase(),
            'cache' => $this->checkCache(),
            'disk_space' => $this->checkDiskSpace(),
        ];
        
        $overall = array_reduce($checks, fn($carry, $check) => $carry && $check, true);
        $status = $overall ? 200 : 503;
        
        $response = $this->responseFactory->createResponse($status);
        $response->getBody()->write(json_encode([
            'status' => $overall ? 'healthy' : 'unhealthy',
            'checks' => $checks,
            'timestamp' => date('c')
        ]));
        
        return $response->withHeader('Content-Type', 'application/json');
    }
    
    private function checkDatabase(): bool
    {
        try {
            $this->connection->executeQuery('SELECT 1');
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }
    
    private function checkCache(): bool
    {
        // Implement cache health check
        return true;
    }
    
    private function checkDiskSpace(): bool
    {
        $freeSpace = disk_free_space('/');
        $totalSpace = disk_total_space('/');
        return ($freeSpace / $totalSpace) > 0.1; // At least 10% free
    }
}
```

### Load Balancer Health Check

Configure load balancer health checks:

```nginx
# Nginx upstream health check
upstream app_servers {
    server app1.internal:9000 max_fails=3 fail_timeout=30s;
    server app2.internal:9000 max_fails=3 fail_timeout=30s;
    server app3.internal:9000 max_fails=3 fail_timeout=30s backup;
}

server {
    location /health {
        access_log off;
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_connect_timeout 5s;
        proxy_read_timeout 5s;
    }
}
```

## Monitoring

### Application Monitoring

Monitor key metrics:

```php
// src/EventListener/PerformanceListener.php
use ForestCityLabs\Framework\Event\Attribute\EventListener;
use ForestCityLabs\Framework\Events\PostMiddlewareHandleEvent;

#[EventListener(PostMiddlewareHandleEvent::class)]
class PerformanceListener  
{
    public function __invoke(PostMiddlewareHandleEvent $event): void
    {
        $request = $event->getRequest();
        $response = $event->getResponse();
        
        // Log performance metrics
        $this->logger->info('Request completed', [
            'method' => $request->getMethod(),
            'uri' => (string) $request->getUri(),
            'status' => $response->getStatusCode(),
            'memory_usage' => memory_get_peak_usage(true),
            'execution_time' => $request->getAttribute('execution_time'),
        ]);
    }
}
```

### Log Aggregation

Configure structured logging for aggregation:

```php
// config/services.php
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Formatter\JsonFormatter;

return [
    'logger' => DI\factory(function () {
        $logger = new Logger('app');
        $handler = new StreamHandler('/var/log/app.json', Logger::INFO);
        $handler->setFormatter(new JsonFormatter());
        $logger->pushHandler($handler);
        return $logger;
    }),
];
```

## Security Considerations

### File Permissions

Set appropriate file permissions:

```bash
# Application files
find /var/www/your-app -type f -exec chmod 644 {} \;
find /var/www/your-app -type d -exec chmod 755 {} \;

# Writable directories
chmod -R 777 /var/www/your-app/var/cache/
chmod -R 777 /var/www/your-app/var/log/  
chmod -R 777 /var/www/your-app/var/sessions/

# Sensitive files
chmod 600 /var/www/your-app/var/keys/*
chmod 600 /var/www/your-app/.env
```

### Database Security

- Use dedicated database users with minimal privileges
- Enable SSL for database connections
- Regular security updates
- Database firewall rules

### Web Server Security

- Keep web server updated
- Configure security headers
- Disable server signatures
- Use fail2ban for brute force protection
- Regular security audits

## Backup Strategy

### Database Backups

```bash
#!/bin/bash
# backup-db.sh

BACKUP_DIR="/var/backups/database"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# MySQL backup
mysqldump -u $DB_USER -p$DB_PASSWORD $DB_NAME > $BACKUP_DIR/db_$TIMESTAMP.sql

# Compress backup
gzip $BACKUP_DIR/db_$TIMESTAMP.sql

# Remove backups older than 30 days
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
```

### File System Backups

```bash
#!/bin/bash
# backup-files.sh

rsync -av --exclude 'var/cache' --exclude 'var/log' \
    /var/www/your-app/ \
    backup-server:/backups/your-app/$(date +%Y%m%d)/
```

## Troubleshooting

### Common Issues

1. **Permission errors**: Check file ownership and permissions
2. **Database connection issues**: Verify credentials and network connectivity
3. **Cache issues**: Clear cache directories and restart services  
4. **Memory issues**: Monitor PHP memory limits and optimize code
5. **SSL certificate issues**: Check certificate validity and configuration

### Log Analysis

Monitor application logs for issues:

```bash
# Real-time log monitoring
tail -f /var/www/your-app/var/log/prod.log

# Error analysis
grep "ERROR" /var/www/your-app/var/log/prod.log | tail -20

# Performance analysis  
grep "execution_time" /var/www/your-app/var/log/prod.log | awk '{print $NF}' | sort -n
```