# Query Builder

<!-- TOC depthTo:3 -->

- [Introduction](#introduction)
- [Retrieving Results](#retrieving-results)
    - [Retrieving All Rows](#retrieving-all-rows)
    - [Retrieving the First Row](#retrieving-the-first-row)
    - [Chunking Results](#chunking-results)
    - [Aggregates](#aggregates)
- [Delaying Query Execution](#delaying-query-execution)
- [Selects](#selects)
- [Joins](#joins)
    - [Inner Join](#inner-join)
    - [Left Join](#left-join)
    - [Cross Join](#cross-join)
- [Unions](#unions)
- [Where Clauses](#where-clauses)
    - [Simple Where Clauses](#simple-where-clauses)
    - [Additional Where Clauses](#additional-where-clauses)
    - [Parameter Grouping](#parameter-grouping)
- [Ordering, Grouping, Limiting and Offset](#ordering-grouping-limiting-and-offset)
- [Inserts](#inserts)
- [Updates](#updates)
- [Deletes](#deletes)

<!-- /TOC -->

## Introduction

Query builder provides a convenient, fluent interface for creating and running database queries. It can be used to perform most database operations in your application. The builder also uses automatic parameter binding to protect your application against SQL injection attacks.

BreezeDb also supports query queuing that guarantees the queries are executed one after another, in the order they are created. [Delayed queries](#delaying-query-execution) are added to the queue only when manually executed. The queue is disabled by default and can be enabled by setting the value of `isQueryQueueEnabled` to `true`:

```as3
BreezeDb.isQueryQueueEnabled = true;
```

> Make sure there are no queries running when changing this option.

## Retrieving Results

All queries are asynchronous, maximizing the responsiveness of your app. That also means the query result is not immediately available. Thus, in order to access the result data, you need to provide a callback method.

The callback can specify up to 2 parameters, and no parameters are allowed as well. If you do specify parameters, the first one must always be of type `Error` - each query returns an error as the first argument, or `null` if no error has occurred. The data type of the second parameter depends on the type of query it corresponds to. See the documentation for each query to learn how to format your callbacks.

> Callback is always optional and the query will be executed even if it is not set, unless explicitly told not to (see [Delaying Query Execution](#delaying-query-execution)).

### Retrieving All Rows

A query builder is always associated with a database and a table within the database. You can create an instance of `BreezeQueryBuilder` manually by passing a reference to your database and a table name to the builder's constructor. It is however more convenient to use the `table` method on the `DB` facade class or on a database instance. The returned query builder instance is already associated with the database and the given table. Having access to query builder instance, you can now chain constraints onto the query and then finally get the results using the `fetch` method:

```as3
// Query the table "photos" in the default database
var builder:BreezeQueryBuilder = DB.table("photos");

// Same as above
var builder:BreezeQueryBuilder = BreezeDb.db.table("photos");

// Retrieve all the photos in the table
builder.fetch(onSelectCompleted);

function onSelectCompleted(error:Error, photos:Collection):void
{
    if(error == null)
    {
        // process the photos
    }
}
```

The `Collection` object contains the results where each result is a generic `Object` instance, representing a single table row. You may access each column's value by using the column name as a property of the object:

```as3
for each(var photo:Object in photos)
{
    trace(photo.id, photo.title); // Here "id" and "title" are columns in the "photos" table
}
```

### Retrieving the First Row

If you need to retrieve only a single row from the database table, you may use the `first` method. This method returns a single `Object` instead of a `Collection`:

```as3
DB.table("photos").first(function(error:Error, firstPhoto:Object):void
{
    if(error == null)
    {
        trace(photo.id, photo.title);
    }
});
```

### Chunking Results

If you work with thousands of database records, consider using the `chunk` method. This method retrieves a small chunk of the results at a time and feeds each chunk into your callback `Function` for processing. For example, let's work with the entire `photos` table in chunks of 100 records at a time:


```as3
DB.table("photos").orderBy("id").chunk(100, function(error:Error, photos:Collection):*
{
    if(error == null)
    {
        // "photos" Collection will have up to 100 records
    }
});
```

You may stop further chunks from being processed by returning `false` from the callback:

```as3
var numChunks:int = 0;

DB.table("photos").orderBy("id").chunk(100, function(error:Error, photos:Collection):*
{
    if(error == null)
    {
        numChunks++;

        if(numChunks > 2)
        {
            return false; // The callback will not be triggered again
        }
    }
});
```

### Aggregates

The query builder also provides a variety of aggregate methods such as `count`, `max`, `min`, `avg`, and `sum`. You may call any of these methods after constructing your query. Your callback should specify a second parameter of type `Number`:

```as3
DB.table("photos").count(function(error:Error, count:Number):void
{
    if(error == null)
    {
        trace("Number of photos =", count);
    }
});
```

Except for the `count` method, all aggregate methods accept a column name on which the aggregate should be calculated:

```as3
// Sum the number of views
DB.table("photos").sum("views", function(error:Error, views:Number):void
{
    if(error == null)
    {
        trace("Total number of views =", views);
    }
});

// Get the average number of views
DB.table("photos").avg("views", function(error:Error, views:Number):void
{
    if(error == null)
    {
        trace("Average number of views =", views);
    }
});

// Find the highest number of downloads
DB.table("photos").max("downloads", function(error:Error, downloads:Number):void
{
    if(error == null)
    {
        trace("Highest number of downloads =", downloads);
    }
});

// Find the lowest number of downloads
DB.table("photos").min("downloads", function(error:Error, downloads:Number):void
{
    if(error == null)
    {
        trace("Lowest number of downloads =", downloads);
    }
});
```

## Delaying Query Execution

Sometimes you may wish to build a query but do not want it to actually run, for example when you need to process multiple SQL statements in a row. In this case you can create the query as usual, however you pass in a value of `false` for the callback method.

```as3
var builder:BreezeQueryBuilder = DB.table("photos").remove(false);
```

You can use the available constant `BreezeDb.DELAY` to make the code a little clearer to read.

```as3
var builder:BreezeQueryBuilder = DB.table("photos").remove(BreezeDb.DELAY);
```

At this point, you can access the `queryString` property to see the resulting SQL query:

```as3
trace(builder.queryString); // DELETE FROM photos
```

To execute the delayed query, simply call the `exec` method with a callback as an argument. The callback's signature depends on the type of query that is executed:

```as3
builder.exec(function(error:Error, rowsDeleted:int):void
{
    if(error == null)
    {
        trace("Deleted", rowsDeleted, "row(s)");
    }
});
```

After a query is executed, you can use the `queryReference` property to access `BreezeQueryReference` object. It can be used to cancel the callback if needed. Note that doing so **does not stop** the actual SQL query from running, it only stops the callback from being called:

```as3
var reference:BreezeQueryReference = builder.queryReference;
if(!reference.isCompleted)
{
    reference.cancel();
}
```

## Selects

Using the `select` method, you can specify a custom `SELECT` clause for the query:

```as3
DB.table("photos")
    .select("id", "title as photo_title")
    .fetch(function(error:Error, results:Collection):void
    {
        if(error == null)
        {
            // each result object will have only "id" and "photo_title" properties
        }
    });
```

You can also use raw expressions in `SELECT` queries:

```as3
DB.table("photos")
    .select("COUNT(*) as total_records", "creation_date")
    .groupBy("creation_date")
    .fetch(function(error:Error, results:Collection):void
    {
        if(error == null)
        {
            // "results" contains "total_records" and "creation_date" fields
        }
    });
```

The `distinct` method allows you to force the query to return distinct results:

```as3
DB.table("photos")
    .distinct("title")
    .fetch(function(error:Error, results:Collection):void
    {
        if(error == null)
        {
            // "results" contains only photos with distinct "title"
        }
    });
```

## Joins

The query builder provides API for all 3 types of joins supported in SQLite.

### Inner Join

To perform a basic "inner join", use the `join` method. The first argument is the name of the table you want to join, and the second argument specifies the column constraints for the join. You can also join multiple tables in a single query:

```as3
DB.table("users")
    .join("contacts", "users.id = contacts.user_id")
    .join("orders", "users.id = orders.user_id")
    .select("users.*", "contacts.phone", "orders.price")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

### Left Join

To perform "left outer join", use the `leftJoin` method. The usage is identical to the `join` method:

```as3
DB.table("photos")
    .leftJoin("authors", "photos.author_id = authors.id")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

### Cross Join

To perform a "cross join", use the `crossJoin` method with the name of the table you wish to join. Cross joins generate a cartesian product between the first table and the joined table. Be careful as cross joins have the potential to generate extremely large tables:

```as3
DB.table("sizes")
    .crossJoin("colors")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

## Unions

The query builder also provides a quick way to "union" multiple queries together. For example, you may create an initial query and use the union method to union it with a second query. Note that duplicate rows are removed from the final result.

```as3
// Select photo with id = 1
var query1:BreezeQueryBuilder = DB.table("photos")
    .select("id", "title", "views", "downloads")
    .where("id", 1);

// Select two photos with id = 1 and id = 2
var query2:BreezeQueryBuilder = DB.table("photos")
    .select("id", "title", "views", "downloads")
    .where("id", "<", 3);

// Union the two queries
query1
    .union(query2)
    .fetch(function(error:Error, results:Collection):void
    {
        if(error == null)
        {
            // "results" contains 2 objects (duplicate rows are not included)
        }
    });
```

> You can only union queries that use the same database.

To union queries while keeping duplicate rows, use the `unionAll` method:

```as3
// Select photo with id = 1
var query1:BreezeQueryBuilder = DB.table("photos")
    .select("id", "title", "views", "downloads")
    .where("id", 1);

// Select two photos with id = 1 and id = 2
var query2:BreezeQueryBuilder = DB.table("photos")
    .select("id", "title", "views", "downloads")
    .where("id", "<", 3);

// Union the two queries and keep duplicate records
query1
    .unionAll(query2)
    .fetch(function(error:Error, results:Collection):void
    {
        if(error == null)
        {
            // "results" contains 3 objects (photo with id of 1 is included twice)
        }
    });
```

## Where Clauses

### Simple Where Clauses

You may use the `where` method to add `WHERE` clauses to the query. The most basic call to `where` requires three arguments. The first argument is the name of the column. The second argument is an operator, which can be any of the database's supported operators. Finally, the third argument is the value to evaluate against the column.

To select photos with exactly 100 views, you could use the following query:

```as3
DB.table("photos")
    .where("views", "=", 100)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

For convenience, if you simply want to verify that a column is equal to a given value, you may pass the value directly as the second argument to the `where` method:

```as3
DB.table("photos")
    .where("views", 100)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

You may use a variety of other operators with the `where` clause:

```as3
DB.table("photos")
    .where("views", ">", 100)
    .fetch(function(error:Error, results:Collection):void
    {

    });

DB.table("photos")
    .where("views", "!=", 100)
    .fetch(function(error:Error, results:Collection):void
    {

    });

DB.table("photos")
    .where("title", "LIKE", "Summer%")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

You can also chain multiple `where` calls. These conditions are joined using the `AND` operator:

```as3
// SELECT * FROM photos WHERE id > 10 AND id != 20;
DB.table("photos")
    .where("id", ">", 10)
    .where("id", "!=", 20)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

Alternatively, you can pass in an `Array` of conditions:

```as3
// SELECT * FROM photos WHERE id > 10 AND id != 20;
DB.table("photos")
    .where([["id", ">", 10], ["id", "!=", 20]])
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

To add `WHERE` clauses joined using the `OR` operator, use the `orWhere` method:

```as3
// SELECT * FROM photos WHERE id = 1 OR id = 5;
DB.table("photos")
    .where("id", 1)
    .orWhere("id", 2)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

### Additional Where Clauses

#### whereBetween / whereNotBetween

The `whereBetween` method selects records where a column's value is between two values (including):

```as3
DB.table("photos")
    .whereBetween("views", 10, 30)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereNotBetween` method selects records where a column's value lies outside of two values:

```as3
DB.table("photos")
    .whereNotBetween("views", 10, 30)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

#### whereIn / whereNotIn

The `whereIn` method selects records where a column's value is contained within the given array:

```as3
DB.table("photos")
    .whereIn("downloads", [10, 20, 30])
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereNotIn` method selects records where a column's value is **not** contained within the given array:

```as3
DB.table("photos")
    .whereNotIn("downloads", [10, 20, 30])
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

#### whereNull / whereNotNull

The `whereNull` method selects records where a column's value is `NULL`:

```as3
DB.table("photos")
    .whereNull("title")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereNotNull` method selects records where a column's value is **not** `NULL`:

```as3
// Verifies that a column's value is NOT NULL
DB.table("photos")
    .whereNotNull("title")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```


#### whereDate / whereMonth / whereDay / whereYear

The `whereDate` method will compare a date column and a date value in `YYYY-MM-DD` format:

```as3
DB.table("photos")
    .whereDate("creation_date", "2016-09-23")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

If you pass in three parameters, you can create a comparison between the dates. This is possible for all the other date methods listed below:

```as3
DB.table("photos")
    .whereDate("creation_date", ">", "2016-09-23")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereMonth` method compares a month of the year to a column value:

```as3
DB.table("photos")
    .whereMonth("creation_date", 1)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereDay` method compares a day of the month to a column value:

```as3
DB.table("photos")
    .whereDay("creation_date", 15)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereYear` method compares a column value to a specific year:

```as3
DB.table("photos")
    .whereYear("creation_date", 2014)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

You can also use the built-in `Date` object as the value for comparison in all the `where` methods related to date. The following selects all the photos created after the month of July (any year):

```as3
DB.table("photos")
    .whereMonth("creation_date", ">", new Date(2015, 6, 15))
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

#### whereColumn

You can compare column values by using the `whereColumn` method. The query below will select photos where number of views is the same as the number of downloads:

```as3
DB.table("photos")
    .whereColumn("views", "downloads")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `whereColumn` method works just like the `where` method so you can pass in an operator or an `Array` of conditions:

```as3
DB.table("photos")
    .whereColumn("views", ">", "downloads")
    .fetch(function(error:Error, results:Collection):void
    {

    });

// SELECT * FROM photos WHERE (title = description) AND (views > downloads)
DB.table("photos")
    .whereColumn([["title", "description"], ["views", ">", "downloads"]])
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

### Parameter Grouping

Sometimes you may need to create more advanced `WHERE` clauses with nested parameter groupings. To do that, you need to pass in a `Function` as the first argument to the `where` / `orWhere` method. The function must accept a single parameter of type `BreezeInnerQueryBuilder`. You can use the provided instance to set the constraints that should be contained within the parenthesis group.

```as3
// SELECT * FROM photos WHERE id > 15 OR (title = "Sunset" AND id != 18)
DB.table("photos")
    .where("id > 15")
    .orWhere(function(query:BreezeInnerQueryBuilder):void
    {
        query
            .where("title", "Sunset")
            .where("id", "!=", 18)
    })
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

## Ordering, Grouping, Limiting and Offset

The `orderBy` method allows you to sort the result of the query by one or multiple columns. The first argument to the `orderBy` method should be the column you wish to sort by, while the second argument controls the direction of the sort and may be either `ASC` or `DESC`:

```as3
DB.table("photos")
    .orderBy("views", "ASC", "downloads", "ASC")
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

The `groupBy` and `having` methods may be used to group the query results. The `having` method's signature is similar to that of the `where` method:

```as3
// SELECT SUM(views) as total_views, strftime('%Y', creation_date) as year_created
// GROUP BY year_created
// HAVING total_views >= 30 AND total_views <= 35
DB.table("photos")
    .select("SUM(views) as total_views, strftime('%Y', creation_date) as year_created")
    .groupBy("year_created")
    .having([["total_views", ">=", 30], ["total_views", "<=", 35]])
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

To limit the number of results returned from the query, or to skip a given number of results in the query, you may use the `limit` and `offset` methods. The following query returns only first 2 rows from the photos table:

```as3
DB.table("photos")
    .limit(2)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

While the following query skips the first 2 rows and selects the 3rd and 4th row:

```as3
DB.table("photos")
    .offset(2)
    .limit(2)
    .fetch(function(error:Error, results:Collection):void
    {

    });
```

## Inserts

You can insert a record by providing an object containing column names and values:

```as3
DB.table("photos")
    .insert({ id: 1, title: "Morning Dew", views: 3, downloads: 0, likes: 0, creation_date: new Date(2014, 1, 25) }),
    function(error:Error):void
    {
        if(error == null)
        {
            // insert was successful
        }
    });
```

You can also perform a multi-row insert by passing in an `Array` of objects, each representing a single row:

```as3
DB.table("photos")
    .insert([
        { id: 1, title: "Morning Dew", views: 3, downloads: 0, likes: 0, creation_date: new Date(2014, 1, 25) },
        { id: 2, title: "Sunset", views: 12, downloads: 4, likes: 5, creation_date: new Date(2014, 2, 26) },
    ]),
    function(error:Error):void
    {

    });
```

If the table has an auto-incrementing id, use the `insertGetId` method to insert a record and then retrieve the new ID in the callback:

```as3
DB.table("photos")
    .insertGetId({ title: "Night Sky", views: 0, downloads: 0, likes: 0, creation_date: new Date() }),
    function(error:Error, newId:int):void
    {

    });
```


## Updates

Updates can be performed using the `update` method and by passing in an `Object` containing the column names and values to be updated. You may constrain the `update` query using `where` clauses. The callback receives a number of rows affected by the query.

```as3
DB.table("photos")
    .where("id", 3)
    .update({ title: "Blue Sky" }), function(error:Error, rowsAffected:int):void
    {

    });
```

When you need to increment or decrement a column value by a specific amount, you do not need to write the `UPDATE` query manually. The query builder provides two convenience methods `increment` and `decrement` that simplify these operations.

Both of these methods accept at least one argument: the column to modify. A second argument may optionally be passed to control the amount by which the column should be incremented or decremented (it defaults to 1). If you want to use the default amount, you can pass in callback as the second argument. **Note the information about the number of rows affected is not available for these two queries.**

```as3
// Increment the views by 1 (default value if not explicitly set)
DB.table("photos")
    .where("id", 10)
    .increment("views",
    function(error:Error):void
    {

    });

// Increment the views by 5
DB.table("photos")
    .where("id", 10)
    .increment("views", 5,
    function(error:Error):void
    {

    });

// Decrement the downloads by 1 (default value if not explicitly set)
DB.table("photos")
    .where("id", 10)
    .decrement("downloads",
    function(error:Error):void
    {

    });

// Decrement the downloads by 5
DB.table("photos")
    .where("id", 10)
    .decrement("downloads", 5,
    function(error:Error):void
    {

    });
```

Additionally, you can also specify columns to update during the operation.

```as3
// Increment the views by 1 AND update title to "Sunset"
DB.table("photos")
    .where("id", 10)
    .increment("views", { title: "Sunset" },
    function(error:Error):void
    {

    });

// Increment the views by 5 AND update title to "Sunset"
DB.table("photos")
    .where("id", 10)
    .increment("views", 5, { title: "Sunset" },
    function(error:Error):void
    {

    });
```


## Deletes

Deletes can be performed using the `remove` method.

> Remember that this method is ```remove``` and not ```delete```. The word `delete` is a reserved word in ActionScript and cannot be used as a method name.

You may constrain `DELETE` statements by adding `WHERE` clauses before calling the `remove` method. The callback receives a number of deleted rows:

```as3
DB.table("photos")
    .where("id", 6)
    .remove(function(error:Error, rowsDeleted:int):void
    {

    })
```