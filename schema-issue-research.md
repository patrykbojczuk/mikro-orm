# mikro-orm migration schema issue

Source: https://github.com/mikro-orm/mikro-orm/discussions/7027

## Summary

When using PostgreSQL with a custom schema (not `public`) via the `schema` config option, `migrator.up()` creates entity tables in `public` instead of the configured schema. The `mikro_orm_migrations` tracking table correctly goes to the configured schema — but the actual migration SQL does not.

## What B4nan didn't understand

### 1. The use case isn't wildcard/multi-tenant

User's setup:
- `public` schema = production
- Custom schemas (e.g. `preview_feat_123`) = clones of `public` for feature preview environments
- Each app instance uses exactly one schema at all times

B4nan kept pushing toward wildcard entities and `em.fork({ schema })` — a pattern for runtime multi-tenant schema switching within a single app instance. That's a completely different use case.

### 2. PostgreSQL `search_path` behavior

When SQL doesn't explicitly qualify table names with a schema (`CREATE TABLE "user"` vs `CREATE TABLE "test123"."user"`), PostgreSQL uses `search_path` to determine which schema to use.

B4nan said "you don't connect to a schema" — technically true, but `search_path` controls the default schema resolution for all unqualified identifiers. The user's DataGrip analogy was valid: selecting a schema in DataGrip sets `search_path`, making unqualified queries run against that schema.

### 3. The database ↔ schema inconsistency

User pointed out: "when I set a different `database` in the config, migrations don't run on the default `postgres` database. Why would `schema` work differently?"

This is a valid parallel. The ORM respects the `database` config for migrations (connects to the right database), but ignores the `schema` config (doesn't set `search_path`). B4nan dismissed this without explaining why the inconsistency is acceptable.

### 4. Non-breaking fix is possible

B4nan claimed any fix "would be breaking." User correctly pointed out:
- A separate migration-specific option wouldn't break anything
- Simply setting `search_path` before running migration SQL wouldn't break anything
- Default schema is `public`, which matches default `search_path` — existing users are unaffected

The user's own workaround proves the fix is trivial and non-breaking.

### 5. Default `migration:create` output has no schema qualification

B4nan said "you did not have the schema set when you generated your migrations." But the default output of `migration:create` produces unqualified SQL — no schema prefix. This means migrations already implicitly depend on the runtime schema context (`search_path`). Respecting the `schema` config at migration time is the logical completion of this behavior.

## The internal inconsistency in MikroORM

The migration tracking table (`mikro_orm_migrations`) IS placed in the configured schema. But the actual migration SQL runs against `public`. This means MikroORM already partially respects the `schema` config during migrations — just not for the SQL that matters most.

## How it should work: preview & dev environments

### Setup
- Entities: `User`, `Post`
- Database: `myapp`
- Production: `public` schema
- Preview environments: per-branch schemas (e.g. `preview_feat_123`)

### Step 1: Generate migrations (dev)

```
# No schema config (defaults to public)
pnpm migration:create
```

Output: unqualified SQL
```sql
CREATE TABLE "user" (id serial primary key, name varchar(255));
CREATE TABLE "post" (id serial primary key, title varchar(255), author_id int references "user"(id));
```

### Step 2: Run on production

```ts
// config: no schema set (defaults to public)
await orm.migrator.up();
// search_path = public (default)
// → tables created in public ✅
```

### Step 3: Create preview environment

```sql
CREATE SCHEMA preview_feat_123;
```

### Step 4: Run on preview

```ts
// config: { schema: 'preview_feat_123' }
await orm.migrator.up();
```

**Current behavior (broken):**
| Table | Schema | Correct? |
|---|---|---|
| `mikro_orm_migrations` | `preview_feat_123` | ✅ |
| `user` | `public` | ❌ |
| `post` | `public` | ❌ |

**Expected behavior:**
| Table | Schema | Correct? |
|---|---|---|
| `mikro_orm_migrations` | `preview_feat_123` | ✅ |
| `user` | `preview_feat_123` | ✅ |
| `post` | `preview_feat_123` | ✅ |

### Step 5: Preview app runs normally

```ts
// config: { schema: 'preview_feat_123' }
// ORM generates queries like: SELECT * FROM "user"
// PostgreSQL resolves "user" → preview_feat_123.user (via search_path)
```

## Workaround (from the discussion author)

```ts
const targetSchema = 'preview_feat_123';
const orm = await MikroORM.init(config);
const connection = orm.em.getConnection();

await connection.transactional(async (tx) => {
  await connection.execute('SET search_path TO ??', [targetSchema], undefined, tx);
  await orm.migrator.up({ transaction: tx });
});
```

## Suggested fix for MikroORM

When `schema` is set in config, set `search_path` to that schema before executing migration SQL.

Non-breaking because:
- Default `search_path` is `public`, which is already the default schema
- Only affects users who explicitly set a custom `schema` in config
- Migration SQL content stays unchanged (no schema qualification needed)
- Existing behavior for users without custom `schema` is identical
