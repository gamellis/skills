---
name: fix-migration
description: Fix or modify an existing Rails migration file
argument-hint: "[VERSION or migration name]"
---

# Fix Rails Migration

When modifying an existing migration file, Rails won't re-run it automatically because it tracks applied migrations in `schema_migrations`. Follow these steps:

## Steps

1. **Identify the migration** - Find the migration file by version number or name in `db/migrate/`

2. **Roll back the migration first** - This is a multi-database app, use the namespaced task:
   ```bash
   bin/rails db:migrate:down:primary VERSION=YYYYMMDDHHMMSS
   ```

3. **Make your changes** - Edit the migration file as needed

4. **Re-run migrations** - This regenerates `schema.rb`:
   ```bash
   bin/rails db:migrate
   ```

5. **Prepare the test database**:
   ```bash
   bin/rails db:test:prepare
   ```

6. **Run tests** to verify the changes work correctly:
   ```bash
   bin/rails test
   ```

## Important Notes

- `db:reset` won't work for testing migration changes - it loads from `schema.rb`, not migration files
- Only modify existing migrations in development before they've been deployed
- Create new migrations for changes to already-deployed schemas
- If $ARGUMENTS is provided, use it to find the specific migration to fix
