# Schema Builder

<!-- TOC -->

- [Introduction](#introduction)
- [Accessing the Builder](#accessing-the-builder)
- [Tables](#tables)
    - [Creating tables](#creating-tables)
    - [Checking Existing Schema](#checking-existing-schema)
    - [Renaming / Dropping Tables](#renaming--dropping-tables)
- [Columns](#columns)
    - [Adding Columns](#adding-columns)
    - [Column Types](#column-types)
    - [Column Constraints](#column-constraints)
    - [Multiple Primary Keys](#multiple-primary-keys)
- [Indexes](#indexes)
    - [Adding an Index](#adding-an-index)
- [Example](#example)

<!-- /TOC -->

## Introduction

Schema builder provides a convenient interface for creating and editing database tables without having to write raw SQL queries.

## Accessing the Builder

A schema builder is always associated with a database. You can create an instance of `BreezeSchemaBuilder` manually by passing your database reference to the builder's constructor. It is however more convenient to use the `schema` getter on the `DB` facade class or on a database instance. The returned schema builder instance is already associated with the database and ready to use.

```as3
// Schema builder for default database
var schema:BreezeSchemaBuilder = DB.schema;

// For custom database
var schema:BreezeSchemaBuilder = BreezeDb.getDb("database-name").schema;
```

Just like in the case of the `BreezeQueryBuilder`, all queries made by the schema builder are asynchronous. Thus, in order to access the results, you need to provide a callback method. The rule of "errors first" applies for schema callbacks as well. In most of the queries here it is the only parameter you should specify. See the documentation for each query to learn how to format your callbacks.

## Tables

### Creating tables

To create a table in your database, use the `createTable` method. This method accepts the table name, a function that creates the table structure, and a callback method.

```as3
DB.schema.createTable("photos", function(table:TableBlueprint):void
{
    table.increments('id');
},
function(error:Error):void
{
    if(error == null)
    {
        // table was created successfully
    }
});
```

The function used as the second argument is given an instance of `TableBlueprint`. This object provides a convenient API to create your table structure, including column names, their data type and constraints.

You can also delay execution of the query by passing in `BreezeDb.DELAY` instead of a callback method, and see the resulting SQL query:

```as3
DB.schema.createTable("photos", function(table:TableBlueprint):void
{
    table.increments('id');
},
BreezeDb.DELAY);
```

You can also grab a query reference in case you need to cancel the callback:

```as3
var reference:QueryReference = DB.schema.createTable("photos"), function(table:TableBlueprint):void
{
    table.increments('id');
},
callbackMethod).queryReference;

// Cancel the callback
reference.cancel();
```

### Checking Existing Schema

Sometimes you may wish to execute a query depending on the existing schema of a database or table. You can do this with the `hasTable` and `hasColumn` methods:

```as3
DB.schema.hasTable("photos", function(error:Error, hasTable:Boolean):void
{
    if(error == null)
    {
        trace(hasTable);
    }
});


DB.schema.hasColumn("photos", "id", function(error:Error, hasColumn:Boolean):void
{
    if(error == null)
    {
        trace(hasColumn);
    }
});
```

### Renaming / Dropping Tables

You can rename a table using the `renameTable` method. Just pass in the old and new values for the table name:

```as3
DB.schema.renameTable("photos", "new_photos", function(error:Error):void
{
    if(error == null)
    {
        // table renamed successfully
    }
});
```

To drop a table, use the `dropTable` method by passing in the table name:

```as3
DB.schema.dropTable("photos", function(error:Error):void
{
    if(error == null)
    {
        // table dropped successfully
    }
});
```

You can use the `dropTableIfExists` method if you do not wish to receive an error in case the table does not exist:

```as3
DB.schema.dropTableIfExists("photos", function(error:Error):void
{

});
```

## Columns

### Adding Columns

After a table has been created you can add new columns using the `editTable` method:

```as3
DB.schema.editTable("photos", function(table:TableBlueprint):void
{
    // Adds new TEXT column called "description"
    table.string("description");
},
function(error:Error):void
{

});
```

> SQLite does not allow you to modify a column once it has been created.

### Column Types

To create a column, you can use one of the available methods on the `TableBlueprint` object. The following is the list of available methods along with the corresponding SQL data type they create:

| Method                          | SQL Data Type   |
|---------------------------------|-----------------|
| `table.integer("views")`        | `INTEGER`       |
| `table.string("description")`   | `TEXT`          |
| `table.blob("data")`            | `BLOB`          |
| `table.boolean("active")`       | `INTEGER`       |
| `table.number("price")`         | `NUMERIC`       |
| `table.date("created_at")`      | `DATE`          |
| `table.timestamp("created_at")` | `DATETIME`      |

The `TableBlueprint` also provides a method called `increments` that creates the following column constraints:

```
INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL
```

### Column Constraints

Except for the `increments` method where no other constraints can be added, each method returns an object that implements `IColumnConstraint` interface. This interface allows you to put additional constraints on the column:

| Method               | Description                                     |
|----------------------|-------------------------------------------------|
| `notNull()`          | Prevents the column from having a `null` value  |
| `defaultTo(value:*)` | Adds a default value to the column              |
| `defaultNull()`      | Defaults the column value to `null`             |
| `unique()`           | Ensures that the column contains a unique value |
| `primary()`          | Designates the primary key                      |

For example:

```as3
table.increments("id");
table.string("title").notNull().defaultTo("Default title").unique();
table.integer("views").defaultTo(0);
```

### Multiple Primary Keys

If you wish to have multiple primary keys, you can use the `primary` method on the `TableBlueprint` instance and list the column names that should form the primary key. Note that you can only create primary keys in the `createTable` method since SQLite does not allow for column modification after a column is created.

```as3
DB.schema.createTable("photos", function(table:TableBlueprint):void
{
    table.integer("id");
    table.string("title");

    table.primary("id", "title");
},
function(error:Error):void
{

});
```

Alternatively, you can designate multiple columns as primary at the time of creation:

```as3
DB.schema.createTable("photos", function(table:TableBlueprint):void
{
    table.integer("id").primary();
    table.string("title").primary();
},
function(error:Error):void
{

});
```

## Indexes

### Adding an Index

You can create an index in both the `createTable` and `editTable` methods. Just pass in the name of the column that you wish to index:

```as3
table.index("column1");
```

By default, the name of the index will be the name of the column prefixed by `index_`. In the example above, the index name would be `index_column1`. Of course, you can pass in a custom index name as the second argument to the `index` method:

```as3
table.index("column1", "index_name");
```

You can also create a composite index by providing an `Array` of column names:

```as3
table.index(["column1", "column2"]);
```

Again, if an index name is not provided, it will default to `index_column1_column2`. You can specify custom index name even for a composite index:

```as3
table.index(["column1", "column2"], "index_name");
```

Indexes can also be unique. Similar to the `UNIQUE` constraint, such index prevents duplicate entries in the column or combination of columns on which there's an index. Just pass in `true` as the third argument to the `index` method:

```as3
table.index("column1", "index_name", true);
```

If needed, you can drop an index by passing its name to the `dropIndex` method:

```as3
table.dropIndex("index_name");
```

## Example

```as3
DB.schema.createTable("photos", function(table:TableBlueprint):void
{
    table.increments("id");
    table.string("name").notNull().default("Untitled");
    table.string("description");
    table.integer("views").default(0);

    table.index("name", "name_index", true);
},
function(error:Error):void
{

});
```

The example above will create and run the following SQL statements:

```
CREATE TABLE IF NOT EXISTS [photos]
(
    [id] INTEGER NOT NULL PRIMARY KEY,
    [name] TEXT NOT NULL DEFAULT ("Untitled")
    [views] INTEGER DEFAULT (0)
);

CREATE INDEX IF NOT EXISTS [name_index] ON [name];
```