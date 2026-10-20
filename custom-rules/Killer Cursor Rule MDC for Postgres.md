

# SQLAlchemy Cursor Rules for PostgreSQL

## Safe Migration Practices

- Always use **Alembic** for schema migrations; never run raw database deletion scripts directly.[^1_1][^1_2]
- Use the **expand -> migrate -> contract** migration pattern to ensure zero downtime and avoid breaking running code.[^1_2]
- **Never use models inside migrations**—define all tables and columns in migration scripts to avoid dependency issues.[^1_1][^1_2]
- For data manipulation, prefer raw SQL or manually defined tables in migrations—keep migrations atomic and idempotent.[^1_1]
- Always test migrations locally using the **stairway test** (successive upgrade/downgrade) before deploying to production.[^1_1]


## Naming and Organization

```
- Use **Unix timestamp** filename format for migrations: `%Y%m%d_%H%M%S_<rev>_<slug>.py`.[^1_1]
```

- Add clear comments to migrations with `-m 'add user table'` for traceability.[^1_1]
- Use **naming conventions** in SQLAlchemy for predictable constraint names:

```python
convention = {
    'ix': 'ix__%(table_name)s__%(all_column_names)s',
    'uq': 'uq__%(table_name)s__%(all_column_names)s',
    'fk': 'fk__%(table_name)s__%(all_column_names)s__%(referred_table_name)s',
    'pk': 'pk__%(table_name)s',
}
```

- Comment all tables and columns in SQLAlchemy for clarity.[^1_1]


## Safety and Security

- **Never add or remove columns indexed with `CONCURRENTLY`** in a single migration step; handle index creation separately and use `IF NOT EXISTS` for safety.[^1_2]
- **Always set a lock timeout** (`lock_timeout=2000`) in Alembic to fail fast instead of hanging.[^1_2]
- Use **linter tools** (like Squawk) to catch risky operations before merging migrations.[^1_2]
- **Explicitly confirm** all migrations in PRs with a `db-migration-ok` label or manual checklist.[^1_2]
- **Never run destructive scripts**—always use `DROP ... IF EXISTS` and double-check migrations.[^1_2]


## ORM and Model Best Practices

- Use **mixins** for common fields (e.g., timestamps, UUIDs) to avoid duplication.[^1_1]
- For columns being dropped, mark them as `deferred` in the ORM model before removal.[^1_2]
- **Never use string concatenation** in queries—always use parameterized queries for SQL injection safety.[^1_3]


## Example Safe Migration

```python
from alembic import op
from sqlalchemy import Column, Integer, String, Table

def upgrade():
    # Define table manually
    users = Table(
        'users',
        op.get_bind().metadata,
        Column('id', Integer, primary_key=True),
        Column('name', String),
    )
    op.add_column('users', Column('email', String))

def downgrade():
    op.drop_column('users', 'email')
```


***
