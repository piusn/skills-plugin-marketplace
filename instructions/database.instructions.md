---
description: "Database design and data modeling standards covering SQL/NoSQL patterns, migrations, indexing strategies, query optimization, and data integrity."
applyTo: "**/*.sql,**/migrations/**,**/models/**,**/entities/**,**/schema/**,**/*Context*.cs,**/*Repository*.cs"
---

# Database Design & Data Modeling Standards

Databases outlive applications. Design them to last decades, not sprints.

---

## Data Modeling Philosophy

- **Normalize first, denormalize with justification.** Start at 3NF minimum. If you denormalize, document WHY in a comment or ADR — "performance" is not sufficient; include the specific query pattern and measured impact.
- **Every table needs a reason to exist.** If you can't explain the entity it represents in one sentence, reconsider the design.
- **Design for queries, not just writes.** Understand your read patterns before finalizing schema. The access pattern drives the model.

---

## Naming Conventions

### Tables and Columns
- **Tables**: PascalCase singular nouns — `User`, `OrderItem`, `BuildConfiguration`. Use singular because each row represents one entity.
- **Columns**: PascalCase to match the table convention — `CreatedAt`, `FirstName`, `OrderId`. Be consistent with whatever the ORM/framework expects.
- **If the team uses snake_case** (e.g., Python/PostgreSQL teams): `created_at`, `first_name`, `order_id`. Pick one convention and enforce it project-wide. Never mix.
- **Boolean columns**: Prefix with `Is`, `Has`, or `Can` — `IsActive`, `HasChildren`, `CanEdit`.
- **Date columns**: Suffix with `At` or `On` — `CreatedAt`, `DeletedAt`, `ScheduledOn`.

### Indexes
- Name format: `IX_{Table}_{Column1}_{Column2}` for non-unique, `UQ_{Table}_{Column1}` for unique.
- Example: `IX_Order_CustomerId_CreatedAt`, `UQ_User_Email`.

### Foreign Keys
- Name format: `FK_{ChildTable}_{ParentTable}_{Column}`.
- Example: `FK_OrderItem_Order_OrderId`.

### Constraints
- `CK_{Table}_{Description}` for check constraints.
- `DF_{Table}_{Column}` for defaults.

---

## Primary Keys

### Distributed Systems (Multiple Databases/Services)
- Use **UUIDs/GUIDs** (specifically UUIDv7 when available — time-ordered for index friendliness).
- Store as `UNIQUEIDENTIFIER` (SQL Server), `uuid` (PostgreSQL), or `BINARY(16)` (MySQL).
- Generate on the client side to allow pre-association before insert.

### Single-Database Systems
- **Sequential integer IDs** (`IDENTITY` / `SERIAL`) are acceptable and more storage-efficient.
- Combine with a public-facing UUID if IDs are exposed in URLs (never expose sequential IDs in public APIs — they leak information about volume).

### Never Do
- Never use natural keys (email, SSN, phone) as primary keys — they change.
- Never use composite primary keys unless modeling a pure junction table. Even then, consider a surrogate key.

---

## Foreign Keys — Always Enforce

- **Every relationship MUST have a foreign key constraint.** No exceptions.
- Foreign keys are documentation, validation, and protection against orphaned data.
- If performance is a concern, the answer is better indexing, not dropping constraints.
- Define `ON DELETE` behavior explicitly:
  - `CASCADE` — when child records have no meaning without the parent (e.g., `OrderItem` → `Order`).
  - `RESTRICT` / `NO ACTION` — when deletion should be prevented if children exist (default and safest).
  - `SET NULL` — rarely appropriate, only when the relationship is genuinely optional.
- **Never use `ON DELETE CASCADE` on core business entities** (Users, Orders, Accounts). Use soft deletes instead.

---

## Indexing Strategy

### Core Rules
- **Every query in production MUST use an index.** Check execution plans. A table scan on a growing table is a ticking time bomb.
- **Every foreign key column MUST be indexed.** Without it, JOINs and CASCADE deletes perform full table scans.
- **Composite index column order matters.** Put the most selective (highest cardinality) column first, or match the order of your WHERE clause.
- **Covering indexes** reduce I/O — include frequently selected columns with `INCLUDE`.

### What to Index
- Foreign key columns (always).
- Columns in WHERE clauses with high selectivity.
- Columns in ORDER BY / GROUP BY.
- Columns used in JOIN conditions.

### What NOT to Index
- Low-cardinality boolean columns (unless combined in a composite index).
- Columns that are frequently updated (index maintenance cost).
- Tables with fewer than ~1000 rows (full scan is faster).

### Index Maintenance
- Monitor index usage statistics. Drop unused indexes — they slow down writes for zero benefit.
- Rebuild fragmented indexes on a schedule (>30% fragmentation → rebuild, 10–30% → reorganize).
- For SQL Server: use `sys.dm_db_index_usage_stats` to find unused indexes.

---

## Migrations

### Rules
- **Every schema change is a migration.** No manual DDL in any environment, ever.
- **Migrations MUST be reversible.** Every `Up()` has a corresponding `Down()`.
- **Migrations MUST be idempotent.** Running the same migration twice should not fail.
- **Never modify an existing migration** that has been applied to any shared environment. Create a new migration instead.
- **Name migrations descriptively**: `20240115_AddUserEmailVerification`, not `Migration42`.

### Safe Migration Patterns
- **Adding a column**: Add as nullable first, backfill data, then add NOT NULL constraint in a separate migration.
- **Renaming a column**: Add new column → backfill → update code to use new column → drop old column. Never rename in one step.
- **Dropping a column**: Remove all code references first, deploy, then drop the column in a subsequent release.
- **Adding an index**: Use `CREATE INDEX CONCURRENTLY` (PostgreSQL) or online index operations to avoid locking.

### Dangerous Operations — Require Review
- Dropping tables or columns.
- Changing column types (potential data loss).
- Adding NOT NULL constraints to existing columns.
- Any migration that locks tables for extended periods.

---

## Query Optimization

### Always Do
- **Use parameterized queries.** Always. No exceptions. String concatenation in SQL is an injection vulnerability, not a shortcut.
- **Avoid `SELECT *`.** Select only the columns you need. This matters for covering indexes and network bandwidth.
- **Paginate all collection queries.** No unbounded result sets. Default page size of 25, max of 100.
- **Use EXISTS instead of COUNT** when checking for existence: `IF EXISTS (SELECT 1 FROM ...)` not `IF (SELECT COUNT(*) FROM ...) > 0`.
- **Batch large operations.** Don't update 1M rows in a single transaction — batch in chunks of 1000–5000.
- **Use appropriate isolation levels.** `READ COMMITTED` for most queries. `SNAPSHOT` for reports. `SERIALIZABLE` only when absolutely necessary.

### Never Do
- Never use `NOLOCK` / `READ UNCOMMITTED` as a performance fix. It reads dirty data. Fix the actual contention issue.
- Never use cursors for set-based operations. If you're looping row-by-row in SQL, rewrite as a set operation.
- Never put business logic in triggers. Triggers are invisible, untestable, and create maintenance nightmares.
- Never use dynamic SQL without parameterization.

---

## Entity Framework Core Patterns

### Read Operations
```csharp
// Always use AsNoTracking for read-only queries
var users = await context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserDto { Id = u.Id, Name = u.Name })
    .ToListAsync();
```

### Loading Related Data
- **Use explicit `.Include()`** for related entities you know you need.
- **Never enable lazy loading.** It causes N+1 queries silently and is the #1 EF Core performance killer.
- **Project to DTOs** with `.Select()` when you don't need the full entity graph — this generates optimized SQL.

### Write Operations
```csharp
// Keep change tracking scope small
var user = await context.Users.FindAsync(userId);
user.Name = newName;
await context.SaveChangesAsync();
```

### DbContext Lifetime
- **Scoped per request** in web applications (default DI registration).
- **Never share a DbContext across threads.** It is not thread-safe.
- **Never use a singleton DbContext.** It will grow unboundedly and eventually OOM.

### Compiled Queries
Use compiled queries for hot paths executed thousands of times:
```csharp
private static readonly Func<AppDbContext, Guid, Task<User?>> GetUserById =
    EF.CompileAsyncQuery((AppDbContext ctx, Guid id) =>
        ctx.Users.FirstOrDefault(u => u.Id == id));
```

---

## Connection Pooling

- **Always use connection pooling.** It is enabled by default in most drivers — do not disable it.
- Set pool size based on: `pool_size = num_cores * 2 + effective_spindle_count`. For cloud databases, start at 20 and tune.
- Monitor for pool exhaustion — it manifests as connection timeouts under load.
- Close/dispose connections promptly. `using` statements in C#, context managers in Python.
- For Azure SQL: use `Connection Timeout=30;Max Pool Size=100;` in connection strings.

---

## Transaction Boundaries

- **Keep transactions as short as possible.** Acquire lock → do work → commit. No external calls (HTTP, message queue) inside a transaction.
- **Use the appropriate isolation level.** Don't default to `SERIALIZABLE` — it's a concurrency killer.
- **Wrap logical units of work** in a single transaction. If step 3 of 5 fails, steps 1–2 should roll back.
- **Use the Outbox Pattern** when you need to update a database and publish an event atomically.

---

## Soft Deletes vs Hard Deletes

### Soft Delete (Preferred for Business Entities)
```sql
ALTER TABLE [User] ADD DeletedAt DATETIME2 NULL;
ALTER TABLE [User] ADD DeletedBy NVARCHAR(256) NULL;
CREATE INDEX IX_User_DeletedAt ON [User](DeletedAt) WHERE DeletedAt IS NULL;
```
- Add a **global query filter** in EF Core: `.HasQueryFilter(u => u.DeletedAt == null)`.
- Soft deletes preserve audit trail and allow recovery.
- Use for: Users, Orders, Projects, Documents — anything with business or compliance value.

### Hard Delete (Acceptable for Transient Data)
- Use for: Session data, cache entries, temporary processing records, logs past retention.
- Always cascade-delete orphaned child records to prevent data rot.

---

## Audit Columns

Every table MUST have these columns:

```sql
CreatedAt    DATETIME2    NOT NULL  DEFAULT GETUTCDATE(),
CreatedBy    NVARCHAR(256) NOT NULL,
UpdatedAt    DATETIME2    NOT NULL  DEFAULT GETUTCDATE(),
UpdatedBy    NVARCHAR(256) NOT NULL
```

- Populate via application code (not triggers) for transparency and testability.
- Use UTC for all timestamps. No local time zones in the database.
- For EF Core, implement `SaveChangesAsync` override to auto-populate:
  ```csharp
  public override Task<int> SaveChangesAsync(CancellationToken ct = default)
  {
      foreach (var entry in ChangeTracker.Entries<IAuditable>())
      {
          if (entry.State == EntityState.Added)
          {
              entry.Entity.CreatedAt = DateTime.UtcNow;
              entry.Entity.CreatedBy = _currentUser.Identity;
          }
          entry.Entity.UpdatedAt = DateTime.UtcNow;
          entry.Entity.UpdatedBy = _currentUser.Identity;
      }
      return base.SaveChangesAsync(ct);
  }
  ```

---

## Data Seeding

- **Seed reference data** (countries, currencies, status codes) in migrations — it's schema, not data.
- **Seed test data** in a separate seeding mechanism, never in migrations.
- Seeding must be idempotent — use upsert patterns (`MERGE` or `INSERT ... ON CONFLICT`).
- Never seed sensitive data (passwords, API keys) in code. Use environment-specific configuration.

---

## Backup & Recovery

- **Every production database MUST have automated backups** with tested restore procedures.
- **Test restores quarterly.** An untested backup is not a backup.
- Point-in-time recovery (PITR) should be enabled for all production databases.
- For Azure SQL: geo-redundant backups enabled, retention of at least 35 days.
- Document the Recovery Point Objective (RPO) and Recovery Time Objective (RTO) for each database.
- Keep backup retention aligned with compliance requirements (GDPR, SOX, etc.).

---

## Database Review Checklist

Before any schema change ships:

- [ ] All tables have primary keys.
- [ ] All relationships have foreign key constraints.
- [ ] All foreign key columns are indexed.
- [ ] Audit columns (CreatedAt, UpdatedAt, CreatedBy, UpdatedBy) are present.
- [ ] Migrations are reversible and idempotent.
- [ ] No `SELECT *` in production queries.
- [ ] All queries use parameterized inputs.
- [ ] Collection queries are paginated.
- [ ] Execution plans reviewed for new queries — no table scans on large tables.
- [ ] Connection pooling is configured.
- [ ] Backup and recovery procedures are documented and tested.
- [ ] Sensitive data is encrypted at rest and in transit.
