# Options

The options object modifies query behavior, either by applying additional clauses to the query itself or by changing how Massive handles results.

Options can be passed to most query and persistence functions as the final argument.

```javascript
db.tests.find({
  is_active: true
}, {
  offset: 20,
  limit: 10,
  only: true,
  stream: true
}).then(stream => {
  // a stream returning the active tests 21-30,
  // omitting rows from any descendant tables
});
```

## SQL Clauses

Certain SQL clauses are used with different types of query. For example, a `LIMIT` clause can only be used with a function which emits a `SELECT` such as `find` or `count`.

| Option key       | Use in | Description |
|------------------|--------|-------------|
| columns          | `SELECT` | Change the `SELECT` list by specifying an array of columns to include in the resultset. |
| limit            | `SELECT` | Set the number of rows to take. |
| offset           | `SELECT` | Set the number of rows to skip. |
| only             | `SELECT`, `UPDATE`, `DELETE` | Set to `true` to restrict the query to the table specified, if any others inherit from it. |
| order            | `SELECT` | An array of order objects (see below) or a literal string in the form `column1 ASC, column2 DESC`. |
| orderBody        | `SELECT` | If querying a document table, set to `true` to apply `options.order` to fields in the document body rather than the table. |
| onConflictIgnore | `INSERT` | If the inserted data would violate a unique constraint, do nothing. |

*nb. The `columns` and `order` properties allow comma-delimited string as well as array values. Take care when using raw strings since the values are interpolated directly into the emitted SQL. If user input is included in the values, you open yourself up to SQL injection attacks.*

### Ordering Results

The `order` option may be passed an array of order objects. These are used to build a SQL `ORDER BY` clause. An order object must contain a `field`; all other properties are optional.

* `field`: The name of the column being sorted on. May be a JSON path if sorting by an element nested in a JSON field or document table body. The column name will be quoted unless `raw` is also specified.
* `direction`: The sort direction, `ASC` or `DESC`.
* `type`: Define a cast type for values. Useful with JSON fields.
* `raw`: True to leave the `field` unquoted. Useful if you're ordering by something other than a single column value, for example `col1 + col2`.

## Results Processing

Results processing options are generally applicable to all query types, although `stream` is principally useful with query functions.

| Option key | Description |
|------------|-------------|
| build      | Set to `true` to return the query text and parameters *without* executing anything. |
| document   | Set to `true` to invoke [document table handling](/documents). |
| single     | Set to `true` to return the first result as an object instead of a results array. |
| stream     | Set to `true` to return results as a stream instead of an array. Streamed results cannot be `decompose`d. |
| decompose  | Provide a schema to transform the results into an object graph (see below). Not compatible with `stream`. |

### Decomposition Schemas

The `decompose` option takes a schema which represents the desired output structure. A schema is a JavaScript object with a few specific properties, and which may contain further schemas.

* `pk` (for "primary key") specifies the field in the resultset which uniquely identifies the entity represented by this schema.
* `columns` is either a map of fields in the resultset (keys) to fields in the output entity (values), or an array of field names if they do not need to be transformed.
* `array` is only usable on schemas nested at least one level deep. If `true`, the entities this schema represents are considered a collection instead of a nested object.

Any other key on a schema is taken to represent a nested schema, and nested schemas **may not be named** with one of the reserved keys. The following schema:

```javascript
db.user_tests.find({}, {
  decompose: {
    pk: 'user_id',
    columns: ['user_id', 'username'],
    tests: {
      pk: 'test_id',
      columns: {
        test_id: 'id',
        name: 'name'
      },
      array: true
    }
  }
}).then(...)
```

will transform this recordset:

```javascript
[
  {user_id: 1, username: 'alice', test_id: 1, name: 'first'},
  {user_id: 1, username: 'alice', test_id: 2, name: 'second'},
  {user_id: 2, username: 'bob', test_id: 3, name: 'third'},
]
```

into this:

```javascript
[{
  user_id: 1,
  username: 'alice',
  tests: [{
    id: 1,
    name: 'first'
  }, {
    id: 2,
    name: 'second'
  }]
}, {
  user_id: 2,
  username: 'bob',
  tests: [{
    id: 3,
    name: 'third'
  }]
}]
```

The `decompose` option can be applied to any result set, although it will generally be most useful with views and scripts.
