Hi @B4nan, I wanted to revisit this with a more detailed explanation of the use case, as I think I might not have explained it clearly enough last time.

## Use case: PR preview environments

This has nothing to do with dynamic tenants or wildcard schemas. Here's the setup:

- One PostgreSQL database (e.g. `myapp_dev`)
- Production runs on `public` schema
- Each PR preview environment gets its own schema (e.g. `pr_142`, `pr_187`)
- Each preview is a **separate, independent deployment** — its own app instance, its own config, its own schema
- There is no runtime schema switching — each instance uses exactly one schema for its entire lifetime

The flow looks like this:

### 1. Developer generates migrations locally (no schema config)

```
npx mikro-orm migration:create
```

Produces unqualified SQL — no schema prefix:
```sql
CREATE TABLE "user" (id serial primary key, ...);
CREATE TABLE "post" (id serial primary key, ...);
```

### 2. Production deployment runs migrations

```ts
const orm = await MikroORM.init({
  // no schema set, defaults to public
});
await orm.migrator.up();
// → tables in public ✅
```

### 3. PR preview deployment runs migrations

```ts
const orm = await MikroORM.init({
  schema: 'pr_142',
});
await orm.migrator.up();
```

**What happens now:**
- `mikro_orm_migrations` → `pr_142` schema ✅
- `user`, `post` → `public` schema ❌

The migration tracking table respects the `schema` config, but the actual migration SQL doesn't.

## Why this isn't the wildcard pattern

In the wildcard pattern:
- A **single** app instance handles **multiple** schemas at runtime
- Schemas are switched dynamically via `em.fork({ schema })`
- Migrations shouldn't target any specific schema

In my use case:
- **Separate** app instances, each with its **own** config pointing to a **single** schema
- No runtime schema switching at all
- Migrations should run in the configured schema, just like queries do

## `search_path` — the PostgreSQL mechanism for this

When SQL doesn't qualify table names with a schema, PostgreSQL resolves them via `search_path`. Setting `search_path = pr_142` before running migration SQL would make all unqualified tables land in `pr_142`. This is the same mechanism DataGrip uses when you select a schema context.

My workaround from the previous discussion proves this works:

```ts
await connection.transactional(async (tx) => {
  await connection.execute('SET search_path TO ??', [schema], undefined, tx);
  await orm.migrator.up({ transaction: tx });
});
```

## How other tools handle this

Other migration tools solve this by setting `search_path` on the database connection at migration runtime — migration files stay schema-agnostic, the target schema is determined by config:

| Tool | Config | Mechanism | Proof |
|---|---|---|---|
| **Flyway** | `flyway.defaultSchema` | Executes `SELECT set_config('search_path', ?, false)` before each migration | [source](https://github.com/flyway/flyway/blob/main/flyway-database/flyway-database-postgresql/src/main/java/org/flywaydb/database/postgresql/PostgreSQLConnection.java), [docs](https://documentation.red-gate.com/fd/flyway-default-schema-setting-277578987.html) |
| **Liquibase** | `--default-schema-name` | Sets `search_path` on the connection; all changesets without explicit schema use it | [docs](https://docs.liquibase.com/commands/update/update.html), [issue](https://github.com/liquibase/liquibase/issues/3312) |
| **Rails** | `schema_search_path` in `database.yml` | Executes `SET search_path TO ...` on connection setup | [source](https://github.com/rails/rails/blob/cb91d817a78352fb41e33764d651991d566ae82b/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb#L307-L311) |
| **Django** | `OPTIONS: { options: '-c search_path=...' }` | Forwards to libpq which sets `search_path` at connection start | [source](https://github.com/django/django/blob/1ce6e78dd4beed702f15fa0be798dd17a15d4ba8/django/db/backends/postgresql/base.py#L246-L314), [docs](https://docs.djangoproject.com/en/5.2/ref/databases/) |
| **Sequelize** | `searchPath` + `prependSearchPath: true` | Prepends `SET search_path to ...;` before every query | [source](https://github.com/sequelize/sequelize/blob/main/packages/postgres/src/query-generator.js), [PR](https://github.com/sequelize/sequelize/pull/4534) |

Notably, Flyway and Liquibase both have **dedicated migration-specific schema options** — exactly the pattern I'm suggesting for MikroORM.

## Suggestion: a new migration option

If you find this breaking, instead of changing the existing `schema` behavior, a dedicated option like `migrations.schema` (or similar) could control the `search_path` when running migrations.

- If left `undefined` — current behavior, nothing changes, fully backwards-compatible
- If set — `search_path` is set to that value before executing migration SQL

This wouldn't change how migrations are generated or how queries work. It only affects the schema context when migration SQL is executed. No existing user would be impacted.

Would you be open to reconsidering this with the dedicated option approach?
