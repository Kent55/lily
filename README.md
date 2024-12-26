# [WIP] Lily (ORM)

Lily (ORM) is a TypeScript-based PostgreSQL ORM and schema management tool that is strictly-typed and provides: global configuration, migrations, hooks, and chainable queries. It enables developers to build and maintain evolving PostgreSQL-backed applications with confidence, clarity, and flexibility.

## Key Features
- **Chai-Style Chainable APIs:** Compose queries, schema definitions, and migrations in a fluent and intuitive manner.
- **Strict Typing and Strong Type Safety:** Every interaction is typed, catching errors at compile-time.
- **Hooks System:** Insert custom logic before or after select, insert, update, and delete operations.
- **Typed Migrations:** Version-controlled migrations let you evolve your database schema safely and revert changes if needed.
- **Advanced Queries:** Use DISTINCT, UNION, HAVING, and flexible WHERE/HAVING clauses to build complex SQL queries.
- **Environment Checks:** Enforces Node.js (≥18.0.0) and PostgreSQL (≥14.0.0) minimum versions.
- **Global Config and Inline Overrides:** Set global defaults or override them inline, including primary key defaults, minimum versions, and migrations table prefixing to avoid conflicts.
- **Schema Builder:** Programmatically define and modify schemas with typed column definitions, constraints, and foreign keys.

## Installation
```bash
npm install lily-orm // WIP
```
Ensure Node.js ≥18.0.0 and PostgreSQL ≥14.0.0. Lily will throw descriptive errors if these requirements are not met, guiding you to update your environment.

## Getting Started
### Configuring Defaults
```typescript
import { BaseORM, setGlobalConfig } from 'lily-orm';

setGlobalConfig({
  minNodeVersion: '18.0.0',
  minPostgresVersion: '14.0',
  primaryKeyDefaultType: 'int',
  migrationTablePrefix: 'lily_'
});

BaseORM.setDefaultConnectionConfig({
  user: 'postgres',
  host: 'localhost',
  database: 'testdb',
  password: 'password',
  port: 5432
});
```
Now Node.js must be at least 18.0.0, PostgreSQL at least 14.0.0, primary keys default to `int`, and migrations are tracked in `lily_migrations` rather than `migrations` to avoid conflicts.

### Defining a Model
```typescript
interface UserRow {
  id: number;
  name: string;
  age: number;
}

class UserORM extends BaseORM<UserRow> {
  mapRowToModel(row: any): UserRow {
    return { id: row.id, name: row.name, age: row.age };
  }
}

const UserModel = new UserORM('users');
```
`UserRow` defines the shape of a single user record (a database row). By extending `BaseORM<UserRow>`, all queries and operations related to `UserModel` return typed `UserRows`.

### Querying (Chai-Style)
Lily (ORM) provides a chai-style API for queries. You can reorder chainable methods before `.fetch()` without affecting the final SQL.

```typescript
const users = await UserModel
  .where.field('age').is(30)
  .and()
  .where.field('name').like('%John%')
  .select.all
  .order.by('name')
  .fetch();
```
Or reorder them
```typescript
const users = await UserModel
  .select.all
  .where.field('name').like('%John%')
  .and()
  .where.field('age').is(30)
  .order.by('name')
  .fetch();
```
The result is the same, as Lily constructs the correct SQL regardless of the order of chainable methods (apart from `.fetch()`).
#### Additional Query Features
- `.select.distinct` for distinct rows
- `.union.with('...')`, `.union.all('...')` for UNION queries
- `.having.field(...)` after grouping
- `.limit.to(n)` to limit results

### CRUD Operations
```typescript
await UserModel.insert({ name: 'Alice', age: 25 });
await UserModel.where.field('id').is(1).update({ age: 26 });
await UserModel.where.field('id').is(1).delete();
```

### Hooks
Insert logic before/after operations:
```typescript
UserModel.registerHook('beforeSelect', async (orm) => {
  console.log('About to run SELECT');
});
```
Available hooks: `beforeSelect`, `afterSelect`, `beforeInsert`, `afterInsert`, `beforeUpdate`, `afterUpdate`, `beforeDelete`, `afterDelete`.

## Schema Builder
Define or modify database schemas with the chai-style schema builder and column builder:

```typescript
import { SchemaBuilder } from 'lily-orm';

await new SchemaBuilder({ connectionString: 'postgres://user:pass@localhost:5432' })
  .create.table('users')
    .add.column('id').is.primary.key().done()
    .add.column('name').varchar(100).is.not.null.is.unique.done()
    .add.column('age').integer.default(0).done()
    .add.column('profile_id').integer.foreign.key('profiles', 'id').done()
  .with.data([{ name: 'John Doe', age: 30 }, { name: 'Jane Smith', age: 25 }])
  .execute();
```

#### ColumnBuilder Features
- Data types: `.integer`, `.text`, `.boolean`, `.date`, `.timestamp`, `.varchar(length)`, `.serial`, or `.type('custom')`
- Constraints: `.is.not.null`, `.is.unique`, `.is.primary.key(type?)`
- Foreign keys: `.foreign.key('table', 'col')`

If working with an existing database, consider setting `migrationTablePrefix` or generating a unique prefix to avoid conflicts with existing `migrations` tables. You can also override types inline, for instance `is.primary.key('INT UNSIGNED')`.

## Migrations
Migrations let you track and apply schema changes over time. Lily includes typed migrations and a `MigrationManager` to apply or roll them back.

**Existing Migrations as Examples:** The `migrations` directory contains example migrations (`001_create_users_table.ts`, `002_create_posts_table.ts`) that are runnable references. They aren’t just for show; you can run them to create example tables and understand how migrations work.

#### Running Migrations:
```typescript
import { MigrationManager } from 'lily-orm/schema/MigrationManager';
import { migrations } from 'lily-orm/migrations';

// MigrationManager accepts either a connection string or ORMOptions
const manager = new MigrationManager('postgres://user:pass@localhost:5432');

migrations.forEach(m => manager.registerMigration(m));
await manager.migrate.up();

console.log('Current version:', await manager.getCurrentVersion());

// Roll back to a previous state
await manager.migrate.down('001_create_users_table');
```

#### Writing Fresh Migrations for a New Database:
```typescript
import { Migration } from 'lily-orm/types/Migrations';
import { Client } from 'pg';

export const initialSetup: Migration = {
  id: '001_initial_setup',
  async up(client: Client) {
    await client.query(`CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(100), age INT);`);
  },
  async down(client: Client) {
    await client.query(`DROP TABLE IF EXISTS users;`);
  }
};
```
Register and run similarly to the examples provided.

Migrations for an Existing Database: Write migrations that only add or modify columns/tables without breaking what’s already there. If concerned about naming conflicts, set `migrationTablePrefix` globally or inline to ensure a unique migrations table name, e.g., `lily_migrations`.

## How to Run Migrations and Schema Builder in Practice
Lily (ORM) does not ship with a built-in CLI command. Instead, you integrate these operations into your own scripts. For instance, you could:

1. **npm Scripts:**
   - Create a `migrate.js` script in your project:
```javascript
// migrate.js
const { MigrationManager } = require('lily-orm/schema/MigrationManager');
const { migrations } = require('./migrations'); // your local migrations directory

(async () => {
  const manager = new MigrationManager('postgres://user:pass@localhost:5432');
  migrations.forEach(m => manager.registerMigration(m));
  await manager.migrate.up();
  console.log('Migrations applied.');
})().catch(console.error);
```
Then run `node migrate.js` or add `"migrate": "node migrate.js"` to your `package.json` scripts.

2. **Inline Code in Your App's Start-up:**
    - Your application’s initialisation code could run migrations before starting the main server logic.
3. **Custom CLI Tools:**
    - If you prefer a more elaborate setup, you can wrap Lily (ORM) calls in a CLI using tools like `yargs` or `commander`, providing commands like `myapp migrate up`.

Similarly, for the Schema Builder, you write a script that uses `SchemaBuilder` to create or modify tables, then run that script via npm or directly with `node`.

#### Example SchemaBuilder Script:
```javascript
// build-schema.js
const { SchemaBuilder } = require('lily-orm/schema/SchemaBuilder');

(async () => {
  await new SchemaBuilder({ connectionString: 'postgres://user:pass@localhost:5432' })
    .create.table('users')
      .add.column('id').is.primary.key().done()
      .add.column('name').varchar(100).is.not.null.done()
    .execute();

  console.log('Schema created/updated.');
})().catch(console.error);
```

Run `node build-schema.js` or add `"build-schema": "node build-schema.js"` in `package.json`.

By integrating Lily’s APIs into your own scripts or CLI tools, you have complete control and can run migrations and schema modifications whenever needed.

---

## Environment Checks
If Node.js or PostgreSQL versions are lower than configured minimums, Lily halts early and advises you to upgrade. Override these defaults with `setGlobalConfig()` if different versions are acceptable.

---

## Testing
Write Jest-based tests in a `tests/` directory for queries, hooks, migrations, and schema building. Use `npm test` to run them. By thoroughly testing each component, you ensure the robustness and maintainability of your application.

---

## Conclusion
Lily (ORM) provides a flexible, typed, and maintainable foundation for building, querying, and evolving PostgreSQL-backed applications. With chai-style chaining, environment checks, typed migrations, hooks, schema builders, and custom configurability (such as prefixing migration tables), Lily equips you with all the tools needed to manage and grow your database confidently.

Integrate migrations and schema building into your own scripts or npm tasks, reorder chained methods without changing outcomes, and rely on strict typing for safety and clarity. Lily (ORM) is ready to support you as your application and database requirements expand and evolve.

---

## License
Lily (ORM) is open-sourced under the MIT license. Other third party dependencies which Lily requires may have alternate
licences. Please ensure you verify each package and their respective licenses.
