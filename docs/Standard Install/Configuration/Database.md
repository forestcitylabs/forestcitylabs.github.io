# Database Configuration

The FCL Framework uses Doctrine DBAL for database connectivity and Doctrine Migrations for schema management. This guide covers database setup, configuration, and management.

## Database Connection

### Connection Configuration

Configure your database connection using the `DATABASE_URI` environment variable:

```env
# MySQL
DATABASE_URI="mysql://user:password@localhost:3306/database_name"

# PostgreSQL
DATABASE_URI="postgresql://user:password@localhost:5432/database_name"

# SQLite
DATABASE_URI="sqlite:///path/to/database.db"
```

### Service Configuration

The database connection is configured in `config/services.php`:

```php
return [
    'doctrine.connection' => DI\factory(function (ContainerInterface $c) {
        return \Doctrine\DBAL\DriverManager::getConnection([
            'url' => $_ENV['DATABASE_URI']
        ]);
    }),
];
```

### Advanced Connection Configuration

For more complex connection requirements:

```php
'doctrine.connection' => DI\factory(function (ContainerInterface $c) {
    $config = [
        'url' => $_ENV['DATABASE_URI'],
        'charset' => 'utf8mb4',
        'driverOptions' => [
            \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
            \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
        ],
    ];
    
    // Add SSL configuration for production
    if ($_ENV['ENVIRONMENT'] === 'production') {
        $config['driverOptions'][\PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT] = false;
        $config['driverOptions'][\PDO::MYSQL_ATTR_SSL_CA] = $_ENV['DB_SSL_CA'] ?? null;
    }
    
    return \Doctrine\DBAL\DriverManager::getConnection($config);
}),
```

## Entity Configuration

### Entity Classes

Entities are stored in `src/Entity/` and use Doctrine attributes:

```php
<?php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'users')]
class User
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;
    
    #[ORM\Column(type: 'string', length: 255)]
    private string $name;
    
    #[ORM\Column(type: 'string', length: 255, unique: true)]
    private string $email;
    
    #[ORM\Column(type: 'datetime_immutable')]
    private \DateTimeImmutable $createdAt;
    
    #[ORM\Column(type: 'datetime_immutable', nullable: true)]
    private ?\DateTimeImmutable $updatedAt = null;
    
    public function __construct(string $name, string $email)
    {
        $this->name = $name;
        $this->email = $email;
        $this->createdAt = new \DateTimeImmutable();
    }
    
    // Getters and setters
    public function getId(): int
    {
        return $this->id;
    }
    
    public function getName(): string
    {
        return $this->name;
    }
    
    public function setName(string $name): void
    {
        $this->name = $name;
        $this->updatedAt = new \DateTimeImmutable();
    }
    
    public function getEmail(): string
    {
        return $this->email;
    }
    
    public function setEmail(string $email): void
    {
        $this->email = $email;
        $this->updatedAt = new \DateTimeImmutable();
    }
    
    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }
    
    public function getUpdatedAt(): ?\DateTimeImmutable
    {
        return $this->updatedAt;
    }
}
```

### Entity Relationships

Configure relationships between entities:

```php
#[ORM\Entity]
#[ORM\Table(name: 'posts')]
class Post
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    private int $id;
    
    #[ORM\Column(type: 'string', length: 255)]
    private string $title;
    
    #[ORM\Column(type: 'text')]
    private string $content;
    
    #[ORM\ManyToOne(targetEntity: User::class)]
    #[ORM\JoinColumn(nullable: false)]
    private User $author;
    
    #[ORM\OneToMany(mappedBy: 'post', targetEntity: Comment::class, cascade: ['persist', 'remove'])]
    private Collection $comments;
    
    #[ORM\ManyToMany(targetEntity: Tag::class, inversedBy: 'posts')]
    #[ORM\JoinTable(name: 'post_tags')]
    private Collection $tags;
    
    public function __construct(string $title, string $content, User $author)
    {
        $this->title = $title;
        $this->content = $content;
        $this->author = $author;
        $this->comments = new ArrayCollection();
        $this->tags = new ArrayCollection();
    }
    
    // ... getters and setters
}
```

### Entity Manager Configuration

Configure the Doctrine ORM Entity Manager:

```php
'doctrine.entity_manager' => DI\factory(function (ContainerInterface $c) {
    $config = \Doctrine\ORM\ORMSetup::createAttributeMetadataConfiguration(
        paths: [__DIR__ . '/../src/Entity'],
        isDevMode: $_ENV['ENVIRONMENT'] === 'development',
        cache: $c->get('cache')
    );
    
    return new \Doctrine\ORM\EntityManager(
        $c->get('doctrine.connection'),
        $config
    );
}),
```

## Database Migrations

### Migration Setup

The framework includes Doctrine Migrations for schema management. Configuration is typically handled automatically, but you can customize it:

```php
// config/migrations.php
return [
    'table_storage' => [
        'table_name' => 'doctrine_migration_versions',
        'version_column_name' => 'version',
        'version_column_length' => 191,
        'executed_at_column_name' => 'executed_at',
    ],
    'migrations_paths' => [
        'Application\\Migrations' => __DIR__ . '/../src/Migrations'
    ],
    'all_or_nothing' => true,
    'transactional' => true,
    'check_database_platform' => true,
];
```

### Creating Migrations

Generate new migrations:

```bash
# Create a new migration
./vendor/bin/console doctrine:migrations:generate

# Create migration with custom name
./vendor/bin/console doctrine:migrations:generate --name=AddUserTable
```

Example migration:

```php
<?php

declare(strict_types=1);

namespace Application\Migrations;

use Doctrine\DBAL\Schema\Schema;
use Doctrine\Migrations\AbstractMigration;

final class Version20240101000000 extends AbstractMigration
{
    public function getDescription(): string
    {
        return 'Create users table';
    }

    public function up(Schema $schema): void
    {
        $this->addSql('
            CREATE TABLE users (
                id INT AUTO_INCREMENT NOT NULL,
                name VARCHAR(255) NOT NULL,
                email VARCHAR(255) NOT NULL UNIQUE,
                created_at DATETIME NOT NULL COMMENT "(DC2Type:datetime_immutable)",
                updated_at DATETIME DEFAULT NULL COMMENT "(DC2Type:datetime_immutable)",
                PRIMARY KEY(id)
            ) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci ENGINE = InnoDB
        ');
    }

    public function down(Schema $schema): void
    {
        $this->addSql('DROP TABLE users');
    }
}
```

### Running Migrations

Execute migrations:

```bash
# Run all pending migrations
./vendor/bin/console doctrine:migrations:migrate

# Migrate to specific version
./vendor/bin/console doctrine:migrations:migrate Version20240101000000

# Rollback one migration
./vendor/bin/console doctrine:migrations:migrate prev

# Check migration status
./vendor/bin/console doctrine:migrations:status

# Show available migrations
./vendor/bin/console doctrine:migrations:list
```

### Migration Best Practices

1. **Always test migrations** on a copy of production data
2. **Make migrations reversible** when possible
3. **Backup before migrations** in production
4. **Use transactions** for complex migrations
5. **Avoid ORM in migrations** - use raw SQL for reliability

```php
public function up(Schema $schema): void
{
    // Good: Use raw SQL
    $this->addSql('UPDATE users SET status = "active" WHERE status IS NULL');
    
    // Avoid: Using ORM in migrations
    // $users = $this->entityManager->getRepository(User::class)->findAll();
}
```

## Repository Configuration

### Repository Classes

Create repository classes for data access:

```php
<?php

namespace Application\Repository;

use Application\Entity\User;
use Doctrine\DBAL\Connection;
use Psr\Cache\CacheItemPoolInterface;

class UserRepository
{
    public function __construct(
        private Connection $connection,
        private CacheItemPoolInterface $cache
    ) {}
    
    public function findById(int $id): ?User
    {
        $cache_key = "user_{$id}";
        $item = $this->cache->getItem($cache_key);
        
        if ($item->isHit()) {
            return $item->get();
        }
        
        $sql = 'SELECT * FROM users WHERE id = ?';
        $row = $this->connection->fetchAssociative($sql, [$id]);
        
        if (!$row) {
            return null;
        }
        
        $user = new User($row['name'], $row['email']);
        // Set private properties via reflection or use factory method
        
        $item->set($user);
        $item->expiresAfter(3600); // 1 hour
        $this->cache->save($item);
        
        return $user;
    }
    
    public function findByEmail(string $email): ?User
    {
        $sql = 'SELECT * FROM users WHERE email = ?';
        $row = $this->connection->fetchAssociative($sql, [$email]);
        
        if (!$row) {
            return null;
        }
        
        return new User($row['name'], $row['email']);
    }
    
    public function save(User $user): void
    {
        if ($user->getId()) {
            $this->update($user);
        } else {
            $this->insert($user);
        }
    }
    
    private function insert(User $user): void
    {
        $sql = 'INSERT INTO users (name, email, created_at) VALUES (?, ?, ?)';
        $this->connection->executeStatement($sql, [
            $user->getName(),
            $user->getEmail(),
            $user->getCreatedAt()->format('Y-m-d H:i:s')
        ]);
    }
    
    private function update(User $user): void
    {
        $sql = 'UPDATE users SET name = ?, email = ?, updated_at = ? WHERE id = ?';
        $this->connection->executeStatement($sql, [
            $user->getName(),
            $user->getEmail(),
            $user->getUpdatedAt()?->format('Y-m-d H:i:s'),
            $user->getId()
        ]);
        
        // Invalidate cache
        $this->cache->deleteItem("user_{$user->getId()}");
    }
}
```

### Repository Service Configuration

Configure repositories in the service container:

```php
return [
    UserRepository::class => DI\factory(function (ContainerInterface $c) {
        return new UserRepository(
            $c->get('doctrine.connection'),
            $c->get('cache')
        );
    }),
    
    PostRepository::class => DI\autowire(),
    CommentRepository::class => DI\autowire(),
];
```

## Database Seeding

### Seeder Classes

Create seeders for development data:

```php
<?php

namespace Application\Seeder;

use Application\Entity\User;
use Doctrine\DBAL\Connection;

class UserSeeder
{
    public function __construct(private Connection $connection) {}
    
    public function seed(): void
    {
        $users = [
            ['name' => 'John Doe', 'email' => 'john@example.com'],
            ['name' => 'Jane Smith', 'email' => 'jane@example.com'],
            ['name' => 'Bob Johnson', 'email' => 'bob@example.com'],
        ];
        
        foreach ($users as $userData) {
            $this->connection->insert('users', [
                'name' => $userData['name'],
                'email' => $userData['email'],
                'created_at' => (new \DateTimeImmutable())->format('Y-m-d H:i:s')
            ]);
        }
    }
}
```

### Seeder Command

Create a console command to run seeders:

```php
<?php

namespace Application\Command;

use Application\Seeder\UserSeeder;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class SeedDatabaseCommand extends Command
{
    protected static $defaultName = 'db:seed';
    
    public function __construct(private UserSeeder $userSeeder)
    {
        parent::__construct();
    }
    
    protected function configure(): void
    {
        $this->setDescription('Seed the database with test data');
    }
    
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $output->writeln('Seeding database...');
        
        $this->userSeeder->seed();
        
        $output->writeln('Database seeded successfully!');
        
        return Command::SUCCESS;
    }
}
```

## Caching Configuration

### Database-Backed Cache

Use the database as a cache backend:

```php
'cache' => DI\factory(function (ContainerInterface $c) {
    return new \ForestCityLabs\Framework\Cache\Pool\DbalCachePool(
        $c->get('doctrine.connection'),
        'cache_items'
    );
}),
```

Create the cache table:

```bash
# Create cache table
./vendor/bin/console cache:table:create
```

### Query Result Caching

Cache expensive database queries:

```php
class UserRepository
{
    public function findActive(): array
    {
        $cache_key = 'active_users';
        $item = $this->cache->getItem($cache_key);
        
        if ($item->isHit()) {
            return $item->get();
        }
        
        $sql = 'SELECT * FROM users WHERE status = "active" ORDER BY created_at DESC';
        $results = $this->connection->fetchAllAssociative($sql);
        
        $item->set($results);
        $item->expiresAfter(900); // 15 minutes
        $this->cache->save($item);
        
        return $results;
    }
}
```

## Connection Management

### Connection Pooling

Configure connection pooling for high-traffic applications:

```php
'doctrine.connection' => DI\factory(function (ContainerInterface $c) {
    $config = [
        'url' => $_ENV['DATABASE_URI'],
        'persistent' => true,
        'pooled' => true,
        'max_connections' => 20,
        'max_idle_time' => 300,
    ];
    
    return \Doctrine\DBAL\DriverManager::getConnection($config);
}),
```

### Read/Write Splitting

Configure separate read and write connections:

```php
return [
    'doctrine.write_connection' => DI\factory(function () {
        return \Doctrine\DBAL\DriverManager::getConnection([
            'url' => $_ENV['DATABASE_WRITE_URI']
        ]);
    }),
    
    'doctrine.read_connection' => DI\factory(function () {
        return \Doctrine\DBAL\DriverManager::getConnection([
            'url' => $_ENV['DATABASE_READ_URI']
        ]);
    }),
];
```

## Performance Optimization

### Connection Optimization

Optimize database connections:

```php
'doctrine.connection' => DI\factory(function (ContainerInterface $c) {
    $config = [
        'url' => $_ENV['DATABASE_URI'],
        'charset' => 'utf8mb4',
        'defaultTableOptions' => [
            'charset' => 'utf8mb4',
            'collate' => 'utf8mb4_unicode_ci',
            'engine' => 'InnoDB',
        ],
        'driverOptions' => [
            \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
            \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
            \PDO::ATTR_EMULATE_PREPARES => false,
            \PDO::MYSQL_ATTR_INIT_COMMAND => 'SET sql_mode="STRICT_TRANS_TABLES"',
        ],
    ];
    
    return \Doctrine\DBAL\DriverManager::getConnection($config);
}),
```

### Query Optimization

Use database-specific optimizations:

```php
// MySQL optimizations
'doctrine.connection' => DI\factory(function (ContainerInterface $c) {
    $connection = \Doctrine\DBAL\DriverManager::getConnection([
        'url' => $_ENV['DATABASE_URI']
    ]);
    
    // Set MySQL-specific optimizations
    $connection->executeStatement('SET SESSION sql_mode = "STRICT_TRANS_TABLES"');
    $connection->executeStatement('SET SESSION optimizer_search_depth = 62');
    
    return $connection;
}),
```

## Troubleshooting

### Connection Testing

Test database connectivity:

```bash
# Test basic connection
./vendor/bin/console doctrine:dbal:run-sql "SELECT 1"

# Test with specific query
./vendor/bin/console doctrine:dbal:run-sql "SELECT COUNT(*) FROM users"

# Show connection info
./vendor/bin/console doctrine:dbal:run-sql "SELECT CONNECTION_ID(), USER(), DATABASE()"
```

### Common Issues

#### Connection Refused
```bash
# Check database server status
systemctl status mysql  # or postgresql

# Check connection parameters
echo $DATABASE_URI

# Test connection manually
mysql -h hostname -u username -p database_name
```

#### Migration Failures
```bash
# Check migration status
./vendor/bin/console doctrine:migrations:status

# Mark migration as executed (if manually fixed)
./vendor/bin/console doctrine:migrations:version Version20240101000000 --add

# Rollback problematic migration
./vendor/bin/console doctrine:migrations:migrate prev
```

#### Permission Issues
```sql
-- Grant necessary permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON database_name.* TO 'username'@'%';
GRANT CREATE, ALTER, DROP, INDEX ON database_name.* TO 'username'@'%';
FLUSH PRIVILEGES;
```

## Best Practices

1. **Use transactions** for multi-step operations
2. **Index frequently queried columns**
3. **Avoid N+1 queries** - use JOINs or eager loading
4. **Cache expensive queries** appropriately
5. **Validate input** before database operations
6. **Use prepared statements** for security
7. **Monitor query performance** regularly
8. **Backup regularly** and test restore procedures