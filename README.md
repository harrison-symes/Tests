# Tests
A place for me to keep notes on how to write tests for different parts of an app

### Test Knex database functions
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
2. Write tests
> In root/tests/db/db.test.js
```
var test = require('ava')
//Can also use tape, but requires diff configuration

var configurationDatabase = require('../helpers/database-config')
configurationDatabase(test)
//Tell test to use the temp database we created

var db = require('../../server/db/db_functions')

test('getUser gets a single user by id in the database', function (t) {
  var expected = 'Sabrina'
  return db.getUser(1, t.context.connection)
    .then(function(result){
      var actual = result[0].user_name
      t.is(expected, actual)
    })
})

var object={
  user_name:'Mary',
}

test('saveUser saves a gingle user to the database', function (t){
  return db.saveUser(object, t.context.connection)
    .then((res)=>{
      t.is(res.length,1) //the length of array of id of items inserted should be only one
    })
})

test('saveUser saves user info into the database', function (t){
  return db.saveUser(object, t.context.connection)
    .then((res)=>{
      return t.context.connection('users').select().where('id',2).first()
    })
    .then((res)=>{
      t.is(res.user_name,"Mary")
    })
})
```
3. Add the test in package.json
> In root/package.json
```
"scripts": {
    "test": "ava -v tests/**/*.test.js",
    "test-db":"ava -v tests//db/*.test.js",
  }
```
This way you can use "npm run test-db" to run only tests in the db folder and avoid distractions.
