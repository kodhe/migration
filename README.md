# Kodhe Migration

A database schema versioning package — a refactored version of the *Migrations* library (CI Bonfire) under the `Kodhe\Framework\Migration` namespace. It provides two classes:

- `Migration` — a modern runner based on the `schema_version` table, supporting **core** and **per-module** (`app_<module>_`) migrations; it can run automatically on page load (`autoLatest()`) or manually per version.
- `Legacy` — a classic CI3-style runner (`application/db/migrations/` + `$config['migration_*']` config) for projects that already have existing migration files.

## Installation

```bash
composer require kodhe/migration
```

Requirements: PHP >= 8.1, an active database connection (uses the `kodhe/database` query builder inside CI's abstract `Migration` class), and the CI constants (`BASEPATH`, `APPPATH`).

## Quick Start

```php
<?php

declare(strict_types=1);

use Kodhe\Framework\Migration\Migration;

// From CI config: $config['migrations_path'] = APPPATH.'Migration';
$migration = new Migration(['migrations_path' => APPPATH . 'Migration']);

// Run all core migrations up to the latest version
$migration->install();

// Check the result
echo $migration->getVersion();          // currently active version
echo $migration->getVersion('', true);  // latest available version
```

## Directory Structure

```
src/
├── Migration.php   # Main runner (schema_version table, module-aware)
└── Legacy.php      # CI3-compatible runner (db/migrations + migration_enabled)
```

## Usage

### 1. Creating migration files

Files go in `application/Migration/` with sequential numbering:

```
001_install_initial_schema.php
002_add_user_avatar.php
```

Each file contains a class extending CI's abstract `Migration` and must define `up()`/`down()`:

```php
<?php // 002_add_user_avatar.php

class Migration_add_user_avatar extends CI_Migration
{
    public function up()
    {
        $this->dbforge->add_column('users', [
            'avatar' => ['type' => 'varchar', 'constraint' => 255, 'null' => true],
        ]);
    }

    public function down()
    {
        $this->dbforge->drop_column('users', 'avatar');
    }
}
```

### 2. Per-module migrations (app migrations)

File names start with `app_<module>_<number>_<description>` → the runner tracks a separate version per module:

```php
$migration->install('blog');            // only the blog module
$migration->version(3, 'blog');         // to blog module version 3
$versions = $migration->getModuleVersions(); // ['' => 5, 'blog' => 3, ...]
```

### 3. Auto-migrate on application load

```php
// application/config/*.php
$config['migrate']['auto_core'] = TRUE;
$config['migrate']['auto_app']  = TRUE;

// In a hook/bootstrap:
$migration->autoLatest();
```

### 4. Plain SQL without a migration class

```php
$migration->doSqlMigration("ALTER TABLE users ADD COLUMN token VARCHAR(64) NULL");
```

### 5. Handling errors

```php
if ($migration->version(4) === false) {
    echo $migration->getErrorMessage();  // combined message
    print_r($migration->getErrors());    // raw array
}
```

### 6. Legacy mode (fully CI3 compatible)

```php
use Kodhe\Framework\Migration\Legacy;

// Requires standard CI3 config:
// $config['migration_enabled'], ['migration_type'], ['migration_path'],
// ['migration_table'], ['migration_auto_latest'], ['migration_version']
$m = new Legacy();
$m->latest();                 // upgrade to the latest version
$m->current();                // active version
$m->version(2);               // move to version 2 (up/down via up()/down())
echo $m->error_string();
```

## API Reference (`Migration`)

| Method | Description |
|---|---|
| `__construct(array $params)` | Optional: `migrations_path`; creates the `schema_version` table if missing |
| `autoLatest()` | Runs core/app auto-migration according to the `migrate` config |
| `install($type = '')` | Upgrades to the latest version for a given type/module (`''` = core) |
| `version($version, $type = '')` | Moves to a specific version (forward/backward) |
| `doSqlMigration($sql)` | Executes a raw SQL statement directly & records its version |
| `getVersion($type, $getLatest = false)` | Active version / latest available version |
| `getAvailableVersions($type)` | Lists all version numbers in the folder |
| `getModuleVersions()` | Version map per module |
| `getErrors()` / `getErrorMessage()` | Retrieve errors |
| `setVerbose($state)` | Prints progress to the output |

## CodeIgniter 3 Compatibility

Behavior follows CI_Bonfire Migrations; `Legacy` mimics the original `CI_Migration`. The tracking table is named `schema_version` (not `ci_migrations`) — when migrating from CI3, adjust the table name via the `$migrationsTable` property.

## Notes

- Depends on the CI environment (`get_instance()`, `show_error()`, dbforge) — not a fully standalone package.
- Version numbers must be sequential integers; files without a numeric prefix are ignored.

## Testing

```bash
vendor/bin/phpunit --filter Migration
```
