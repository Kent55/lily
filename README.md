# WIP: Lily (Object-relational Mapper)
Lily (ORM) is a minimalist PostgreSQL ORM written in TypeScript that allows you to easily interact with your database. With built-in support for query building, hooks, and flexible chaining, Lily (ORM) makes database interactions simple and efficient while giving developers full control over their database operations.

- - -
## Features
- Flexible Query Chaining: The order in which you write the query chains doesn’t matter. Lily (ORM) automatically handles the correct order for SQL query construction.
- Hooks System: Customise behavior by injecting hooks into various stages of the query lifecycle, such as before and after select, insert, update, and delete operations.
- TypeScript Support: Fully typed with autocompletion, ensuring a smooth development experience.
- Minimalist and Lightweight: Only the essentials, no unnecessary overhead.

### Custom Error Handling (Dependency Injection)
Lily allows you to inject your own error-handling mechanism, giving you full control over how database-related errors are managed in your application. This is particularly useful for logging errors, integrating with external monitoring tools, or customising error messages.
#### How to use a Custom Error Handler
You can pass your custom error handler when creating an instance of `Lily`. The error handler should be a class or object that implements the `handleError` method, which takes the error and an optional context string as arguments.
```typescript
import { Lily } from 'lily-orm';

class CustomErrorHandler {
  static handleError(error: Error, context: string): void {
    console.error(`[Custom Error Handler] [${context}]`, error.message);
    // Additional logic such as logging to an external service
  }
}

const UserModel = new Lily('users', {
  connectionString: 'your_connection_string',
  errorHandler: CustomErrorHandler, // Inject custom error handler here
});

// Example usage
try {
  const users = await UserModel
    .where.field('age').is(30)
    .all();
} catch (err) {
  console.error('Handled by CustomErrorHandler');
}
```
#### Benefits
- Centralised error handling across all queries and operations.
- Enhanced flexibility for logging or monitoring database issues.
- Seamless integration with existing error-reporting infrastructure.


> <em>If no custom error handler is provided, Lily uses its default error handler, which logs errors to the console.</em>
- - -
## Installation
```bash
    $ npm install lily-orm # WIP: not published yet so ignore (currently finishing off unit tests for Lily)
```

- - -
## Getting Started
### 1. Create a Model Class
Create a class that extends `Lily` and implements the `mapRowToModel` method, which maps database rows to your model.
```typescript
import { Lily } from 'lily-orm';

class User extends Lily<UserModel> {
    mapRowToModel(row: any): UserModel {
        return {
            id: row.id,
            name: row.name,
            age: row.age,
        };
    }
}

const UserModel = new User('users', { user: 'postgres', host: 'localhost', database: 'testdb', password: 'password', port: 5432 });
```
### 2. Define Query Chains
You can build queries using flexible chains like `select`, `where`, `group`, `order`, and `limit`.
#### Example 1: Select All Fields with where Condition
```typescript
const users = await UserModel
  .where.field('name').equals('John')
  .selectAll;

console.log(users);
```
#### Example 2: Select Specific Fields and Apply where, group, and order
```typescript
const users = await UserModel
  .select.fields('id', 'name')
  .where.field('age').is(30)
  .limit.to(5)
  .order.by('name')
  .group.by('age')
  .all();

console.log(users);
```
#### Example 3: Query with `.and` and `.or`
```typescript
const users = await UserModel
  .where.field('age').is(30)
  .and()
  .where.field('name').like('%John%')
  .or()
  .where.field('city').is('New York')
  .all();
```
Generates:
```sql
SELECT * FROM users WHERE age = 30 AND name LIKE '%John%' OR city = 'New York'
```
### 3. Execute Insert, Update, and Delete Operations
Lily (ORM) also supports `insert`, `update`, and `delete` methods with hooks.
```typescript
// Insert a user
await UserModel.insert({ name: 'Alice', age: 25 });

// Update a user
await UserModel
  .where.field('id').is(1)
  .update({ name: 'Alice', age: 26 });

// Delete a user
await UserModel
  .where.field('id').is(1)
  .delete();
```

- - -
### Hook System
You can hook into various stages of the ORM lifecycle, such as before and after `select`, `insert`, `update`, and `delete` operations. This allows for powerful customisation and validation.
#### Available Hooks
- `beforeSelect`: Triggered before a `SELECT` query.
- `afterSelect`: Triggered after a `SELECT` query.
- `beforeInsert`: Triggered before an `INSERT` operation.
- `afterInsert`: Triggered after an `INSERT` operation.
- `beforeUpdate`: Triggered before an `UPDATE` operation.
- `afterUpdate`: Triggered after an `UPDATE` operation.
- `beforeDelete`: Triggered before a `DELETE` operation.
- `afterDelete`: Triggered after a `DELETE` operation.

#### Example: Register a Hook
```typescript
// Register a hook to log before a SELECT operation
UserModel.registerHook('beforeSelect', async (orm) => {
  console.log('Before SELECT:', orm);
});

// Register a hook to validate data before inserting
UserModel.registerHook('beforeInsert', async (orm, data) => {
  if (!data.name || !data.age) {
    throw new Error('Name and Age are required');
  }
});
```

- - -
## Query Chain Methods
### `where.field(field: string)`
Adds a `WHERE` clause with the specified field.
#### Supported Operators
- `.is(value: any)`: Adds `field = value`.
- `.equal(value: any)`: `Alias for .is(value)`.
- `.equals(values: any[])`: Adds `field IN (value1, value2, ...)`.
- `.contains(values: any[])`: Alias for `.equals(values)`.
- `.like(value: any)`: Adds `field LIKE value`.

#### Example:
```typescript
.where.field('name').equals('John')
```
`.and()`
Adds an `AND` connector between conditions.
#### Example:
```typescript
.where.field('age').is(30)
.and()
.where.field('name').like('%John%')
```
Generates:
```sql
WHERE age = 30 AND name LIKE '%John%'
```
`.or()`
Adds an `OR` connector between conditions.
#### Example:
```typescript
.where.field('age').is(30)
.or()
.where.field('name').like('%John%')
```
Generates:
```sql
WHERE age = 30 OR name LIKE '%John%'
```
### `select.fields(...fields: string[])`
Selects specific fields for the query.
#### Example:
```typescript
.select.fields('id', 'name')
```
### `selectAll`
Selects all fields (`SELECT *`).
#### Example:
```typescript
.selectAll
```
### `group.by(field: string)`
Groups results by the specified field.
#### Example:
```typescript
.group.by('age')
```
### `order.by(field: string)`
Orders the results by the specified field.
#### Example:
```typescript
.order.by('name')
```
### `limit.to(count: number)`
Limits the number of results.
#### Example:
```typescript
.limit.to(5)
```
### `all()`
Executes the `SELECT` query and returns the results.
#### Example:
```typescript
.all()
```
### `insert(data: Partial<T>)`
Inserts a new record into the database.
#### Example:
```typescript
.insert({ name: 'Alice', age: 25 })
```
### `update(data: Partial<T>)`
Updates existing records based on the `WHERE` conditions.
#### Example:
```typescript
.update({ name: 'Alice', age: 26 })
```
### `delete()`
Deletes records based on the `WHERE` conditions.
#### Example:
```typescript
.delete()
```

- - -

## Chaining Flexibility
Lily (ORM) allows for flexible query chaining. The order in which you chain methods like `where`, `select`, `group`, `order`, and `limit` doesn't matter. You can apply them in any order, and the ORM will handle constructing the correct SQL query.
#### Examples of Flexible Chaining
```typescript
const result1 = await UserModel
  .where.field('name').equals('John')
  .selectAll;

const result2 = await UserModel
  .selectAll
  .where.field('name').equals('John');

const result3 = await UserModel
  .select.fields('id', 'name')
  .where.field('age').is(30)
  .limit.to(5)
  .order.by('name')
  .group.by('age')
  .all();
```
In all of these examples, the order of method calls doesn't matter. The ORM will internally manage the correct construction of the SQL query.

- - -
## Conclusion
Lily (ORM) is a flexible, minimalist PostgreSQL ORM for TypeScript that lets you build queries with ease while maintaining full control over the execution. With its flexible chaining system, hooks, and support for all essential SQL operations, Lily (ORM) offers a simple yet powerful approach to database management in your applications.

For more advanced use cases, you can extend the base functionality and leverage hooks for complex workflows.

- - -
## License
MIT License. See `LICENSE` file for more details.
