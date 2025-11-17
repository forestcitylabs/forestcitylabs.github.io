# Docker and Deployment

The FCL Framework standard install includes Docker support for local development and production deployment. This guide covers Docker configuration, deployment strategies, and production optimizations.

## Docker Configuration

### Docker Compose Setup

The standard install includes Docker support with `docker-compose.yml`:

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:80"
    environment:
      - ENVIRONMENT=development
      - DATABASE_URI=mysql://root:password@db:3306/app
      - TRUSTED_HOST=localhost
    volumes:
      - .:/var/www/html
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: app
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  db_data:
```

### Dockerfile

Create a multi-stage Dockerfile for optimized builds:

```dockerfile
# Dockerfile
FROM php:8.2-fpm-alpine AS base

# Install system dependencies
RUN apk add --no-cache \
    nginx \
    supervisor \
    git \
    unzip \
    curl \
    && docker-php-ext-install pdo pdo_mysql

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Configure nginx
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/default.conf /etc/nginx/conf.d/default.conf

# Configure supervisor
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

WORKDIR /var/www/html

# Development stage
FROM base AS development

# Install development dependencies
RUN apk add --no-cache \
    php82-xdebug

# Copy Xdebug configuration
COPY docker/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini

# Copy application files
COPY . .

# Install dependencies
RUN composer install --no-scripts --no-autoloader

# Generate autoloader and run post-install scripts
RUN composer dump-autoload --optimize && composer run-script post-install-cmd

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

# Production stage
FROM base AS production

# Copy application files (excluding dev dependencies)
COPY . .

# Install production dependencies
RUN composer install --no-dev --optimize-autoloader --no-scripts

# Generate optimized autoloader
RUN composer dump-autoload --optimize --classmap-authoritative

# Run post-install scripts
RUN composer run-script post-install-cmd

# Set proper permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html \
    && chmod -R 777 /var/www/html/var

EXPOSE 80

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### Docker Configuration Files

#### Nginx Configuration

```nginx
# docker/nginx.conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# docker/default.conf
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Increase timeouts for GraphQL queries
        fastcgi_read_timeout 300;
        fastcgi_send_timeout 300;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

#### Supervisor Configuration

```ini
# docker/supervisord.conf
[supervisord]
nodaemon=true
user=root
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid

[program:nginx]
command=nginx -g "daemon off;"
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
autorestart=true
startretries=0

[program:php-fpm]
command=php-fpm -F
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
autorestart=true
startretries=0
```

## Running with Docker

### Development Commands

```bash
# Start development environment
docker compose up -d

# View logs
docker compose logs -f app

# Run migrations
docker compose exec app ./vendor/bin/console doctrine:migrations migrate

# Install dependencies
docker compose exec app composer install

# Access container shell
docker compose exec app sh

# Stop services
docker compose down

# Rebuild containers
docker compose build --no-cache
```

### Environment-Specific Configurations

#### Development Override

```yaml
# docker-compose.override.yml
version: '3.8'

services:
  app:
    build:
      target: development
    environment:
      - ENVIRONMENT=development
      - XDEBUG_MODE=debug
      - XDEBUG_START_WITH_REQUEST=yes
    volumes:
      - .:/var/www/html
    ports:
      - "9003:9003" # Xdebug port
```

#### Production Configuration

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      target: production
    environment:
      - ENVIRONMENT=production
      - DATABASE_URI=${DATABASE_URI}
      - TRUSTED_HOST=${TRUSTED_HOST}
    restart: unless-stopped
    
  db:
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    restart: unless-stopped
    
  redis:
    restart: unless-stopped
```

## Production Deployment

### Build Process

Create a deployment script:

```bash
#!/bin/bash
# deploy.sh

set -e

echo "Starting deployment..."

# Build production image
docker build --target production -t myapp:latest .

# Stop existing containers
docker compose -f docker-compose.prod.yml down

# Pull latest images
docker compose -f docker-compose.prod.yml pull

# Start new containers
docker compose -f docker-compose.prod.yml up -d

# Run migrations
docker compose -f docker-compose.prod.yml exec -T app ./vendor/bin/console doctrine:migrations migrate --no-interaction

# Clear caches
docker compose -f docker-compose.prod.yml exec -T app ./vendor/bin/console cache:clear

# Generate optimized autoloader
docker compose -f docker-compose.prod.yml exec -T app composer dump-autoload --optimize --classmap-authoritative

# Health check
if curl -f http://localhost:8080/health; then
    echo "Deployment successful!"
else
    echo "Deployment failed - rolling back..."
    docker compose -f docker-compose.prod.yml down
    # Restore previous version logic here
    exit 1
fi
```

### Environment Variables for Production

```bash
# .env.production
ENVIRONMENT=production
DATABASE_URI=mysql://app_user:secure_password@db:3306/app_production
TRUSTED_HOST=yourdomain.com
REDIS_URI=redis://redis:6379

# Database credentials
MYSQL_ROOT_PASSWORD=very_secure_root_password
MYSQL_DATABASE=app_production
MYSQL_USER=app_user
MYSQL_PASSWORD=secure_password

# Application secrets
JWT_SECRET=your-production-jwt-secret
API_KEY=your-production-api-key
```

### Container Orchestration

#### Docker Swarm

```yaml
# docker-stack.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        max_attempts: 3
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    environment:
      - ENVIRONMENT=production
      - DATABASE_URI=mysql://app_user:password@db:3306/app
    networks:
      - app-network
    secrets:
      - db_password
      - jwt_secret

  db:
    image: mysql:8.0
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MYSQL_DATABASE: app
      MYSQL_USER: app_user
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app-network
    secrets:
      - db_root_password
      - db_password

networks:
  app-network:
    driver: overlay

volumes:
  db_data:

secrets:
  db_root_password:
    external: true
  db_password:
    external: true
  jwt_secret:
    external: true
```

Deploy with:

```bash
# Initialize swarm
docker swarm init

# Create secrets
echo "secure_root_password" | docker secret create db_root_password -
echo "secure_user_password" | docker secret create db_password -
echo "your-jwt-secret" | docker secret create jwt_secret -

# Deploy stack
docker stack deploy -c docker-stack.yml myapp
```

#### Kubernetes

```yaml
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 80
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: DATABASE_URI
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database_uri
        - name: TRUSTED_HOST
          value: "yourdomain.com"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

## Performance Optimization

### Production Optimizations

```dockerfile
# Multi-stage production build with optimizations
FROM php:8.2-fpm-alpine AS production

# Install and configure OPcache
RUN docker-php-ext-install opcache
COPY docker/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

# Install APCu for user cache
RUN pecl install apcu && docker-php-ext-enable apcu

# Configure PHP for production
COPY docker/php.ini /usr/local/etc/php/php.ini

# Install and configure New Relic (optional)
RUN curl -L https://download.newrelic.com/php_agent/archive/9.21.0.311/newrelic-php5-9.21.0.311-linux-musl.tar.gz | tar -C /tmp -zx \
    && export NR_INSTALL_USE_CP_NOT_LN=1 \
    && export NR_INSTALL_SILENT=1 \
    && /tmp/newrelic-php5-*/newrelic-install install \
    && rm -rf /tmp/newrelic-php5-*

# Copy application and install dependencies
COPY . /var/www/html
WORKDIR /var/www/html

RUN composer install --no-dev --optimize-autoloader --classmap-authoritative

# Warm up caches
RUN php bin/console cache:warmup --env=prod

# Set permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html \
    && chmod -R 777 /var/www/html/var
```

### PHP Configuration

```ini
; docker/php.ini
[PHP]
memory_limit = 256M
max_execution_time = 300
upload_max_filesize = 50M
post_max_size = 50M

; Error reporting
display_errors = Off
log_errors = On
error_log = /var/log/php/error.log

; Session
session.gc_maxlifetime = 3600
session.cookie_secure = On
session.cookie_httponly = On
session.cookie_samesite = "Lax"

; Security
expose_php = Off
allow_url_fopen = Off
allow_url_include = Off
```

```ini
; docker/opcache.ini
[opcache]
opcache.enable = 1
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 8
opcache.max_accelerated_files = 4000
opcache.revalidate_freq = 60
opcache.fast_shutdown = 1
opcache.enable_cli = 1
opcache.preload = /var/www/html/var/cache/preload.php
opcache.preload_user = www-data
```

### Health Checks

Create health check endpoints:

```php
<?php
// public/health.php

header('Content-Type: application/json');

$checks = [
    'database' => false,
    'cache' => false,
    'disk_space' => false,
];

try {
    // Database check
    $pdo = new PDO($_ENV['DATABASE_URI']);
    $pdo->query('SELECT 1');
    $checks['database'] = true;
} catch (Exception $e) {
    // Database is down
}

try {
    // Cache check (Redis)
    $redis = new Redis();
    $redis->connect('redis', 6379);
    $redis->ping();
    $checks['cache'] = true;
} catch (Exception $e) {
    // Cache is down
}

// Disk space check
$free_space = disk_free_space('/var/www/html');
$total_space = disk_total_space('/var/www/html');
$usage_percent = (($total_space - $free_space) / $total_space) * 100;
$checks['disk_space'] = $usage_percent < 90;

$overall_status = array_reduce($checks, fn($carry, $check) => $carry && $check, true);

http_response_code($overall_status ? 200 : 503);

echo json_encode([
    'status' => $overall_status ? 'healthy' : 'unhealthy',
    'checks' => $checks,
    'timestamp' => time()
]);
```

## Monitoring and Logging

### Log Configuration

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es_data:/usr/share/elasticsearch/data
    
  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch
    
  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

### Backup Strategy

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"

# Create backup directory
mkdir -p $BACKUP_DIR

# Database backup
docker compose exec -T db mysqldump -u root -ppassword app > $BACKUP_DIR/db_$DATE.sql

# Application files backup (if needed)
tar -czf $BACKUP_DIR/app_$DATE.tar.gz -C /var/www/html .

# Upload to S3 (optional)
aws s3 cp $BACKUP_DIR/db_$DATE.sql s3://your-backup-bucket/
aws s3 cp $BACKUP_DIR/app_$DATE.tar.gz s3://your-backup-bucket/

# Clean up old backups (keep last 7 days)
find $BACKUP_DIR -name "db_*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "app_*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

## Troubleshooting

### Common Docker Issues

#### Container Won't Start

```bash
# Check container logs
docker compose logs app

# Check container status
docker compose ps

# Inspect container
docker compose exec app sh

# Check resource usage
docker stats
```

#### Database Connection Issues

```bash
# Test database connectivity from app container
docker compose exec app php -r "
try {
    \$pdo = new PDO(\$_ENV['DATABASE_URI']);
    echo 'Database connection successful';
} catch (Exception \$e) {
    echo 'Database connection failed: ' . \$e->getMessage();
}
"

# Check database logs
docker compose logs db

# Connect to database directly
docker compose exec db mysql -u root -p
```

#### Permission Issues

```bash
# Fix file permissions
docker compose exec app chown -R www-data:www-data /var/www/html
docker compose exec app chmod -R 755 /var/www/html
docker compose exec app chmod -R 777 /var/www/html/var
```

#### Memory Issues

```bash
# Check memory usage
docker stats

# Increase memory limits in docker-compose.yml
services:
  app:
    deploy:
      resources:
        limits:
          memory: 1G
```

## Best Practices

1. **Use multi-stage builds** to minimize image size
2. **Pin base image versions** for reproducible builds
3. **Run as non-root user** in production
4. **Use health checks** for container orchestration
5. **Implement proper logging** for debugging
6. **Regular security updates** for base images
7. **Monitor resource usage** and set appropriate limits
8. **Backup regularly** and test restore procedures
9. **Use secrets management** for sensitive data
10. **Implement graceful shutdown** handling