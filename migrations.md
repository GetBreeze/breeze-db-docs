# Migrations

<!-- TOC -->

- [Introduction](#introduction)
- [Migration Structure](#migration-structure)
- [Running Migrations](#running-migrations)

<!-- /TOC -->

## Introduction

Migrations allow you to create and edit your database structure between app versions. Note that migrations are forward-only, i.e. they **cannot** be rolled back once completed.

Migrations are atomic and run within a transaction, thus if one migration fails then all preceding migrations in the same session will be rolled back. After a migration is completed successfully, it will never run again.

## Migration Structure

Migrations are simple classes that run custom database operations, typically making use of the schema builder to create and edit database tables. You can create your own migration by subclassing the `BreezeMigration` class and overriding the `run` method. Use the provided `db` parameter to access the database reference where the migration is running:

```as3
public class CustomMigration extends BreezeMigration
{
    override public function run(db:IBreezeDatabase):void
    {
        // Do NOT call super.run(db)!

        // Add migration code here

        // Call the protected 'done' method to indicate the migration has finished
        done();
    }
}
```

If a migration fails, the `done` method should be called with `false` as the only argument:

```as3
override public function run(db:IBreezeDatabase):void
{
    db.schema.dropTable("photos", function(error:Error):void
    {
        // The migration will fail if there is an error
        done(error == null);
    });
}
```

Note that the name of the migration class is very important and you will not want to use generic names like `CustomMigration`. After a migration is completed successfully, its class name is stored in the database to make sure it is never run again. Therefore you should try to name your migration classes based on date and a short description of what it does, for example:

```as3
public class Migration_2017_06_25_Create_Photos_Table extends BreezeMigration
{
    override public function run(db:IBreezeDatabase):void
    {
        db.schema.createTable("photos", function (table:TableBlueprint):void
        {
            table.increments("id");
            table.string("title").defaultNull();
            table.integer("views").defaultTo(0);
            table.integer("downloads").defaultTo(0);
            table.integer("likes").defaultTo(0);
            table.date("creation_date");
        }, onTableCreated);
    }

    private function onTableCreated(error:Error):void
    {
        done(error == null);
    }
}
```

Of course, migrations are not limited to using the schema builder only. You can also update data within tables using the query builder:

```as3
public class Migration_2017_06_25_Insert_Default_Photos extends BreezeMigration
{

	private const PHOTOS:Array = [
		{ title: "Mountains",   views: 35,  downloads: 10,  likes: 4,  creation_date: new Date(2014, 1, 25) },
		{ title: "Flowers",     views: 6,   downloads: 6,   likes: 6,  creation_date: new Date(2015, 3, 3) },
		{ title: "Lake",        views: 35,  downloads: 0,   likes: 0,  creation_date: new Date(2016, 5, 19) }
	];

	override public function run(db:IBreezeDatabase):void
	{
		db.table("photos").insert(PHOTOS, function(error:Error):void
		{
			done(error == null);
		});
	}
}
```

## Running Migrations

BreezeDb allows you to run migrations either during a database setup or after setup at any time.

To run migrations during your database setup, set the `migrations` property before calling the `setup` method. All given migrations will run before the setup callback is triggered. If any migration fails, the database will not be set up. You can set either one or multiple migration classes:

```as3
DB.migrations = [Migration_2017_06_25_Create_Table_Photos, Migration_2017_06_25_Insert_Default_Photos];
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
DB.runMigrations(Migration_2017_06_25_Create_Photos_Table, function(error:Error):void
{
    if(error == null)
    {
        // The migration has run successfully
    }
});
```