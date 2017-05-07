# Database

<!-- TOC -->

- [Introduction](#introduction)
- [Accessing Databases](#accessing-databases)
    - [DB Facade](#db-facade)
- [Setup](#setup)
    - [SQLite File](#sqlite-file)
    - [Encryption](#encryption)
    - [Migrations](#migrations)
- [Events](#events)
    - [BreezeDatabaseEvent](#breezedatabaseevent)
    - [BreezeQueryEvent](#breezequeryevent)
    - [BreezeMigrationEvent](#breezemigrationevent)
- [Running Raw SQL Queries](#running-raw-sql-queries)
    - [Running A Select Query](#running-a-select-query)
    - [Running An Insert Statement](#running-an-insert-statement)
    - [Running An Update Statement](#running-an-update-statement)
    - [Running A Delete Statement](#running-a-delete-statement)
    - [Running A General Statement](#running-a-general-statement)
- [Query Reference](#query-reference)
- [Transactions](#transactions)
- [Running Multiple Queries](#running-multiple-queries)

<!-- /TOC -->

## Introduction

BreezeDb provides a simple and developer-friendly API for interacting with SQLite databases. SQL connections are always asynchronous which maximizes the responsiveness of your app. Multiple SQL connections are supported as well.

## Accessing Databases

To start interacting with databases, use the `BreezeDb` class. The static `db` getter returns a reference to the default database connection. If you are using multiple SQL databases and want to access a different database connection, you can use the `getDb` method along with the name of your database. A new database reference will be created for you, if it does not exist yet:

```as3
var db:IBreezeDatabase = BreezeDb.db;

var customDb:IBreezeDatabase = BreezeDb.getDb("custom-db");
```

### DB Facade

Most of the time you only need to interact with the default database. In these cases you can simply reference the `DB` facade class. This class provides static methods that mirror the API defined by the `IBreezeDatabase` interface:

```as3
// These two approaches are identical
BreezeDb.db.table("photos").fetch(resultCallback);

DB.table("photos").fetch(resultCallback);
```

The point of the `DB` facade is for easier code readability, thus it is used in most of the snippets in this documentation.

## Setup

Before you start making SQL queries, the database must be set up. You can do so by simply calling the `setup` method with a callback:

```as3
DB.setup(function(error:Error):void
{
    if(error == null)
    {
        // The database has been set up
    }
});
```

### SQLite File

By default the library creates a file for each database in the device's application storage directory (i.e. `File.applicationStorageDirectory`). The file is named after the database and uses `.sqlite` as an extension. The default database is called `database`, thus the following two calls return the same reference:

```as3
var db1:IBreezeDatabase = BreezeDb.db;
var db2:IBreezeDatabase = BreezeDb.getDb("database");

trace(db1 == db2); // true
```

To change the default directory, set the `file` property:

```as3
DB.file = File.documentsDirectory;
```

The documents directory will now be used for all newly set up databases.

> Custom directory must be set before calling the `setup` method.

You can also change the default file extension by setting the `fileExtension` property:

```as3
DB.fileExtension = ".database";
```

If needed, you can provide a reference to a file when calling the `setup` method. This file is then used instead of the default directory and extension settings:

```as3
DB.setup(callback, File.applicationStorageDirectory.resolvePath("custom-db-file.dat));
```

### Encryption

You may wish to encrypt the database to deter users from viewing and modifying your app's data on jailbroken devices.

> You cannot add, modify or remove an encryption key once the database has been created.

To enable encryption supply an encryption key before calling the `setup` method. It can be an arbitrary `String` that has at least one character:

```as3
DB.encryptionKey = "encryption_key";
```

You may also pass in an existing `ByteArray`. It must be exactly 16 bytes long, otherwise an error will be thrown:

```as3
// Write 16 bytes
var key:ByteArray = new ByteArray();
for(var i:int = 0; i < 16; ++i)
{
    key.writeByte(i);
}

DB.encryptionKey = key;
```

### Migrations

Migrations allow you to create and edit your database structure between app versions.

> See the [Migrations document](../migrations/) to learn more about this feature.

BreezeDb allows you to run migrations either during a database setup or after setup at any time.

To run migrations during the setup, set the `migrations` property before calling the `setup` method. All given migrations will run before the setup callback is triggered. If any migration fails, the database will not be set up. You can set either one or multiple migration classes:

```as3
DB.migrations = [Migration_Create_Table_Photos, Migration_Insert_Default_Photos];
DB.setup(function(error:Error):void
{
    if(error == null)
    {
        // Database is setup and migrations have run
    }
});
```

Use the `runMigrations` method to run migrations manually at any time after the database has been setup. Again, either one or multiple classes can be specified in the first argument:

```as3
DB.runMigrations(Migration_Create_Photos_Table, function(error:Error):void
{
    if(error == null)
    {
        // The migration has run successfully
    }
});
```

## Events

The BreezeDb API provides callbacks as a way to retrieve response. It is more convenient than using event listeners due to less code needed and better response formatting for the various methods that are available.

However you can still make use of the standard events, for example if you want to log errors or executed queries for analytics. There are 3 main types of events:

* `BreezeDatabaseEvent` - provides information about database operations, e.g. setup, close and transaction state.
* `BreezeQueryEvent` - provides information about executed queries.
* `BreezeMigrationEvent` - provides information about executed database migration.

To listen to any of these events, you need to obtain a reference to your database. When using the static `DB` facade, use the `instance` getter:

```as3
BreezeDb.db.addEventListener(BreezeDatabaseEvent.SETUP_SUCCESS, onDatabaseSetup);

DB.instance.addEventListener(BreezeDatabaseEvent.SETUP_SUCCESS, onDatabaseSetup);
```

### BreezeDatabaseEvent

| Event name          | Description                                 |
|---------------------|---------------------------------------------|
| `SETUP_SUCCESS`     | A database has been set up successfully.    |
| `SETUP_ERROR`       | Failed to set up a database.                |
| `CLOSE_SUCCESS`     | A database was closed successfully.         |
| `CLOSE_ERROR`       | Failed to close a database.                 |
| `BEGIN_SUCCESS`     | A transaction began successfully.           |
| `BEGIN_ERROR`       | Failed to begin a transaction.              |
| `COMMIT_SUCCESS`    | A transaction was committed successfully.   |
| `COMMIT_ERROR`      | Failed to commit a transaction.             |
| `ROLL_BACK_SUCCESS` | A transaction was rolled back successfully. |
| `ROLL_BACK_ERROR`   | Failed to roll back a transaction.          |

When you receive one of the `ERROR` events, use the event's `error` property to see details about the error that occurred during the corresponding operation.

### BreezeQueryEvent

| Event name | Description                        |
|------------|------------------------------------|
| `SUCCESS`  | A query was executed successfully. |
| `ERROR`    | Failed to execute a query.         |

The event provides access to `BreezeSQLResult` object via the `result` property. To see the raw SQL query that was executed, use the `query` property. The event's `error` property references any error that occurred when executing the corresponding query.

### BreezeMigrationEvent

| Event name    | Description                                                                                                            |
|---------------|------------------------------------------------------------------------------------------------------------------------|
| `RUN_SUCCESS` | A migration ran successfully.                                                                                          |
| `RUN_ERROR`   | Failed to run a migration.                                                                                             |
| `SKIP`        | A migration was skipped (it already ran in the past).                                                                  |
| `FINISH`      | Finished running migrations. Note this does not mean all migrations have run, only that no further migrations will run.|

This event has no custom properties. You can listen to the `BreezeQueryEvent.ERROR` event to be notified about queries that fail during migrations.

## Running Raw SQL Queries

The database provides an API for running raw SQL queries. You can run `SELECT`, `INSERT`, `DELETE` and `UPDATE` queries by using the corresponding method. For any other queries, you can use the generic  `query` method.

### Running A Select Query

To run a raw `SELECT` query, use the `select` method on the `DB` facade or an `IBreezeDatabase` reference:

```as3
DB.select("SELECT id, title FROM photos", function(error:Error, results:Collection):void
{

});
```

The first argument is always the raw query to be executed. The query above does not use parameter bindings, thus the callback is used as the second argument. However, if a query has parameters, you must provide a key-value `Object` as the second argument. Query parameters provide protection against SQL injection attacks and are typically used for values of the `WHERE` clause:

```as3
DB.select("SELECT id, title FROM photos WHERE id = :id", { id: 2 }, function(error:Error, results:Collection):void
{

});
```

By making the `SELECT` query using the `select` method, your callback receives the results typed to a `Collection` instead of a generic `BreezeSQLResult`.

### Running An Insert Statement

Running other types of queries is very similar. To insert a new row, use the `insert` method. In this case the query does not return any specific data, thus only a single `Error` parameter should be specified in the callback:

```as3
DB.insert("INSERT INTO photos (id, name) VALUES (:id, :name)", {id: 1, name: "Test"}, function(error:Error):void
{
    if(error == null)
    {
        // Insert was successful
    }
});
```

> The `insert` method provided by the database can insert only one row. To perform a multi-row insert, use [BreezeQueryBuilder instead](../query-builder/#inserts).

### Running An Update Statement

To update a set of records, use the `update` method. The callback receives an `int` that specifies how many rows were affected by the query:

```as3
DB.update("UPDATE photos SET title = :title WHERE id = :id", { id: 1, title: "Sunset" }, function(error:Error, rowsAffected:int):void
{
    trace("Updated", rowsAffected, "row(s)");
});
```

### Running A Delete Statement

Deletes can be performed using the `remove` method. Just as in the case of the `update` method, the callback receives an `int` that specifies how many rows were deleted:

> Remember that this method is ```remove``` and not ```delete```. The word `delete` is a reserved word in ActionScript and cannot be used as a method name.

```as3
DB.remove("DELETE FROM photos WHERE id = :id", { id: 5 }, function(error:Error, rowsDeleted:int):void
{
    trace("Deleted", rowsDeleted, "row(s)");
});
```

### Running A General Statement

To execute any other raw queries, you can use the `query` method. In this case the result data is returned as a generic `BreezeSQLResult`:

```as3
DB.query("SELECT * FROM photos WHERE id > :id", { id: 2 }, function(error:Error, result:BreezeSQLResult):void
{
    if(error == null)
    {
        trace(result.data);
        trace(result.rowsAffected);
        trace(result.lastInsertRowID);
    }
});
```

## Query Reference

When you call any of the raw query methods, you can store the returned instance of `BreezeQueryReference`. It can be used to cancel the callback if needed. Note that doing so **does not stop** the actual SQL query from running, it only stops the callback from being called:

```as3
var reference:BreezeQueryReference = DB.insert("INSERT INTO photos (id, name) VALUES (:id, :name)", {id: 1, name: "Test"}, function(error:Error):void
{

});

...

if(!reference.isCompleted)
{
    // The callback will no longer be triggered,
    // however the query will still be executed
    reference.cancel();
}
```

## Transactions

Transactions can be used if you need to make atomic commits. Note that beginning / ending the transcation is an asynchronous process, therefore you must provide a callback method to be notified when you can start making a query within the transaction:

```as3
// Begin a new transaction
DB.beginTransaction(function(error:Error):void
{
    if(error == null)
    {
        // We can start making queries
    }
});

// Commit the transaction
DB.commit(function(error:Error):void
{
    if(error == null)
    {
        // The transaction has been comitted
    }
});

// Rollback the transaction
DB.rollBack(function(error:Error):void
{
    if(error == null)
    {
        // The transaction has been rolled back
    }
});
```

Below is an example of how you would use a transaction. Note you can achieve the same result with much less code by using the [multiQueryTransaction method](#running-multiple-queries).

```as3
DB.beginTransaction(onTransactionBegan);

function onTransactionBegan(error:Error):void
{
    if(error == null)
    {
        // Make the first query
        DB.table("photos").where("id", 15).remove(onFirstQueryCompleted);
    }
}

function onFirstQueryCompleted(error:Error, rowsDeleted:int):void
{
    if(error == null)
    {
        // Continue with the second query
        DB.table("photos").where("id", 18).remove(onSecondQueryCompleted);
    }
    else
    {
        // Roll back if an error occurred
        DB.rollBack(onTransactionRolledBack);
    }
}

function onSecondQueryCompleted(error:Error, rowsDeleted:int):void
{
    if(error == null)
    {
        // Commit the transaction
        DB.commit(onTransactionCommitted);
    }
    else
    {
        // Roll back if an error occurred
        DB.rollBack(onTransactionRolledBack);
    }
}

function onTransactionCommitted(error:Error):void
{
    if(error == null)
    {
        // Hooray
    }
}

function onTransactionRolledBack(error:Error):void
{
    if(error == null)
    {
        // Try again...
    }
}
```

## Running Multiple Queries

To execute multiple raw queries with a single method call, you can use the `multiQuery` method and pass in an `Array` of queries. This method is an exception to the errors-first rule regarding the callback parameters. Instead, a list of `BreezeQueryResult` is given where each result has the `error` property:

```as3
DB.multiQuery([
        "SELECT id, title FROM photos",
        "UPDATE photos SET title = :title WHERE id = :id"
    ],
    [null, { title: "Mountains", id: 1 }], // Array of query parameters
    function(results:Vector.<BreezeQueryResult>):void
    {
        // "results" list will contain 2 results
        for each(var result:BreezeQueryResult in results)
        {
            trace(result.error);
            trace(result.data);
            trace(result.rowsAffected);
        }
    });
```

Note that the values for query parameters is provided as an `Array` where each element corresponds to the query at the same index. It is also worth noting that the `multiQuery` method executes all the queries, even if some of them fail.

If you would like the execution to stop after a query fails, use the `multiQueryFailOnError` method. In the example below, the third query will **not** be executed due to the syntax error in the second query:

```as3
DB.multiQueryFailOnError([
        "SELECT id, title FROM photos",
        "DROP TABLEz photos", // forced syntax error
        "UPDATE photos SET title = :title WHERE id = :id" // will not be executed
    ],
    [null, null, { title: "Mountains", id: 1 }],
    function(error:Error, results:Vector.<BreezeQueryResult>):void
    {
        // "error" will refer to the SQL syntax error encountered in the second query
        // "results" list will contain only 2 results, for the first and second query
    });
```

Be aware that even if a query fails, any changes made by the preceding queries will take effect and will not be rolled back. Also note the difference in the callback signature where the first parameter is of type `Error`. This will either be `null` or hold a reference to the error that caused the execution to stop.

Finally there is also the `multiQueryTransaction` method which wraps the execution of all queries within a transaction and if any of them fails the changes are not committed. This method provides a simplified way of making atomic commits than when controlling the [transaction manually](#transactions):

```as3
DB.multiQueryTransaction([
        "UPDATE photos SET title = :title WHERE id = :id",
        "DROP TABLEz photos", // forced syntax error
        "SELECT id, title FROM photos" // will not be executed
    ],
    [{ title: "Mountains", id: 1}],
    function(error:Error, results:Vector.<BreezeQueryResult>):void
    {
        // "error" will refer to the SQL syntax error encountered in the second query
        // "results" list will contain only 2 results, for the first and second query
    });
```

In the example above, the `UPDATE` made by the first query will **not** take effect because the second query fails with an error.

The `multiQuery` methods are not limited to raw queries only. It is often more convenient to use the `BreezeQueryBuilder` to create multiple queries and have them all executed later. To be able to do so, the query builder must be "delayed" by using the `BreezeDb.DELAY` constant in place of a callback. Then simply feed all the query builders to one of the `multiQuery` methods.

> All `BreezeQueryBuilder` objects passed to one of the `multiQuery` methods must use the same database connection.

```as3
var query1:BreezeQueryBuilder = DB.table("photos").where("id", 1).update({name: "Sunset"}, BreezeDb.DELAY);
var query2:BreezeQueryBuilder = DB.table("photos").where("id", 2).delete(BreezeDb.DELAY);
var query3:BreezeQueryBuilder = DB.table("photos").where("id", 3").update({name: "Beach"}, BreezeDb.DELAY);

DB.multiQuery([query1, query2, query3],
    function(results:Vector.<BreezeQueryResult>):void
    {
        for each(var result:BreezeQueryResult in results)
        {
            trace(result.error);
            trace(result.data);
            trace(result.rowsAffected);
            trace(result.lastInsertRowID);
        }
    }
);
```