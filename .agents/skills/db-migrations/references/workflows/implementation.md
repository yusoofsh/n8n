### Migration Types

| Interface | When to use |
|---|---|
| `ReversibleMigration` | Schema changes that can be cleanly undone (add/drop column, create/drop table). Requires a working `down()`. |
| `IrreversibleMigration` | Data transformations, destructive changes, or anything where `down()` would lose data. No `down()` allowed. |

### MigrationContext API

Source of truth: `packages/@n8n/db/src/migrations/migration-types.ts`. Check the source for exact signatures when in doubt.

```typescript
interface MigrationContext {
	// Database info
	dbType: 'postgresdb' | 'sqlite';
	isSqlite: boolean;
	isPostgres: boolean;
	tablePrefix: string;
	dbName: string;

	// Schema DSL
	schemaBuilder: { createTable, dropTable, addColumns, dropColumns, column,
		createIndex, dropIndex, addForeignKey, dropForeignKey,
		addNotNull, dropNotNull };

	// Query execution
	runQuery<T>(sql: string, namedParameters?: object): Promise<T>;
	runInBatches<T>(query: string, operation: (rows: T[]) => Promise<void>, limit?: number): Promise<void>;
	copyTable(from: string, to: string, fromFields?: string[], toFields?: string[], batchSize?: number): Promise<void>;

	// Utilities
	escape: { tableName(n: string): string; columnName(n: string): string; indexName(n: string): string };
	parseJson<T>(data: string | T): T;
	loadSurveyFromDisk(): string | null;
	logger: Logger;
	migrationName: string;
	queryRunner: QueryRunner;  // Avoid direct use — prefer runQuery()
}
```

### Creating Migrations

> **Temporary timestamp workaround:** This repository currently has future-dated migrations, with the head at `1784000000008` (`2026-07-14T03:33:20.008Z`). Until real time passes that timestamp, a migration created with `Date.now()` would sort before the deployed head and can run out of order on databases that already applied later migrations. Use the generator during this window — it picks `max + 1` when needed. See [PR #30511](https://github.com/n8n-io/n8n/pull/30511) for context.

Migration files are named `{TIMESTAMP}-{DescriptiveName}.ts`. The timestamp must be strictly greater than every existing migration timestamp in this package (across `common/`, `postgresdb/`, and `sqlite/`). TypeORM runs unrecorded migrations in timestamp order, so inserting a value below the current max corrupts ordering on databases that have already executed the later migrations.

Use the generator — it picks a safe timestamp, writes the scaffold, and registers the migration in the relevant `index.ts` files:

```sh
pnpm --filter=@n8n/db migration:new <Name> [--folder=common|postgresdb|sqlite]
```

`<Name>` is PascalCase and describes the change (e.g. `AddTracingToExecution`). `--folder` defaults to `common`; use `postgresdb` or `sqlite` only for dialect-specific migrations. The generator picks `Date.now()` when it's greater than the current head, otherwise `max + 1`.

The `migration-timestamp` rule in `@n8n/code-health` enforces both invariants (strict ordering and no far-future fabrication) at lint time; the generator is the easy path, the rule is the safety net.

### Applying and Reverting Migrations

Pending migrations are applied during normal n8n startup. In a local checkout, run `pnpm start` with the target code version to apply them manually.

To revert the most recently applied reversible migration, use the CLI command:

```sh
n8n db:revert
```

In a local checkout, run the same command through the package script:

```sh
pnpm start -- db:revert
```

Do **not** revert migrations by editing the migrations table or running
hand-written SQL. `db:revert` runs the migration's `down()` method and
preserves TypeORM's migration bookkeeping.

### Prefer `runQuery()` over `queryRunner`

Run SQL through `runQuery()` from `MigrationContext`. Never call `queryRunner.query()` or `queryRunner.manager.*` from a migration.

**Why:** `runQuery()` handles named parameter binding consistently, while identifiers still need `escape.tableName()`, `escape.columnName()`, and `escape.indexName()`. `queryRunner.query()` bypasses the parameter helper. `queryRunner.manager` calls couple the migration to TypeORM entity definitions, which change over time — a migration that worked at v1.0 can break at v2.0 if the entity shape evolves.

### Don't edit a previously merged migration

Once shipped, migrations are immutable. Write a new migration. To remove a column added by an earlier migration, do it in a separate follow-up migration (typically in a later release — see [Deprecate columns, then drop in a follow-up](#deprecate-columns-then-drop-in-a-follow-up)).

## Schema Migrations

## Data Migrations

Data migrations transform existing rows: parsing JSON, backfilling columns, migrating data between tables, cleaning up invalid data.

### Mixed Schema + Data Migrations

When a migration both adds a column and backfills data, structure it clearly with one method per concern:

```typescript
export class AddAndBackfillColumn1234567890000 implements IrreversibleMigration {
	async up(ctx: MigrationContext) {
		await ctx.schemaBuilder.addColumns(
			'my_table',
			[ctx.schemaBuilder.column('newCol').text],
			{ recreatesOnSqlite: true },
		);
		await this.backfillNewCol(ctx);
	}

	private async backfillNewCol({ escape, runQuery, runInBatches }: MigrationContext) {
		const table = escape.tableName('my_table');
		await runInBatches<{ id: string; oldCol: string }>(
			`SELECT id, oldCol FROM ${table}`,
			async (rows) => {
				for (const row of rows) {
					const transformed = transform(row.oldCol);
					await runQuery(`UPDATE ${table} SET newCol = :val WHERE id = :id`, {
						val: transformed,
						id: row.id,
					});
				}
			},
		);
	}
}
```

The schema change and the data backfill have different failure modes, different transaction implications, and different testing needs — keeping them in separate methods makes review easier and lets `down()` (if reversible) call the same helpers in reverse.

### Atomic SQL within the migration's transaction

Some migrations override with `transaction = false as const` for big DDL on engines that disallow it inside a transaction. The DSL/wrapper sets `transaction = false` automatically when `withFKsDisabled = true`. Otherwise, leave transactions alone — TypeORM wraps each migration in one by default.

---

### Single Migration File or Separate for SQLite & Postgres

- **Small differences** (a single statement, a CHECK constraint, slightly different syntax): keep one migration in `common/` and branch on `isSqlite` / `isPostgres`.
- **Large differences** (different table recreation strategies, different intermediate steps, fundamentally different SQL): write **separate files** in `postgresdb/` and `sqlite/`. A common migration full of `if (isSqlite) { ... }` blocks is harder to read and review than two focused files.

If only Postgres needs the change, just put the file in `postgresdb/`; don't write a no-op SQLite migration with `if (isPostgres)`. For SQLite column adds, follow the [SQLite table recreation risk](#sqlite-table-recreation-risk) decision tree before deciding whether a common migration is enough or a SQLite subclass/raw `ALTER TABLE` path is needed.

### Prefer `ALTER` over drop-and-recreate

For renames, use `ALTER TABLE ... RENAME TO`. Faster, atomic, no data-loss risk.

---

