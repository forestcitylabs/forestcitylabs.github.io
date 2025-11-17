# CLI Commands

The FCL Framework provides several CLI commands to help with development, maintenance, and code generation tasks. Commands are built using the Symfony Console component and can be integrated into your application.

## Available Commands

The framework includes the following built-in commands:

### Cache Commands
- `cache:clear` - Clear application cache
- `cache:table:create` - Create database table for cache storage

### Session Commands  

### GraphQL Commands
- `graphql:dump-schema` - Export GraphQL schema to file
- `graphql:generate-from-schema` - Generate PHP classes from GraphQL schema
- `graphql:schema-diff` - Compare two GraphQL schemas
- `graphql:validate-schema` - Validate GraphQL schema

### Entity Commands
- `generate:entity` - Generate Doctrine entity class

### Fixture Commands
- `fixtures:load` - Load test fixtures

## Using Commands

### Running Commands

Commands are run using the framework's console application:

```bash
# Using the console binary
./vendor/bin/console <command>
```

### Getting Help

Get help for any command:

```bash
./vendor/bin/console help cache:clear
./vendor/bin/console cache:clear --help
```

## Command Details

### Cache Management

#### Clear Cache

Remove all cached data:

```bash
# Clear all cache
./vendor/bin/console cache:clear

# Clear specific cache pool (if supported)  
./vendor/bin/console cache:clear --pool=routing
```

The command clears:
- Route metadata cache
- GraphQL schema cache
- Application-specific cached data

#### Create Cache Table

Create the database table for DBAL cache storage:

```bash
./vendor/bin/console cache:table:create

# Specify custom table name
./vendor/bin/console cache:table:create --table-name=app_cache

# Preview SQL without executing
./vendor/bin/console cache:table:create --dry-run
```

### Session Management



### GraphQL Commands

#### Dump Schema

Export your GraphQL schema to a file:

```bash
# Dump to stdout
./vendor/bin/console graphql:dump-schema

# Save to file
./vendor/bin/console graphql:dump-schema > schema.graphql

# Specify output file
./vendor/bin/console graphql:dump-schema --output=schema.graphql

# Format output (SDL, JSON)
./vendor/bin/console graphql:dump-schema --format=json
```

#### Generate from Schema

Generate PHP classes from a GraphQL schema file:

```bash
# Generate types in specified directory
./vendor/bin/console graphql:generate-from-schema schema.graphql src/GraphQL

# Specify namespace
./vendor/bin/console graphql:generate-from-schema schema.graphql src/GraphQL --namespace="App\\GraphQL"

# Overwrite existing files
./vendor/bin/console graphql:generate-from-schema schema.graphql src/GraphQL --force
```

Generated files include:
- Object types
- Input types
- Enum types
- Interface types
- Query and Mutation root types

#### Schema Validation

Validate your GraphQL schema for errors:

```bash
# Validate current schema
./vendor/bin/console graphql:validate-schema

# Validate specific schema file
./vendor/bin/console graphql:validate-schema --schema=schema.graphql

# Verbose output
./vendor/bin/console graphql:validate-schema --verbose
```

Common validation errors:
- Invalid type definitions
- Circular references
- Missing required fields
- Invalid field types

#### Schema Comparison

Compare two schemas to identify differences:

```bash
# Compare current schema with file
./vendor/bin/console graphql:schema-diff schema-old.graphql

# Compare two files  
./vendor/bin/console graphql:schema-diff schema-old.graphql schema-new.graphql

# Output format (text, json)
./vendor/bin/console graphql:schema-diff old.graphql new.graphql --format=json

# Only show breaking changes
./vendor/bin/console graphql:schema-diff old.graphql new.graphql --breaking-only
```

### Entity Generation

Generate Doctrine entity classes:

```bash
# Generate basic entity
./vendor/bin/console generate:entity User

# Specify properties
./vendor/bin/console generate:entity User --properties=name:string,email:string

# Include relationships
./vendor/bin/console generate:entity Post --properties=title:string --relations=author:User

# Generate in specific namespace
./vendor/bin/console generate:entity User --namespace="App\\Entity"

# Specify output directory
./vendor/bin/console generate:entity User --output-dir=src/Entity
```

Generated entities include:
- Basic class structure
- Doctrine annotations/attributes
- Property getters and setters
- Constructor
- Relationship mappings

### Fixture Loading

Load test fixtures into your database:

```bash
# Load all fixtures
./vendor/bin/console fixtures:load

# Load specific fixtures
./vendor/bin/console fixtures:load --fixtures=UserFixtures,PostFixtures

# Skip confirmation prompt
./vendor/bin/console fixtures:load --no-interaction

# Purge database before loading
./vendor/bin/console fixtures:load --purge-with-truncate
```

## Creating Custom Commands

### Basic Command Structure

Create custom commands by extending Symfony's Command class:

```php
<?php

namespace Application\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;

class MyCustomCommand extends Command
{
    protected static $defaultName = 'app:my-command';
    protected static $defaultDescription = 'Description of my custom command';
    
    protected function configure(): void
    {
        $this->addArgument('name', InputArgument::REQUIRED, 'The name argument');
        $this->addOption('option', 'o', InputOption::VALUE_OPTIONAL, 'An optional option');
    }
    
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $name = $input->getArgument('name');
        $option = $input->getOption('option');
        
        $output->writeln("Hello {$name}!");
        
        return Command::SUCCESS;
    }
}
```

### Command with Dependencies

Use dependency injection for services:

```php
<?php

namespace Application\Command;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Application\Repository\UserRepository;

class UserStatsCommand extends Command
{
    protected static $defaultName = 'app:user-stats';
    
    public function __construct(
        private UserRepository $userRepository
    ) {
        parent::__construct();
    }
    
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $userCount = $this->userRepository->count();
        
        $output->writeln("Total users: {$userCount}");
        
        return Command::SUCCESS;
    }
}
```

### Registering Commands

Register commands in your service container:

```php
// In your service configuration
use Application\Command\MyCustomCommand;

$container->set(MyCustomCommand::class)
          ->tag('console.command');
```

Or register manually in your console application:

```php
use Symfony\Component\Console\Application;

$console = new Application();
$console->add($container->get(MyCustomCommand::class));
```

## Command Development Best Practices

### Input Validation

Always validate command input:

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    $email = $input->getArgument('email');
    
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $output->writeln('<error>Invalid email address</error>');
        return Command::FAILURE;
    }
    
    // Continue with command logic...
    return Command::SUCCESS;
}
```

### Progress Indicators

Show progress for long-running operations:

```php
use Symfony\Component\Console\Helper\ProgressBar;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $users = $this->userRepository->findAll();
    
    $progressBar = new ProgressBar($output, count($users));
    $progressBar->start();
    
    foreach ($users as $user) {
        // Process user...
        $progressBar->advance();
    }
    
    $progressBar->finish();
    $output->writeln(''); // New line after progress bar
    
    return Command::SUCCESS;
}
```

### Confirmation Prompts

Ask for confirmation for destructive operations:

```php
use Symfony\Component\Console\Question\ConfirmationQuestion;

protected function execute(InputInterface $input, OutputInterface $output): int
{
    $helper = $this->getHelper('question');
    $question = new ConfirmationQuestion(
        'This will delete all users. Continue? [y/N] ', 
        false
    );
    
    if (!$helper->ask($input, $output, $question)) {
        $output->writeln('Aborted');
        return Command::SUCCESS;
    }
    
    // Perform destructive operation...
    return Command::SUCCESS;
}
```

### Error Handling

Handle errors gracefully:

```php
protected function execute(InputInterface $input, OutputInterface $output): int
{
    try {
        // Command logic here...
        $output->writeln('<info>Operation completed successfully</info>');
        return Command::SUCCESS;
    } catch (\Exception $e) {
        $output->writeln('<error>Error: ' . $e->getMessage() . '</error>');
        
        if ($output->isVerbose()) {
            $output->writeln('<comment>Stack trace:</comment>');
            $output->writeln($e->getTraceAsString());
        }
        
        return Command::FAILURE;
    }
}
```

### Logging Command Execution

Log command execution for audit trails:

```php
use Psr\Log\LoggerInterface;

class MyCommand extends Command
{
    public function __construct(
        private LoggerInterface $logger
    ) {
        parent::__construct();
    }
    
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $this->logger->info('Command started', [
            'command' => $this->getName(),
            'arguments' => $input->getArguments(),
            'options' => $input->getOptions()
        ]);
        
        // Command logic...
        
        $this->logger->info('Command completed successfully');
        return Command::SUCCESS;
    }
}
```

## Testing Commands

Test commands using Symfony's CommandTester:

```php
use PHPUnit\Framework\TestCase;
use Symfony\Component\Console\Application;
use Symfony\Component\Console\Tester\CommandTester;

class MyCommandTest extends TestCase
{
    public function testExecute(): void
    {
        $application = new Application();
        $application->add(new MyCustomCommand());
        
        $command = $application->find('app:my-command');
        $commandTester = new CommandTester($command);
        
        $commandTester->execute([
            'name' => 'Test Name',
            '--option' => 'test-value'
        ]);
        
        $output = $commandTester->getDisplay();
        $this->assertStringContainsString('Hello Test Name!', $output);
        $this->assertEquals(Command::SUCCESS, $commandTester->getStatusCode());
    }
}
```

## Scheduling Commands

For production environments, schedule commands using cron:

```bash
# Run cache clear daily at 2 AM
0 2 * * * cd /path/to/app && ./vendor/bin/console cache:clear

# Run session cleanup hourly  
# 0 * * * * cd /path/to/app && ./vendor/bin/console session:clear --older-than=24h  # Not needed with default PHP sessions

# Validate schema before deployments
0 0 * * * cd /path/to/app && ./vendor/bin/console graphql:validate-schema
```