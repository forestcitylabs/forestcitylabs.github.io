# Database (DBAL)

`config/packages/dbal.php` configures the Doctrine DBAL connection and adds the `dbal:run-sql` console command.

## What it does

- Parses the `DATABASE_URI` environment variable using Doctrine's `DsnParser` and creates a `Connection`.
- Registers three Ramsey UUID custom DBAL types (`uuid`, `uuid_binary`, `uuid_binary_ordered_time`) on the connection's platform.
- Binds `ConnectionProvider` to `SingleConnectionProvider` for use by console commands.
- Adds the `dbal:run-sql` command to the console.

## Configuration parameters

| Key | Environment variable | Default | Description |
|-----|----------------------|---------|-------------|
| `dbal.database_uri` | `DATABASE_URI` | — | Doctrine DBAL DSN. Must be set in your `.env` file. |

### DSN format

The DSN uses the `pdo-<driver>` scheme as expected by Doctrine's `DsnParser`:

```
pdo-mysql://user:password@host:3306/dbname
pdo-pgsql://user:password@host:5432/dbname
pdo-sqlite:///:memory:
```

### `.env` example

```
DATABASE_URI=pdo-mysql://root:root@db:3306/app
```

## UUID types

The following custom Doctrine types are registered automatically:

| Type name | Class | Notes |
|-----------|-------|-------|
| `uuid` | `UuidType` | String UUID stored as `CHAR(36)`. |
| `uuid_binary` | `UuidBinaryType` | UUID stored as 16-byte binary. |
| `uuid_binary_ordered_time` | `UuidBinaryOrderedTimeType` | Time-ordered binary UUID (better index performance). |

## Console commands registered

| Command | Description |
|---------|-------------|
| `dbal:run-sql` | Execute arbitrary SQL against the configured database connection. |
