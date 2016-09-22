# rethinkdb-101
Agile web-development with RethinkDB

## Getting Started

* Start RethinkDB
```
c:\rethinkdb>rethinkdb.exe
Running rethinkdb 2.3.5-windows (MSC 190023506)...
Running on 6.2.9200 (Windows 8, Server 2012)
Loading data from directory c:\rethinkdb\rethinkdb_data
Listening for intracluster connections on port 29015
Listening for client driver connections on port 28015
Listening for administrative HTTP connections on port 8080
Listening on cluster address: 127.0.0.1
Listening on driver address: 127.0.0.1
Listening on http address: 127.0.0.1
To fully expose RethinkDB on the network, bind to all addresses by running rethinkdb with the --bind all command line option.
Server ready, "DESKTOP" b986a7e3-eb4a-4d6e-adad-5ef06b4f1856
warn: Problem when checking for new versions of RethinkDB: HTTP request to update.rethinkdb.com failed.
```

* Go to http://127.0.0.1:8080/

* Click on **Data Explorer** tab in the web-application. You can use this page to run ReQL queries.

* Create database and table.
```js
r.dbCreate('samples');
r.db('samples').tableCreate('zips');
``` 

* Add new documents to the database.
```js
r.db('samples').table('zips').insert([
    { "city" : "AGAWAM", "loc" : [ -72.622739, 42.070206 ], "pop" : 15338, "state" : "MA", "_id" : "01001" },
    { "city" : "CUSHMAN", "loc" : [ -72.51564999999999, 42.377017 ], "pop" : 36963, "state" : "MA", "_id" : "01002" },
    { "city" : "CHALKYITSIK", "loc" : [ -143.638121, 66.71899999999999 ], "pop" : 99, "state" : "AK", "_id" : "99788" },
    { "city" : "NUIQSUT", "loc" : [ -150.997119, 70.19273699999999 ], "pop" : 354, "state" : "AK", "_id" : "99789" },
    { "city" : "JUNEAU", "loc" : [ -134.529429, 58.362767 ], "pop" : 24947, "state" : "AK", "_id" : "99801" },
    { "city" : "PATEROS", "loc" : [ -119.913322, 48.059147 ], "pop" : 696, "state" : "WA", "_id" : "98846" },
    { "city" : "PESHASTIN", "loc" : [ -120.613928, 47.581294 ], "pop" : 1030, "state" : "WA", "_id" : "98847" },
    { "city" : "QUINCY", "loc" : [ -119.845922, 47.197574 ], "pop" : 7429, "state" : "WA", "_id" : "98848" }
]);
```

* Show zips we've just added to the table: records, count.
```js
r.db('samples').table('zips');
r.db('samples').table('zips').count();
```

* Find document by id (primary key). RethinkDB is an automatically generated UUID field. You can set your own ID in JSON document to avoid default UUID function.
```js
r.db('samples').table('zips').get('838ac293-0a50-450c-af33-df5fd4f13d5b');
```

* Sample queries
```js
// Filtering
r.db('samples').table('zips').filter(r.row('state').eq('MA'));
// The same query using an anonymous function
r.db('samples').table('zips').filter(function (i) { return i('state').eq('MA'); });
r.db('samples').table('zips').filter(r.row('city').match('^C'));
r.db('samples').table('zips').filter(r.row('city').match('^C')).pluck('city', 'state', 'pop');
r.db('samples').table('zips').filter(r.row('city').match('^C')).pluck('city', 'state', 'population': 'pop');

// Create index and query using it to avoid table scan
r.db('samples').table('zips').indexCreate('state');
// You should specify index here, because RethinkDB does not have an optimizer yet.
// You cannot chain multiple getAll commands. 
// Use a compound index to efficiently retrieve documents by multiple fields.
r.db('samples').table('zips').getAll('MA', { index: 'state' });

// Accessing nesterd fields
r.table('users').get(10001).pluck({contact: [{phone: 'work', im: 'msn'}]};

// Ordering and paging
r.db('samples').table('zips').orderBy('state').pluck('state', 'city');
r.db('samples').table('zips').orderBy(r.desc('state')).pluck('state', 'city');

// Creating compound index to sort by multiple columns
r.db('samples').table('zips').indexCreate('stateAndCity', [r.row('state'), r.row('city')]);
r.db('samples').table('zips').orderBy({ 'index': 'stateAndCity' }).pluck('state', 'city');
r.db('samples').table('zips').orderBy('state').skip(3).limit(2).pluck('city', 'state');
```

## Join tables

```js
// Create states table
r.db('samples').tableCreate('states');
r.db('samples').table('states').insert([
    { id: 'AK', name: 'Alaska' },
    { id: 'WA', name: 'Washington' }
]);

r.db('samples').tableCreate('states2');
r.db('samples').table('states2').insert([
    { state: 'AK', name: 'Alaska' },
    { state: 'WA', name: 'Washington' }
]);

// In RethinkDB joins are automatically distributed

// Inner joins (primary key). You can also join tables using secondary indexes.
r.db('samples').table('zips').eqJoin('state', r.db('samples').table('states'))
  .pluck({ left: ['id', 'city', 'state' ], right: ['name'] })
  .zip()

// Inner joins (any column). Less efficient then the previous join.
r.db('samples').table('zips')
  .innerJoin(r.db('samples').table('states2'), 
    function (z, s) { return z('state').eq(s('state')); })
  .map({
    id: r.row('left')('id'),
    name: r.row('left')('city'),
    state: r.row('right')('name')
  })

// Outer join
r.db('samples').table('zips')
  .outerJoin(r.db('samples').table('states2'), 
    function (z, s) { return z('state').eq(s('state')); })
  .pluck({ left: ['id', 'city', 'state' ], right: ['name'] })
  .zip()
```

## Map-reduct in RethinkDB

```js
// Map-reduce works by processing the data on each server in parallel and then combining those results into one set.
// Group - map - reduce process

// Be careful! Make sure your reduction function doesn’t assume the reduction step executes from left to right!

// Calculate state total population
r.db('samples').table('zips')
  .group('state')
  .map(r.row('pop'))
  .reduce(function (a, b) {
    return a.add(b);
  });

r.db('samples').table('zips')
  .group('state')
  .sum('pop');
```

## CRUD operation

* Insert a document.
```js
r.db('samples').table('zips').insert({ 'text': 'test', 'value': 100 });
// Check that document has been added to the table
r.db('samples').table('zips').filter(r.row('text').eq('test'));
```

* Update a document.
```js
r.db('samples').table('zips').filter(r.row('text').eq('test')).update({ 'value': 500 });
// Check that document has been updated
r.db('samples').table('zips').filter(r.row('text').eq('test'));
```

* Delete a document.
```js
r.db('samples').table('zips').filter(r.row('text').eq('test')).delete();
// Check that document has been removed
r.db('samples').table('zips').filter(r.row('text').eq('test'));
```

* Drop database.
```js
r.dbDrop('samples');
```

## Running cluster on localhost

* Run the first instance.
```
c:\rethinkdb>rethinkdb.exe
Running rethinkdb 2.3.5-windows (MSC 190023506)...
Running on 6.2.9200 (Windows 8, Server 2012)
Loading data from directory c:\rethinkdb\rethinkdb_data
Listening for intracluster connections on port 29015
Listening for client driver connections on port 28015
Listening for administrative HTTP connections on port 8080
Listening on cluster address: 127.0.0.1
Listening on driver address: 127.0.0.1
Listening on http address: 127.0.0.1
To fully expose RethinkDB on the network, bind to all addresses by running rethinkdb with the --bind all command line option.
Server ready, "LAPTOPTA3KJMF5xq8" b986a7e3-eb4a-4d6e-adad-5ef06b4f1856
warn: Problem when checking for new versions of RethinkDB: HTTP request to update.rethinkdb.com failed.
```

* Run the second instance and join the first one.
```
c:\rethinkdb>rethinkdb.exe --port-offset 1 --directory rethinkdb_data2 --join 127.0.0.1:29015
In recursion: removing file 'c:\rethinkdb\rethinkdb_data2\tmp'
warn: Trying to delete non-existent file 'c:\rethinkdb\rethinkdb_data2\tmp'
Initializing directory c:\rethinkdb\rethinkdb_data2
Running rethinkdb 2.3.5-windows (MSC 190023506)...
Running on 6.2.9200 (Windows 8, Server 2012)
Loading data from directory c:\rethinkdb\rethinkdb_data2
Listening for intracluster connections on port 29016
Connected to server "LAPTOPTA3KJMF5xq8" b986a7e3-eb4a-4d6e-adad-5ef06b4f1856
Listening for client driver connections on port 28016
Listening for administrative HTTP connections on port 8081
Listening on cluster address: 127.0.0.1
Listening on driver address: 127.0.0.1
Listening on http address: 127.0.0.1
To fully expose RethinkDB on the network, bind to all addresses by running rethinkdb with the --bind all command line option.
Server ready, "LAPTOPTA3KJMF5gol" 68fe6d59-6898-4c2e-b8c2-f54c8b4d2d8c
warn: Problem when checking for new versions of RethinkDB: HTTP request to update.rethinkdb.com failed.
```

* Web-interface: [http://localhost:8080/](http://localhost:8080/) or [http://localhost:8081/](http://localhost:8081/).

* Database connection: localhost:28015 or localhost:28016

## Useful Links

* [RethinkDB](https://www.rethinkdb.com/)
* [RethinkDB installation](https://www.rethinkdb.com/docs/install/)
* [Thirty-second quickstart](https://www.rethinkdb.com/docs/quickstart/)
* [Ten-minute guide](https://www.rethinkdb.com/docs/guide/javascript/)
* [RethinkDB Cookbook](https://www.rethinkdb.com/docs/cookbook/javascript/) 
* [RethinkDB Cheat sheet](https://www.rethinkdb.com/docs/sql-to-reql/javascript/) 