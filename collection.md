# Collection

<!-- TOC -->

- [Introduction](#introduction)
- [Creating Collections](#creating-collections)
- [Available Methods](#available-methods)
    - [all](#all)
    - [add](#add)
    - [avg, sum, min, max](#avg-sum-min-max)
    - [contains](#contains)
    - [first](#first)
    - [last](#last)
    - [get](#get)
    - [has](#has)
    - [isEmpty](#isempty)
    - [pluck](#pluck)
    - [prepend](#prepend)
    - [pull](#pull)
    - [reduce](#reduce)
    - [search](#search)
    - [unique](#unique)
    - [where](#where)
    - [whereStrict](#wherestrict)
    - [whereIn](#wherein)
    - [whereInStrict](#whereinstrict)

<!-- /TOC -->

## Introduction

The `Collection` class provides a wrapper for working with arrays of data. It is built upon the standard `Array` class and provides a handful of new methods.

## Creating Collections

The `Collection` is typically returned as a result of `SELECT` queries. Of course, you may create your own instance if you wish. You can pre-populate the collection by passing the elements to the constructor:

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11, 13, 17);
```

To convert an existing `Array` to `Collection`, use the static `fromArray` method:

```as3
var primes:Collection = Collection.fromArray( [2, 3, 5, 7, 11, 13, 17] );
```

## Available Methods

This document is an overview of the additional `Collection` methods. Some of the methods have a return type of `Collection` allowing you to chain other methods to fluently manipulate the underlying `Array`. Such methods usually return a completely new `Collection` instance, keeping the original `Collection` intact.

### all

The `all` getter returns the underlying `Array` represented by the collection:

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11, 13, 17);

primes.all; // [2, 3, 5, 7, 11, 13, 17]
```

### add

Adds a single element to the end of the collection. If inserting a single element then this method is preferred to the `push` method where new allocation is made due to the `rest` parameter.  New `Collection` is **not** created, the original instance is modified instead.

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);

primes.add(13).add(17);
primes.all; [2, 3, 5, 7, 11, 13, 17];
```

### avg, sum, min, max

The `avg` method returns the average of all items in the collection:

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);

primes.avg(); // 5.6
```

If the collection contains objects, you should provide a key that will be used to retrieve the value to be averaged:

```as3
var devices:Collection = new Collection({ name: "iPhone 6", price: 549 }, { name: "Galaxy S6", price: 399 });

devices.avg("price"); // 474
```

You can also provide function that retrieves the value that is to be averaged:

```as3
var devices:Collection = new Collection({ name: "iPhone 6", price: 549 }, { name: "Galaxy S6", price: 399 });

devices.avg(function(device:Object):Number
{
   return device.price;
}); // 474
```

The usage is exactly the same for the other aggregate methods.

```as3
// Sum all the numbers in the collection
var primes:Collection = new Collection(2, 3, 5, 7, 11);
primes.sum(); // 28
```

Using the `min` method to find the lowest price:

```as3
var devices:Collection = new Collection({ name: "iPhone 6", price: 549 }, { name: "Galaxy S6", price: 399 });

devices.min("price"); // 399
```

### contains

The `contains` method returns `true` if the collection contains the given item. Strict equality operator (`===`) is used for comparison.

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);

primes.contains(5); // true
primes.contains("5"); // false
primes.contains(4); // false
```

If the collection contains objects, you may also provide a key as the second argument:

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

// Is there an object with the name "iPhone 6"?
devices.contains("iPhone 6", "name"); // true

devices.contains("iPhone 6"); // false
```

### first

The `first` method returns the first item in the collection that passes the given test. If the truth test is not provided then the first item is returned. If the collection is empty then `null` is returned.

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);
primes.first(); // 2

// Find the first prime greater than 5
primes.first(function(n:Number):Boolean
{
    return n > 5;
}); // 7

var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S8",   brand: "Samsung", price: 599 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

// Find first non-Apple device with price below 500
devices.first(function(device:Object):Boolean
{
    return device.price < 500 && device.brand != "Apple";
}); // { name: "Galaxy S6", brand: "Samsung", price: 399 }
```

### last

The `last` method works the opposite way and returns the last item in the collection that passes the given test.

```as3
var numbers:Collection = new Collection(-3, -2, -1, 0, 1, 2, 3, 4, 5);

// Find the last number that is less than zero
numbers.last(function(n:Number):Boolean {
   return n < 0;
}); // -1
```

### get

The `get` method returns the value at a given key, or `null` if the key does not exist.

```as3
var deviceToPrice:Collection = new Collection(
    { "iPhone 6": 549 },
    { "iPhone SE": 399 },
    { "Galaxy S6": 399 }
);

deviceToPrice.get("iPhone SE"); // 399
```

You can also provide a default value that will be returned if the key is not found:

```as3
var deviceToPrice:Collection = new Collection(
    { "iPhone 6": 549 },
    { "iPhone SE": 399 },
    { "Galaxy S6": 399 }
);

deviceToPrice.get("iPad Pro", 0); // 0
```

The default value argument can be a `Function` that returns the value. If the searched key does not exist, the value returned by the function will be returned by the `get` method:

```as3
var deviceToPrice:Collection = new Collection(
    { "iPhone 6": 549 },
    { "iPhone SE": 399 },
    { "Galaxy S6": 399 }
);

deviceToPrice.get("iPad Pro", function():Number
{
    return -1;
}); // -1
```

### has

The `has` method returns `true` if any item in the collection has the given key:

```as3
var deviceToPrice:Collection = new Collection(
    { "iPhone 6": 549 },
    { "iPhone SE": 399 },
    { "Galaxy S6": 399 }
);

deviceToPrice.has("iPhone 6"); // true
deviceToPrice.has("iPad Pro"); // false
```

### isEmpty

The `isEmpty` getter returns `true` if the collection has no items, `false` otherwise.

```as3
var empty:Collection = new Collection();
empty.isEmpty; // true

var primes:Collection = new Collection(2, 3, 5, 7, 11);
primes.isEmpty; // false
```

### pluck

The `pluck` method returns all values for the given key. The original collection is not modified, new `Collection` instance is returned instead.

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

devices.pluck("name").all; // ["iPhone 6", "iPhone SE", "Galaxy S6"]
```

The second parameter specifies how the resulting collection is keyed:

```as3
devices.pluck("price", "name").all; // [{"iPhone 6": 549}, {"iPhone SE": 399}, {"Galaxy S6": 399}]
```

### prepend

The `prepend` method adds an item to the beginning of the collection. If inserting a single element then this method is preferred to the `unshift` method where new allocation is made due to the `rest` parameter. New `Collection` is **not** created, the original instance is modified instead.

```as3
var primes:Collection = new Collection(3, 5, 7, 11);
primes.prepend(2);
primes.all; // [2, 3, 5, 7, 11]
```

### pull

The `pull` method removes and returns an item from the collection by its key.

```as3
var deviceToPrice:Collection = new Collection(
    { "iPhone 6": 549 },
    { "iPhone SE": 399 },
    { "Galaxy S6": 399 }
);

deviceToPrice.pull("iPhone 6"); // { "iPhone 6": 549 }
deviceToPrice.all; // [{ "iPhone SE": 399 }, { "Galaxy S6": 399 }]
```

### reduce

The `reduce` method reduces the collection to a single value, passing the result of each iteration into the subsequent iteration.

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);

primes.reduce(function(carry:*, element:*):*
{
    return carry + element;
}); // 28
```

The value for `carry` on the first iteration is `null`, but you can specify custom inital value by passing a second argument to `reduce`:

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);
primes.reduce(function(carry:*, element:*):*
{
    return carry + element;
}, 10); // 38
```

### search

Similar to the built-in `indexOf` method, this method can be used to find the index for the given value. In addition, the first argument can be a `Function` that is used as a truth test.

```as3
var companies:Collection = new Collection(
   { name: "Microsoft", ceo: "Satya Nadella" },
   { name: "Apple",     ceo: "Tim Cook" },
   { name: "Google",    ceo: "Sundar Pichai" }
);

companies.search(truthTest); // 1

function truthTest(company:Object):Boolean {
   return company.name == "Apple";
};
```

To use strict equality operator (`===`) for comparison, pass in `true` as the second argument.

```as3
var primes:Collection = new Collection(2, 3, 5, 7, 11);
primes.search("5"); // 2
primes.search("5", true); // -1
```

### unique

The `unique` method returns all of the unique items in the collection. The original collection is not modified, new `Collection` instance is returned instead.

```as3
var numbers:Collection = new Collection(1, 1, 3, 4, 6, 6, 7, 8);
var unique:Collection = numbers.unique();
unique.all; // [1, 3, 4, 6, 7, 8]
```

If the collection contains objects, you should pass a key to use for determining the uniqueness:

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy Gear", brand: "Samsung", price: 199 }
);

devices.unique("brand"); // [{ name: "iPhone 6", brand: "Apple", price: 549 }, { name: "Galaxy Gear", brand: "Samsung", price: 199 }]
```

You may also pass in a callback that returns the value to use for determining uniqueness:

```as3
var devices:Collection = new Collection(
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "iPad Mini",   brand: "Apple",   price: 399 },
    { name: "Galaxy Gear", brand: "Samsung", price: 199 }
);

devices.unique(uniqueBrandPrice); // [{ name: "iPhone SE", brand: "Apple", price: 399 }, { name: "Galaxy Gear", brand: "Samsung", price: 199 }]
function uniqueBrandPrice(device:Object):Boolean {
   // Concatenated brand and price to be used for determining uniqueness
   return device.brand + device.price;
};
```

### where

The `where` method filters the collection by the given key / value pair. Uses equality operator (`==`) to compare item values. The original collection is not modified, new `Collection` instance is returned instead.

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

var appleDevices:Collection = devices.where("brand", "Apple");
trace(appleDevices.all); // [{ name: "iPhone 6", brand: "Apple", price: 549 }, { name: "iPhone SE", brand: "Apple", price: 399 }]
```

### whereStrict

The `whereStrict` method works the same way as the `where` method, except that strict equality operator (`===`) is used to compare item values.

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

var devicesFor399:Collection = devices.whereStrict("price", 399);
trace(devicesFor399.all); // [{ name: "iPhone SE", brand: "Apple", price: 399 }, { name: "Galaxy S6", brand: "Samsung", price: 399 }]

// No match for "price" of type String
devicesFor399 = devices.whereStrict("price", "399");
trace(devicesFor399.all); // []
```

### whereIn

The `whereIn` method filters the collection by the given key / value contained within the given `Array`. Uses equality operator (`==`) to compare item values. The original collection is not modified, new `Collection` instance is returned instead.

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

var filtered:Collection = devices.whereIn("price", [199, 549]);
filtered.all; // [{ name: "iPhone 6", brand: "Apple", price: 549 }]
```

### whereInStrict

The `whereInStrict` method works the same way as the `whereIn` method, except that strict equality operator (`===`) is used to compare item values.

```as3
var devices:Collection = new Collection(
    { name: "iPhone 6",    brand: "Apple",   price: 549 },
    { name: "iPhone SE",   brand: "Apple",   price: 399 },
    { name: "Galaxy S6",   brand: "Samsung", price: 399 }
);

var filtered:Collection = devices.whereInStrict("price", [399, "549"]);
filtered.all; // [{ name: "iPhone SE", brand: "Apple", price: 399 }, { name: "Galaxy S6", brand: "Samsung", price: 399 }]
```