# ORM

<!-- TOC -->

- [Introduction](#introduction)
- [Defining Models](#defining-models)
    - [Default Values](#default-values)
    - [Table Name](#table-name)
    - [Database Name](#database-name)
    - [Primary Key](#primary-key)
- [Querying Models](#querying-models)
    - [Retrieving Models by Primary Key](#retrieving-models-by-primary-key)
    - [Aggregates](#aggregates)
- [Inserts](#inserts)
- [Updates](#updates)
    - [Mass Updates](#mass-updates)
- [Other Creation Methods](#other-creation-methods)
    - [First or New](#first-or-new)
    - [First or Create](#first-or-create)
- [Deleting Models](#deleting-models)
    - [Deleting Models by Key](#deleting-models-by-key)
    - [Mass Deletes](#mass-deletes)

<!-- /TOC -->

## Introduction

The ORM support in BreezeDb allows you to seamlessly map each database table to an ActionScript class (model). Models give you the ability to query for data in your tables, as well as insert new records into the table using strongly typed objects.

## Defining Models

You can define model for all or only some of your tables as you see fit. In any case, a model must subclass the base `BreezeModel` class:

```as3
public class Photo extends BreezeModel
{
    public var id:int;
    public var title:String;
    public var views:int;
    public var downloads:int;
    public var creation_date:Date;
}
```

All public variables represent the respective columns in the table.

### Default Values

Although you may specify default values in the database schema, the default values that you provide in the model class will override any in the schema:

```as3
public class Photo extends BreezeModel
{
    public var id:int;
    public var title:String = "Default Title";
    ...
}
```

It is also possible to initialize the model by passing in a key-value `Object` to the `super()` call:

```as3
public class Photo extends BreezeModel
{
    public var id:int;
    public var title:String;
    ...

    public function Photo()
    {
        super({ title: "Default Title" });
    }
}
```

### Table Name

The name of the table corresponding to the model is automatically derived from the model's class name using "snake case" and plural form.

| Class name   | Table name     |
|--------------|----------------|
| `Photo`      | `photos`       |
| `PhotoAlbum` | `photo_albums` |

You can also explicitly specify the table name if you wish by setting the protected `_tableName` variable in the model's constructor:

```as3
public class Photo extends BreezeModel
{
    ...

    public function Photo()
    {
        super();

        _tableName = "my_photos";
    }
}
```

### Database Name

If not explicitly set, each model uses the default BreezeDb database. If you are using a custom database, you need to manually set the `_databaseName` variable in the model's constructor:

```as3
BreezeDb.getDb("my_database").setup(...);

...

public class Photo extends BreezeModel
{
    ...

    public function Photo()
    {
        super();

        _databaseName = "my_database";
    }
}
```

### Primary Key

By default each table is assumed to have a primary key column named `id`. You can change the primary key name using the `_primaryKey` variable. You can also pass a value of `null` if there is no primary key in the table:

```as3
public class Photo extends BreezeModel
{
    public var title:String;
    ...

    public function Photo()
    {
        super();

        // 'title' is now the primary key
        _primaryKey = "title";
    }

}
```

By default the primary key is considered to be auto-incrementing. You can change this by setting the `_autoIncrementId` variable to `false`:

```as3
public class Photo extends BreezeModel
{
    ...

    public function Photo()
    {
        super();

        _autoIncrementId = false;
    }

}
```

## Querying Models

Models can be queried using the `BreezeModelQueryBuilder` class. It is a subclass of `BreezeQueryBuilder` which means you get access to all the existing query methods plus some more. Additionally, the model builder automatically casts the results to your custom model's class so you get to work with strongly typed objects.

To start querying your custom model, you need to obtain a reference to an instance of `BreezeModelQueryBuilder` that is associated with your model's class. You can either create the instance manually by passing your model's class to the constructor of the model builder, or use the static `query` method provided by `BreezeModel`:

```as3
// 'Photo' is the class of our custom model
var builder:BreezeModelQueryBuilder = new BreezeModelQueryBuilder(Photo);

var builder:BreezeModelQueryBuilder = BreezeModel.query(Photo);
```

To further increase the readability of your code, you could create a static `query` method directly on your model's class:

```as3
public class Photo extends BreezeModel
{
    ...

    public static function query():BreezeModelQueryBuilder
    {
        return new BreezeModelQueryBuilder(Photo);
    }

}

...

var builder:BreezeModelQueryBuilder = Photo.query;
```

After you have a `BreezeModelQueryBuilder` reference you can query it just like you would with the `BreezeQueryBuilder`:

```as3
Photo.query.first(function(error:Error, first:Photo):void
{
    if(first != null)
    {
        trace(first.title);
    }
});
```

As you can see, using the model builder gives you the ability to type the callback parameters to your model's class and not a generic `Object`.

This also applies to queries where a `Collection` is returned. Each object within the collection is an instance of your model:

```as3
Photo.query
    .select("id", "title")
    .where("id", "<", 10)
    .fetch(function(error:Error, photos:Collection):void
    {
        if(error == null)
        {
            for each(var photo:Photo in photos)
            {
                trace(photo.id, photo.title);
            }
        }
    })
```

### Retrieving Models by Primary Key

The model builder has an additional method that allows you to retrieve models by primary key. This functionality is provided by the `find` method:

```as3
// Find photo with id of 2
Photo.query.find(2, function(error:Error, photo:Photo):void
{
    if(photo != null)
    {
        trace(photo.id); // 2
    }
});
```

You may also pass in an `Array` of primary keys to retrieve multiple models:

```as3
// Find photos with id of 1, 2 and 3
Photo.query.find([1, 2, 3], function(error:Error, photos:Collection):void
{
    if(error == null)
    {
        for each(var photo:Photo in photos)
        {
            trace(photo.id);
        }
    }
});
```

### Aggregates

All aggregate functions provided by query builder are also available to use with your models:

```as3
// Return number of photos that have more than 100 views
Photo.query.where("views", ">", 100).count(function(error:Error, count:int):void
{
    trace(count);
});
```

## Inserts

To insert new records into your model's table, simply create an instance of the model, set its properties and call the `save` method. The callback method gives you access to the model that has just been saved:

```as3
var photo:Photo = new Photo();
photo.title = "Sunset";
photo.views = 10;
photo.downloads = 3;

photo.save(function(error:Error, savedPhoto:Photo):void
{

});
```

After an insert has been completed, the primary key (if applicable) will be set and the `exists` property will be `true`:

```as3
var photo:Photo = new Photo();
trace(photo.exists); // false

photo.save(function(error:Error, savedPhoto:Photo):void
{
    // savedPhoto.id is now set 
    trace(savedPhoto.exists); // true
});
```

## Updates

The `save` method can also be used to update an existing model. For example, you could use the `find` method to retrieve a specific model, update its properties and save it back to the database:

```as3
// Find photo with id of 1 and update its title
Photo.query.find(1, function(error:Error, photo:Photo):void
{
    // Model not found
    if(photo == null)
    {
        return;
    }
 
    photo.name = "Sunrise";
    photo.save(function(error:Error, savedPhoto:Photo):void
    {
        if(error == null)
        {
            // Photo has been updated
        }
    });
});
```

### Mass Updates

To update several models at once, use the `update` method, preferably with the `where` method:

```as3
// Reset downloads for photos with 100 or more views
Photo.query.where("views", ">", 100).update({ downloads: 0 }, function(error:Error, rowsAffected:int):void
{
    trace("Updated", rowsAffected, "photo(s)");
});
```

## Other Creation Methods

### First or New

The `firstOrNew` method gives you the ability to search the database for a model with given values, and if one is not found, retrieve a new instance with the values already set on the model. Note that if the model is not found then the returned instance has not been persisted to the database yet and you will need to call the `save` method manually to do that:

```as3
Photo.query.firstOrNew({ title: "Sunset" }, function(error:Error, photo:Photo):void
{
    if(error == null)
    {
        trace(photo.title); // Sunset
    }
});
```

### First or Create

The `firstOrCreate` method is very similar, except that the model is automatically saved into the database if it is not found during the lookup:

```as3
Photo.query.firstOrNew({ title: "Sunset" }, function(error:Error, photo:Photo):void
{
    if(error == null)
    {
        trace(photo.title); // Sunset
        trace(photo.exists); // true
    }
});
```

## Deleting Models

To delete a specific model, call the `remove` method on the model's instance:

```as3
Photo.query.find(1, function(error:Error, photo:Photo):void
{
    photo.remove(function(deleteError:Error):void
    {
        if(deleteError == null)
        {
            // Photo has been deleted
        } 
    });
});
```

### Deleting Models by Key

The `removeByKey` methods is a shortcut to the approach above. If you know the model's primary key, you can delete it directly without retrieving it first:

```as3
// Delete photo with id of 1
Photo.query.removeByKey(1, function(error:Error):void
{

});
```

You can also delete multiple models by providing an `Array` of primary keys:

```as3
// Delete photos with id of 1, 2 and 3
Photo.query.removeByKey([1, 2, 3], function(error:Error):void
{
    
});
```

### Mass Deletes

You may also use use the `remove` method to delete several models at once. Use it together with the `where` method to avoid deleting all the records in the table:

```as3
// Delete photos with more than 15 views
Photo.query.where("views", ">", 15).remove(function(error:Error):void
{
   
});
```