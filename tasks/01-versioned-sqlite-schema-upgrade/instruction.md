# Instruction: Versioned SQLite Schema Upgrade with Backup, Rollback, and Integrity Verification

## Overview

Implement a robust, versioned schema upgrade system for the SQLite database used by the application. The system must support deterministic migration steps, automatic backup before upgrades, rollback on failure, integrity verification, and a CLI command to inspect schema status.

## Public API and Behavior

### 1. MigrationRegistry

Create a class `MigrationRegistry` that manages migration functions keyed by schema version.

- **`register(version: int, func: Callable[[sqlite3.Connection], None]) -> None`**
  - Stores the migration function for the given version.
  - If a migration for the same version is already registered, raise `ValueError` with message: `"Migration for version {version} already registered"`.

- **`versions() -> List[int]`**
  - Returns a sorted list of all registered version numbers in ascending order.

- **`migrate(conn: sqlite3.Connection, from_version: int, to_version: int) -> None`**
  - Applies all registered migrations with versions greater than `from_version` and less than or equal to `to_version`, in ascending order.
  - If `from_version` is greater than `to_version`, raise `ValueError` with message: `"from_version must be less than or equal to to_version"`.
  - If any migration version in the range is missing, raise `ValueError` with message: `"Missing migration for version {version}"`.
  - If a migration function raises an exception, propagate that exception and do not apply any further migrations.

- **`validate_contiguous() -> bool`**
  - Returns `True` if the registered versions form a contiguous sequence starting from 1 (i.e., versions are 1, 2, 3, ... with no gaps). Otherwise, returns `False`.

### 2. SqliteUpgrader

Enhance the existing `SqliteUpgrader` class with the following methods:

- **`backup_database() -> Path`**
  - Creates a backup of the current SQLite database file using `sqlite3.Connection.backup`.
  - The backup filename format is `backup-YYYYMMDD-HHMMSS.db`, where the timestamp is the local time at the moment of backup creation.
  - If a file with the same name already exists, append a suffix `-#`, where `#` is the smallest positive integer that makes the filename unique.
  - Returns the `Path` to the backup file.

- **`restore_database(backup_path: Path) -> None`**
  - Replaces the current database file with the content of the backup file and reopens the connection.
  - If `backup_path` does not exist, raise `FileNotFoundError` with message: `"Backup file not found: {backup_path}"`.
  - After restore, the connection must be usable and point to the restored database.

- **`upgrade() -> None`**
  - Applies all pending migrations using the `MigrationRegistry`.
  - Before applying migrations, call `backup_database()` to create a backup.
  - Each migration is executed within a transaction.
  - If any migration raises an exception, roll back the transaction, restore the database from the backup using `restore_database()`, and re-raise the original exception with an additional message: `"Migration failed, database restored to previous state"`.
  - After a successful upgrade, update the schema version to the latest version in the registry.
  - The schema version is stored in a table named `schema_version` with a single row and column `version` (INTEGER). If the table does not exist, create it before applying migrations.
  - If the database is already at the latest version, do nothing and return without creating a backup or applying any migrations.

- **`verify_integrity() -> bool`**
  - Executes `PRAGMA integrity_check` on the current connection.
  - If the result is `'ok'`, return `True`; otherwise, return `False`.

- **`get_schema_version() -> int`**
  - Returns the current schema version as an integer.
  - If the `schema_version` table does not exist or is empty, return `0`.

### 3. CLI Command: `schema-info`

- The CLI parser must recognize the command `schema-info` and set `config.action` to `Actions.schema_info`. The command takes no arguments.
- When `config.action` is `Actions.schema_info`, the main function must create a `SchemaInfo` instance with a `SchemaInfoRepo` and call `execute()`.
- The output must be exactly two lines to stdout:
  - `Schema version: {version}`
  - `Integrity: {ok|corrupt}`
  where `{version}` is the current schema version and `{ok|corrupt}` is `ok` if integrity check passes, otherwise `corrupt`.

### 4. Supporting Classes

- **`SchemaInfo`**
  - Constructor takes a `SchemaInfoRepo`.
  - `execute() -> None` prints the two lines as specified above.

- **`SchemaInfoRepo`**
  - `get_schema_version() -> int`: Returns the current schema version.
  - `verify_integrity() -> bool`: Returns the integrity check result.

## Implementation Constraints

- `MigrationRegistry` must store migrations in a dictionary keyed by version number and sort keys when returning versions.
- `backup_database` must use `sqlite3.Connection.backup` to create a consistent snapshot without blocking other connections.
- The upgrade process must use a single transaction for all migrations, and roll back the entire transaction if any migration fails.

## Error Messages and Precedence

- All error messages must match exactly as specified.
- When multiple errors could occur, the order of checks is as follows:
  - For `migrate`: first check `from_version > to_version`, then check for missing migrations.
  - For `register`: check if version already registered.
  - For `restore_database`: check if backup file exists.

## Lifecycle and Compatibility

- The `schema_version` table must be created if it does not exist before any migration is applied.
- After a successful upgrade, the schema version must be set to the latest version in the registry.
- If a migration fails, the database must be restored to its pre-upgrade state, and the schema version must be reverted to its original value (as part of the restore).
- The `upgrade()` method must not create a backup if the database is already at the latest version.
- All public methods must be deterministic and transactional where specified.

## Acceptance Criteria

- All public methods behave as described, with exact error messages and return values.
- The CLI command `schema-info` produces the correct output.
- The implementation must satisfy the implementation constraints.
- The system must handle edge cases such as missing migrations, duplicate registrations, and backup file name collisions.
