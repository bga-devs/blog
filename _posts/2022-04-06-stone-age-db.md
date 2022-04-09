---
title: Getting the database out of the stone age
author: Guillaume Benny
date: 2022-04-06 8:00:00 -0400
categories: [Tips]
tags: [back,class,database]
pin: true
---

_See what I did there? [Stone
Age](https://boardgamearena.com/gamepanel?game=stoneage)? OK, sorry about
that._

BGA's database layer is... old. Writting database requests by hand is tedious
and prone to errors.

![Stone age: Typing SQL queries by hand](/blog/assets/img/stone-age-db/stone-age.jpg){: width="400"}

So after developing 2 games on BGA, I understood enough
of the framework to build my own library on top of what is provided.

And the first thing on my todo list: a better database layer!

Let's start with a simple table named `shape`:

```php
$this->getObjectListFromDB(
    "SELECT shape_id, shape_type_id, player_id FROM shape"
);
```
* `shape_id` is a unique id for the shape.
* `shape_type_id` is a number. 0 is a triangle, 1 is a square and 2 is an hexagon.
* `player_id` is the player that owns the shape. It's `NULL` if the shape is in the general supply.

We would like to be able to read each rows in a class like this:

```php
class Shape
{
    public $shape_id;
    public $shape_type_id;
    public $player_id;
}
```

We need to be able to match columns with the class properties and also know which property
is the primary key of the table. There are many ways to do that but let's look at _annotations_.

## Annotations / Attributes

PHP 8 adds _attributes_ (called _annotations_ in other languages) which could be very useful
here... but right now we are stuck with PHP 7.4. But all is not lost since we can use
_Doc comments_ to implement something similar. Our class will look like this:

```php
class Shape
{
    /** @dbcol @dbkey */
    public $shape_id;
    /** @dbcol */
    public $shape_type_id;
    /** @dbcol */
    public $player_id;
}
```

Note the double stars at the start of the comments: this normally indicates to
PHP that this comment should be extracted to generate documentation. But we will
use this to implement annotations. (This is not my idea, there are existing
libraries that do this too.)

Here's how it works:

```php
$reflect = new ReflectionClass(Shape::class);
foreach ($reflect->getProperties() as $property) {
    $name = $property->getName();
    $doc = $property->getDocComment();
    echo("$name: $doc\n");
}
```

And the output will look like this:
```
shape_id: /** @dbcol @dbkey */
shape_type_id: /** @dbcol */
player_id: /** @dbcol */
```

We use PHP's [ReflectionClass](https://www.php.net/manual/en/book.reflection.php)
to list properties and get their _doc comment_. This class has other very useful
methods when you need to inspect the internals of classes.

With that, we only need to parse those comments to know which properties are
database columns and which are also the primary key(s) of the table. We'll leave
the parsing out of this blog post, but see the end for a link to the full
implementation.

![Parsing](/blog/assets/img/stone-age-db/parsing.gif){: width="300"}

## Generating SELECT

Generating a `SELECT` is now easy:
```php
$dbColumns = implode(',', array_map(function ($p) {
    // We would need to filter to take only the @dbcol annotations
    return $p->getName();
}, $reflect->getProperties()));

$sql = "SELECT $dbColumns FROM shape";
echo($sql);
// Output:
// SELECT shape_id,shape_type_id,player_id FROM shape
```

## Generating UPDATE, INSERT and DELETE

Generating an `UPDATE` from a `Shape` instance requires a little helper to
properly format values for SQL:

```php
function sqlNullOrValue($value)
{
    if ($value === null) {
        return "NULL";
    }
    if (is_string($value)) {
        return "'" . addslashes($value) . "'";
    } else if (is_bool($value)) {
        return ($value ? "1" : "0");
    } else {
        return "$value";
    }
}
```

With that, we can generate the `UPDATE`:

```php
$row = new Shape();
$row->shape_id = 123;
$row->shape_type_id = 1; // A square!
$row->player_id = null;

$dbValues = implode(', ', array_map(function ($p) use ($row) {
    // We would need to filter to take only the @dbcol annotations
    return $p->getName() . " = " . sqlNullOrValue($p->getValue($row));
}, $reflect->getProperties()));

$dbKeys = implode(' AND ', array_map(function ($p) use ($row) {
    // We would need to filter to take only the @dbkey annotations
    return $p->getName() . " = " . sqlNullOrValue($p->getValue($row));
}, $reflect->getProperties()));

$sql = "UPDATE shape SET $dbValues WHERE $dbKeys";
echo($sql);
// Output (if it was properly filtered):
// UPDATE shape SET shape_id = 123, shape_type_id = 1, player_id = NULL WHERE shape_id = 123
```

Finally, generating an `INSERT` or a `DELETE` is pretty much the same.

## Wrapping everything in a manager class

![Look at me, I'm the manager now](/blog/assets/img/stone-age-db/manager.jpg){: width="300"}

We can now take everything a wrap it in a nice class:

```php
// APP_DbObject allows us to call BGA database functions
class RowManager extends APP_DbObject
{
    // ... private members ...

    public function __construct(string $tableName, string $baseRowClassName)
    {
        // ... keep those parameters in private members ...
    }

    public function insertRow($row)
    {
        // Generate INSERT and call $this->DBQuery($sql)
    }

    public function updateRow(BaseRow $row)
    {
        // Generate UPDATE and call $this->DBQuery($sql)
    }

    public function deleteRow(BaseRow $row)
    {
        // Generate DELETE and call $this->DBQuery($sql)
    }

    public function getAllRows()
    {
        $sql = // Generate SELECT
        foreach ($this->getObjectListFromDB($sql) as $row) {
            $rowObject = new $this->baseClassRowName;
            foreach ($row as $property => $value) {
                $rowObject->$property = $value;
            }
            // $rowObject properties are now initialised. Keep all
            // those objets in an array to return them all.
        }
    }
}
```

We can use our new manager it like this:

```php
$row = new Shape();
$row->shape_id = 123;
$row->shape_type_id = 1; // A square!
$row->player_id = null;

$manager = new RowManager('shape', Shape::class);
$manager->insertRow($row);

$manager->getAllRows(); // Returns an array of Shape instances
```

Success!

![Great success](/blog/assets/img/stone-age-db/success.jpg){: width="300"}

... But we can do much better now that we got a working database manager.

## The case of shape_type_id

The fact that `shape_type_id` is a integer means that we need to write code like this:

```php
class Shape 
{
    // ... Properties ...

    // Not the best example but you get the idea
    public function countSides()
    {
        switch ($this->shape_type_id) {
            case 0:
                return 3;
            case 1:
                return 4;
            case 2:
                return 6;
        }
        throw new BgaSystemException("Unknown shape_type_id!");
    }
}
```

This is not very nice, especially if you have multiple functions and you need to
add a new shape type! Let's try to change how `Shape` is defined:

```php
abstract class Shape
{
    /** @dbcol @dbkey */
    public $shape_id;
    // NOTE THE CHANGE ON THE NEXT 2 LINES
    /** @dbcol @dbclassid */
    public $class_id;
    /** @dbcol */
    public $player_id;

    abstract public function countSides();
}

class Triangle extends Shape
{
    public function countSides()
    {
        return 3;
    }
}

class Square extends Shape
{
    public function countSides()
    {
        return 4;
    }
}

class Hexagon extends Shape
{
    public function countSides()
    {
        return 6;
    }
}

```

Much better! In the database, `class_id` must be a (long enough) varchar. But how to
get our database manager to understand this?

```php
class RowManager extends APP_DbObject
{
    // No change to constructor

    public function insertRow($row)
    {
        // We must now do this:
        $propertyClassId = // Get property with @dbclassid, so 'class_id' in our case
        if ($propertyClassId !== null) {
            // get_class will return the real class as
            // a string, so 'Triangle', 'Square' or 'Hexagon'
            $row->$propertyClassId = get_class($row);
        }
        // Generate INSERT and call $this->DBQuery($sql)
    }

    public function updateRow(BaseRow $row)
    {
        // We do the same thing to update the property with @dbclassid
        // Generate UPDATE and call $this->DBQuery($sql)
    }

    // No change for deleteRow

    public function getAllRows()
    {
        $sql = // Generate SELECT
        foreach ($this->getObjectListFromDB($sql) as $row) {
            $classId = $this->baseClassRowName;
            $propertyClassId = // Get property with @dbclassid, so 'class_id' in our case
            if ($propertyClassId !== null) {
                $classId = $row[$propertyClassId];
            }
            // PHP can create class if you have their name in a string
            $rowObject = new $classId;
            foreach ($row as $property => $value) {
                $rowObject->$property = $value;
            }
            // $rowObject properties are now initialised. Keep all
            // those objets in an array to return them all.
        }
    }
}
```

We can now create shapes and save them:


```php
$row1 = new Square();
$row1->shape_id = 123;
$row1->player_id = null;

$row2 = new Triangle();
$row2->shape_id = 456;
$row2->player_id = 67890123;


$manager = new RowManager('shape', Shape::class);
$manager->insertRow($row1);
$manager->insertRow($row2);

$manager->getAllRows(); // Returns [Square, Triangle]
```

Note that unlike real inheritance, all the properties must be in the base class
since we only have one table.

## And that's not all!

![But wait, that's not all](/blog/assets/img/stone-age-db/but-wait-thats-not-all.jpg){: width="300"}

This post is only a part of what you can do! For example, in my implementation,
I support the annotation `@dbautoincrement` to indicate that the column is
autoincrement in the database so that the library can read the new value from
the database when inserting. I also prefer my database columns to be `snake_case`
(like `shape_id`) while in PHP they are `camelCase` (like `$shapeId`) so I
do this conversion too. And so much more!

If you want to see more of the database library, you can look at the
implementation in
[DB.php](https://github.com/bennygui/barenpark-public/blob/main/modules/php/BX/DB.php).
The annotation library (and more reflection-related functions) is in another
file:
[Meta.php](https://github.com/bennygui/barenpark-public/blob/main/modules/php/BX/Meta.php).

Hopefully, next time we'll talk about what we can build on top of that database
manager. Until next time, have fun!