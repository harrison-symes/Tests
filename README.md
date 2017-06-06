# Tests
A place for me to keep notes on how to write tests for different parts of an app

### Menu
1. Test Knex database functions
2. Test server side API routes
3. Test client side API routes

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
    "test-db":"ava -v tests/db/*.test.js",
  }
```
This way you can use "npm run test-db" to run only tests in the db folder and avoid distractions.

### Test server side API routes
(first step is the same as the one above,using the same temp database as knex db functions test)
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
> In root/tests/api/server-users-test.js
```
var test = require('ava')
var request = require('supertest')

var app = require('../../server/server')
var configureDatabase = require('../helpers/database-config')

configureDatabase(test, function (db) {
  app.set('knex-database', db)
})
//Here we are using the temporary database for testing again, so that actual database is not modified
//'knex-database' is what I called knex in server.js file. It needs to be whatever you called knex if your server.js file

test.cb('GET /api/users/:id gets single user', function (t) {
  request(app)
    .get('/api/users/1')
    .expect('Content-Type', /json/)
    .expect(200)
    .end(function (err, res) {
      if (err) throw err
      t.is(res.body.length, 1)
      t.is(res.body[0].user_name, 'Sabrina')
      t.end()
    })
})

var object={
  user_name:'Mary',
}

test.cb('POST /api/users adds a new user to database', (t) => {
  request(app)
    .post('/api/users')
    .send(object)
    .expect(201)
    .end((err, res) => {
      if (err) throw err
      return t.context.connection('users').select().then((result)=>{
        t.is(result.length, 2)
        t.is(result[1].user_name, 'Mary')
        t.end()
      })
    })
})
```
3. Add the test in package.json
> In root/package.json
```
"scripts": {
    "test": "ava -v tests/**/*.test.js",
    "test-api":"ava -v tests/api/*.test.js",
  }
```
This way you can use "npm run test-api" to run only tests in the api folder and avoid distractions.

### Test client side API routes
reference:
jv's lecture: https://www.youtube.com/watch?v=BR0kK8alxW0&index=7&list=PLL6KkT3b5MhJA80rHn_P05wTonkkEOsWh
Alan's personal project: https://github.com/alan-jordan/gamr/blob/master/test/client/api.test.js
1. Install nock
> In terminal
```
npm i --save-dev nock
```
We use nock here because we don't actually want to hit the server during test. We just want to see if the api request functions are correct. **Therefore if the api server is broken, you will not know by running this test**

2. Write test
In this case, my api functions takes a callback as argument, and pass the result to the callback function.
So if your api function does other things the test will be different

> In root/tests/api/client-users.test.js
```
import test from 'ava'
import nock from 'nock'

import * as api from '../../client/api/users'

const testUserInfo={
  id:1,
  user_name:"Sabrina"
}

test.cb('getUser gets info of a single user', t =>{
  var scope = nock('http://localhost:80')
  //Here you're telling nock: when I visit localhost(the port is 80 I don't know why, it doesn't matter which port your app runs on), give me those things.
    .get('/api/v1/users/1')
    .reply(200, testUserInfo)
    //You're telling nock how to pretend to behave as the real server

  api.getUser(1, (actual)=>{
    scope.done()
    //Here you're saying: did you get a request? If yes this will not retrun error
    t.is(actual.user_name,"Sabrina")
    t.end()
  })
})

test.cb('saveUser hits the correct route, and takes two argument', t =>{
  var scope = nock('http://localhost:80')
    .post('/api/v1/users')
    .reply(201)

  api.saveUser(testUserInfo,()=>{
    scope.done()
    t.end()
  })
})
```
What the functions look like:
```
import request from 'superagent'

export function getUser(id, callback){
  request
    .get(`/api/v1/users/${id}`)
    .end((err,res) => {
      if (err) console.log("error occured while trying to get user info from server api", err)
      else {
        callback(res.body)
      }
    })
}

export function saveUser(object, callback){
  request
    .post('/api/v1/users')
    .send(object)
    .end((err) => {
      if (err) console.log("error occured while trying to get user info from server api", err)
      else {
        callback()
      }
    })
}
```
I might be using nock wrong but it doesn't test many things. And its error messages took me super long time to fix. It's a good way to double check if your routes written in api functions are correct I guess.
