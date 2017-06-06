# Tests
A place for me to keep notes on how to write tests for different parts of an app

### Knex database functions
1. Set up a temporary database in memory to be used for testing
**So that you don't modify your actual database during testing
> In root/knexfile.js
```
  test: {
     client: 'sqlite3',
     connection: {
       filename: ':memory:'
     },
     useNullAsDefault: true
   },
```
> In root/tests/helpers/database-config.js
```
var knex = require('knex')
var config = require('../../knexfile').test
//require the test environment set up in knexfile

module.exports = (test, createServer) => {
  // Create a separate in-memory database before each test.
  // In our tests, we can get at the database as `t.context.connection`.
  test.beforeEach(function(t){
    t.context.connection=knex(config)
    if(createServer) t.context.app = createServer(t.context.connection)
    return t.context.connection.migrate.latest()
      .then(function(){
        return t.context.connection.seed.run()
      })
  })

  test.afterEach(function(t){
    t.context.connection.migrate.rollback()
      //Rollback the new data entered into the table during test after each test.
    .then(() => {
      return t.context.connection.destroy()
     // Destroy the database connection after each test.
     })
  })
}
```
