## Pre-flight checklist

Run through this before requesting review. Each item is a real, recurring reviewer flag; the link points to the section that explains the rule.

- [ ] Migration was scaffolded with `pnpm --filter=@n8n/db migration:new` (timestamp + registration are automatic; the `migration-timestamp` lint rule catches drift). — [Creating Migrations](#creating-migrations)
- [ ] Identifiers go through **`escape.tableName(...)` / `escape.columnName(...)`**. Never hand-write `n8n_table` prefixes. — [Always escape identifiers](#always-escape-identifiers)
- [ ] **Match column type to value semantics.** Native `uuid` for UUIDs, `timestampTimezone()` for timestamps, a numeric type for numbers, `bool` for booleans, `json` for structured data. Never `varchar` as a catch-all. — [Column types](#column-types)
- [ ] **Pick the narrowest sane type within that category:** `int`/`smallint` not `bigint` when range allows; `text` not `varchar(255)` for unbounded strings; never `double` for version numbers. — [Column types](#column-types)
- [ ] **Default `notNull`**, relax only when justified. PK is implicitly NOT NULL. Migration's `notNull` matches the entity's nullability. — [NOT NULL and entity parity](#not-null-and-entity-parity)
- [ ] **Enum-like columns** carry `.withEnumCheck([...])` AND `.comment('explains values')`. Opaque IDs / unix timestamps / JSON shapes also get `.comment()`. — [Constrain enum-like strings](#constrain-enum-like-strings), [Add comments on columns](#add-comments-on-columns)
- [ ] **Every reference column has an explicit FK** with deliberate `onDelete`. Name FKs explicitly when SQLite recreate cycles risk duplicating them. Avoid polymorphic `(typeCol, idCol)` patterns. — [Foreign Key Constraints](#foreign-key-constraints), [General Design Guidance](#general-design-guidance)
- [ ] **Indexes match real query patterns.** A unique constraint already creates an index; a composite PK indexes its prefix. Mirror `withIndexOn(...)` to entity `@Index(...)`. — [Index Management](#index-management)
- [ ] **Sparse-unique columns:** use a partial index `WHERE col IS NOT NULL`. — [Index Management](#index-management)
- [ ] **Composite index column order** matches your actual `WHERE` / `ORDER BY` usage. — [Index Management](#index-management)
- [ ] **Entity ↔ migration parity**: column types, `notNull`, defaults, FKs, `@Index` decorators all match. — [Schema/Entity Drift](#schemaentity-drift)
- [ ] **If using `addColumns`, `dropColumns`, `addNotNull`, `dropNotNull`, `addEnumCheck`, or `dropEnumCheck`:** verified whether the target table has incoming FKs. If so, either set `withFKsDisabled = true as const` (in a `sqlite/` subclass if this is a `common/` migration) or use raw `ALTER TABLE ADD COLUMN` for nullable/defaulted columns. — [SQLite table recreation risk](#sqlite-table-recreation-risk)
- [ ] **No live-app value imports** in the migration body. Inline types/utility code locally. — [Never import entities as values](#never-import-entities-as-values)
- [ ] **`async down()` was tested locally**: `pnpm start && pnpm start -- db:revert && pnpm start` on **both** SQLite and Postgres. — [Reversibility](#reversibility)
- [ ] **One logical change per migration**; split unrelated table changes into separate files. — [Don't combine independent schema changes](#dont-combine-independent-schema-changes)
- [ ] **`up()` / `down()` reads as a list of intentions.** If either body grows past a screen or mixes schema operations with a multi-statement raw-SQL data move, extract the data move into a `private async` method on the same class (e.g. `private async backfillFromX(ctx)`). The top-level should orchestrate, not implement.
- [ ] **Precedent is the bar to fix, not perpetuate.** When the checklist conflicts with what an older migration does (e.g. redundant `.primary.notNull`, hand-quoted identifiers, missing `.comment()`), the checklist wins for new code — don't copy the violation forward. Note the old occurrences in the PR if you spotted them.
- [ ] **Regenerated the schema docs** with `pnpm db:schema:docs` and committed the `docs/generated/` changes. The DB Tests CI job fails on stale docs. — [Schema documentation](#schema-documentation)

Treat the checklist as a floor, not a ceiling.
If any item fails, fix it before opening review.

---

### Verify data integrity when copying tables

Count source vs temp before swapping; throw on mismatch. Silent row loss during a copy is one of the worst failure modes because it surfaces only when someone notices the missing data.

## Tests

**Every data migration ships with an integration test.** Schema-only migrations can usually be reviewed by reading the DSL calls. Data migrations cannot — they encode assumptions about row shape, JSON structure, NULL handling, and edge cases that only show up when the migration actually runs against representative data.

A data migration runs *once* per database, on production data, with no opportunity to retry cleanly. The cost of a bad migration is a customer-facing incident; the cost of a test is ten minutes.

Tests live in `packages/cli/test/migration/`, named to match the migration file (e.g. `1773000000000-create-credential-dependency-table.test.ts`). Use the helpers from `@n8n/backend-test-utils`:

- **`initDbUpToMigration(MigrationName)`** runs every migration *up to but not including* yours, leaving the DB in the exact state your migration will see in production.
- **`runSingleMigration(MigrationName)`** runs just your migration on top of that state.

Full helper API: `packages/@n8n/backend-test-utils/MIGRATION_TESTING.md`.

```typescript
import { initDbUpToMigration, runSingleMigration } from '@n8n/backend-test-utils';

describe('AddAndBackfillColumn1234567890000', () => {
	beforeEach(async () => {
		await initDbUpToMigration('AddAndBackfillColumn1234567890000');
	});

	it('backfills newCol from oldCol', async () => {
		// Seed rows in the pre-migration schema
		await dataSource.query(`INSERT INTO my_table (id, oldCol) VALUES ('1', 'foo')`);

		await runSingleMigration('AddAndBackfillColumn1234567890000');

		const [row] = await dataSource.query(`SELECT newCol FROM my_table WHERE id = '1'`);
		expect(row.newCol).toBe('transformed-foo');
	});

	it('skips rows with NULL oldCol without crashing', async () => {
		await dataSource.query(`INSERT INTO my_table (id, oldCol) VALUES ('1', NULL)`);
		await runSingleMigration('AddAndBackfillColumn1234567890000');
		// assert no error and row still exists
	});
});
```

**Insert fixtures via raw SQL only.** Repositories evolve with the schema and break older tests over time. Use `context.escape.tableName(...)` and `context.runQuery(sql, params)` directly.

**Test name describes behavior**, not the SQL: `'backfills newCol from oldCol'`, not `'runs UPDATE on my_table'`.

**What to cover:**
- The happy path (correctly transforms a typical row).
- Each edge case the migration claims to handle (NULL fields, malformed JSON, missing keys, legacy schema versions).
- Idempotency where applicable — running the migration twice shouldn't double-apply transformations.
- Both SQLite and Postgres if the migration branches on DB type.

---

