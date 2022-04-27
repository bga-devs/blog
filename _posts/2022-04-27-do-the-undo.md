---
title: Do the undo &mdash; The search for an undo system
author: Guillaume Benny
date: 2022-04-27 8:00:00 -0400
categories: [Tips]
tags: [back,class,database]
pin: true
---

Let's talk about undoing.

BGA has an
[undo system](https://en.doc.boardgamearena.com/Main_game_logic:_yourgamename.game.php#Undo_moves)
but it has limitations:

* You only have one undo. If you have a multi-step action, undoing means
restarting from the beginning.
* The undo system saves the whole database and restores everything. So it's
impossible to use when multiple players can do something to the database, whether
in a _multipleactiveplayer_ state --- or if your inactive players can set an option
specific to the table --- or anything else that touches the database.

This means that you often have to roll your own undo system. One solution is
what I did for [The Isle of Cats](https://boardgamearena.com/gamepanel?game=theisleofcats):
you do your multi-step actions on the client side. This is totally doable with your own
system or with
[`setClientState`](https://en.doc.boardgamearena.com/BGA_Studio_Cookbook#Multi_Step_Interactions:_Select_Worker/Place_Worker_-_Using_Client_States).
And `setClientState` is often the simplest --- and best --- choice. But
this can lead to code duplication between the client side and the server side
since you must do more on the client side (like validations).

So after creating a [database layer](/blog/posts/stone-age-db/), I set out
to create a reusable, server-side undo system.

![OK](/blog/assets/img/do-the-undo/ok.gif){: width="400"}

## A simple start

Let's start with a simple game with only one database table, `shape`, with 3 columns:
* `shape_id` is a unique id for the shape.
* `shape_type_id` is a number. 0 is a triangle, 1 is a square and 2 is an hexagon.
* `player_id` is the player that owns the shape. It's `NULL` if the shape is in the general supply.

Let's say that on your turn, you can do something to take the shape. So you need to change
`player_id` from `NULL` to `1234`. To be able to undo that, you could add a new `boolean`
column that is `TRUE` when the row has been modified. This works, but if it's possible to
take more than one shape, you won't know the order in which the actions where done. So
you change this new column to an integer to keep the order. This works until you can take the shape
of another player: you must then remember the previous `player_id`. This really
doesn't scale: soon you'll be taking a copy of the whole database and we're back
to the BGA undo system.

## Something different

We really need a different system. Let's start at the very beginning: something
that can _do_. We'll assume that we have a [database
layer](/blog/posts/stone-age-db/) that return classes for our table:

```php
// Free function
function getShapeById($shapeId)
{
    $shape = // Get Shape class instance from database layer
    return $shape;
}

class PlayerTakeShapeActionCommand
{
    private $playerId;
    private $shapeId;

    public function __construct($playerId, $shapeId)
    {
        // You know the drill
    }

    public function do()
    {
        $shape = getShapeById($this->shapeId);
        $shape->playerId = $this->playerId;
    }
}

// And somewhere in mygame.game.php (so in "class mygame extends Table"):
public function playerTakeShape($shapeId)
{
    // Not shown: other validations...
    $action = new PlayerTakeShapeActionCommand($this->getActivePlayerId(), $shapeId);
    $action->do();
}

protected function getAllDatas()
{
    $result = [];
    $result['shapes'] = getAllShapes(); // getAllShapes() is left to your imagination
    return $result;
}
```

When a player clicks on a shape and `playerTakeShape()` is called, this does... nothing.
* No notifications are sent, so the client doesn't know something happened.
* Nothing is saved to the database so the change does not persist.

This is great! _Great?_ Yes! You'll see later why we need something that does
nothing. But for now, we need it to do a bit more: we need the modification to
the shape to persist _in memory_:

```php
// Global
$modifiedShapes = [];

// Free function
function markShapeModified($shape)
{
    global $modifiedShapes;
    // $shape references a class (so it's _not_ a copy)
    $modifiedShapes[$shape->shapeId] = $shape;
}

// Free function
function getShapeById($shapeId)
{
    global $modifiedShapes;
    if (array_key_exists($shape->shapeId, $modifiedShapes)) {
        return $modifiedShapes[$shape->shapeId];
    }
    $shape = // Get Shape class instance from database layer
    return $shape;
}

class PlayerTakeShapeActionCommand
{
    // ...
    public function do()
    {
        $shape = getShapeById($this->shapeId);
        markShapeModified($shape);
        $shape->playerId = $this->playerId;
    }
}
```

This still does nothing: after the request to the server, what is only in memory
is lost!  But if we could save the instance of `PlayerTakeShapeActionCommand` to
the database and reload it, we could call its `do()` function.  And after the
`do()`, `getShapeById()` would always return the modified shape.

## A detour into serialize-land

![Detour](/blog/assets/img/do-the-undo/detour.jpg){: width="400"}

With PHP's [ReflectionClass](https://www.php.net/manual/en/book.reflection.php),
it's actually easy to tranform (almost) any class to a string and back:
* `ReflectionClass` allows us to loop on all properties and get their names and
values, even if they are private.
* If a value is an array or an object, we need to do a recursive call to
process everything.
* If its an object, we need to remember the class name to recreate it.

```php
// Free function
function extractAllPropertyValues($object)
{
    if (is_array($object)) {
        return array_map(function ($o) {
            return extractAllPropertyValues($o);
        }, $object);
    } else if (!is_object($object)) {
        return $object;
    }
    $allProperties = [
        // The name @classId is arbitrary. It's prefixed with
        // an @ to avoid collisions with real properties
        '@classId' => get_class($object),
    ];

    $reflect = new ReflectionClass(get_class($object));
    foreach ($reflect->getProperties() as $property) {
        $property->setAccessible(true); // Bypass private or protected
        $value = $property->getValue($object);
        $allProperties[$property->getName()] = extractAllPropertyValues($value);
    }

    return $allProperties;
}
```

This function actually gives us an associative array than can be converted to a
string with `json_encode()`. To recreate the object, we do the samething in reverse.
We also need to use `ReflectionClass::newInstanceWithoutConstructor` to avoid the class
constructor (for which we don't know the parameters):

```php
// Free function
function rebuildAllPropertyValues($values)
{
    if (!is_array($values)) {
        return $values;
    }
    if (array_key_exists('@classId', $values)) {
        $reflect = new ReflectionClass($values['@classId']);
        $object = $reflect->newInstanceWithoutConstructor(); // Like 'new' but does not call the constructor
        unset($values['@classId']);
        foreach ($values as $propertyName => $value) {
            $value = rebuildAllPropertyValues($value);
            $property = $reflect->getProperty($propertyName);
            $property->setAccessible(true); // Bypass private or protected
            $property->setValue($object, $value);
        }
        return $object;
    } else {
        return array_map(function ($value) {
            return rebuildAllPropertyValues($value);
        }, $values);
    }
}
```

Again, this requires a call to `json_decode($string, true)` to convert a string
to an associative array that can be passed to `rebuildAllPropertyValues()`.

## Back to doing

Now that we can tranform our class into a string, we only need a table --- let's
call it `action_command` --- with two columns:
* An autoincrement id, which is the order the actions are inserted
* A big enough varchar column to store the serialized class.

Our game functions now looks like this:

```php
// Free function
public function saveActionToDatabase($action)
{
    $string = json_encode(extractAllPropertyValues($action));
    // Save $string in action_command table
}

// Free function
public function reloadActionsFromDatabase()
{
    foreach (/*get rows from action_command table*/ as $row) {
        $action = rebuildAllPropertyValues(json_decode($row->string_column, true));
        // Don't forget, calling "do()" will fill $modifiedShapes
        $action->do();
    }
}

// Somewhere in mygame.game.php (so in "class mygame extends Table"):
public function playerTakeShape($shapeId)
{
    reloadActionsFromDatabase();
    // Not shown: other validations...
    $action = new PlayerTakeShapeActionCommand($this->getActivePlayerId(), $shapeId);
    $action->do();
    saveActionToDatabase($action);
}

protected function getAllDatas()
{
    reloadActionsFromDatabase();
    $result = [];
    // NOTE: getAllShapes() must look in $modifiedShapes for this to work
    $result['shapes'] = getAllShapes(); // getAllShapes() is left to your imagination, as long as you imagine the right implementation :)
}
```

Suddenly, we can save actions and redo them each time we need it! ... But we
still need to reload to see the result. We need to integrate notifications into
this. Let's create a class to help us:

```php
class Notifier
{
    public function notify($notifType, $notifLog, $notifArgs)
    {
        // Get a reference to the game class and call notifyAllPlayers
    }
}
```

We can now notify our players:

```php
class PlayerTakeShapeActionCommand
{
    // ...
    public function do($notifier)
    {
        $shape = getShapeById($this->shapeId);
        markShapeModified($shape);
        $shape->playerId = $this->playerId;
        $notifier->notify(
            'MOVE_SHAPE_TO_PLAYER',
            'Moving!',
            [
                'playerId' => $this->playerId,
                'shapeId' => $this->shapeId,
            ]
        );
    }
}
```

But there's a problem: we want to send a notification only the first time
we do the action, not when we reload it from the database. Another class
to the rescue: just drop notifications when reloading.

```php
class ReloadNotifier
{
    // Yes, a function that does nothing!
    public function notify($notifType, $notifLog, $notifArgs)
    {
    }
}

// Free function
public function reloadActionsFromDatabase()
{
    $notifier = new ReloadNotifier();
    foreach (/*get rows from action_command table*/ as $row) {
        $action = rebuildAllPropertyValues(json_decode($row->string_column, true));
        $action->do($notifier);
    }
}
```

## But... where's the undo?

![Where is the undo](/blog/assets/img/do-the-undo/where.gif){: width="260"}

Now that we know how to _do_, let's _undo_.

Again, if we could convince everyone to just reload the whole page after each
action, undoing would only requires deleting the last row in the
`action_command` table. But we need those notifications. So in our
`PlayerTakeShapeActionCommand` class, we must remember what is needed to undo
the action and add a function to send the notification:

```php
class PlayerTakeShapeActionCommand
{
    private $playerId;
    private $shapeId;
    // NEW
    private $previousPlayerId;

    public function __construct($playerId, $shapeId)
    {
        // You know the drill: store $playerId and $shapeId in properties
        // But leave $previousPlayerId null
    }

    public function do($notifier)
    {
        $shape = getShapeById($this->shapeId);
        // NEW
        $this->previousPlayerId = $shape->playerId;
        markShapeModified($shape);
        $shape->playerId = $this->playerId;
        $notifier->notify(
            'MOVE_SHAPE_TO_PLAYER',
            'Moving!',
            [
                'playerId' => $this->playerId,
                'shapeId' => $this->shapeId,
            ]
        );
    }

    // NEW
    public function undo($notifier)
    {
        $notifier->notify(
            'MOVE_SHAPE_TO_PLAYER',
            'Undoing!',
            [
                'playerId' => $this->previousPlayerId,
                'shapeId' => $this->shapeId,
            ]
        );
    }
}
```

And the implementation of undo is straightforward:

```php
// Somewhere in mygame.game.php (so in "class mygame extends Table"):
public function undo()
{
    // Send an error if the action_command table is empty: no actions to undo!
    $row = // Get last row from from action_command table
    $action = rebuildAllPropertyValues(json_decode($row->string_column, true));
    $action->undo(new Notifier());
    // Delete $row from action_command table
}
```

When the player confirms their actions --- or if they are about to do an action
that cannot be undone --- we need to:
* Load all rows from `action_command` table and call `do()`;
* Save all modified shapes that are in `$modifiedShapes` array;
* Delete all rows from the `action_command` table!

That's it, we've done it!

## A nice side effect

Once you have such a system, something unexpected happens: you know the actions
that the player did and you can get useful information out of this. This can replace
some clunky globals that you need to keep to remember the state of the game.

Here's a real example. In the game [Bärenpark](https://boardgamegeek.com/boardgame/219513/barenpark),
a player turn is like this:
1. You take a tile from your own supply.
2. You place the tile on your own board. In doing this, you cover some icons.
3. You take tiles from the general supply. The tiles you are allowed to take depends on the covered icons.

Normally, to implement this, you need a table or some globals to remember:
1. The tile that is choosen.
2. The icons that are covered.
3. Each time you take a tile from the general supply, you must match it with the covered icons and remove
the icon from the list of icons that are still available.

But with the information in the `action_command` table, you can get this
information _for free_.

For example, when you choose a tile in step 1, you save the
`ChooseTileActionCommand` class with a `$shapeId` property. When you are in the
state for step 2, you can search the actions for this class and get the selected
`$shapeId`.

The same kind of idea works also for step 2 and 3: in the `do()` function of step 2, you can
save the covered icons in a member array of the class. Once in step 3, you can query those
icons to get what tile is available.

The `do()` function of the `ActionCommand` classes is also a great place to
validate the parameters: is the selected `shapeId` really a valid shape? If not,
just throw an exception to get out of there!

## More things to think about

At this point, you might be convinced that this is a good idea. And I am! But if
you are about to go with such a system, you still have other important things
to think about because a lot of things where left out:

* __State and transitions__: At some point, you will need an `ActionCommand` class
to _do_ and _undo_ state transitions.
  * This is a good idea but you need to do the transition only in the first `do()`,
  just like with notifications. So the `Notifier` class is a good place to encapsulate
  the code that really calls `$this->gamestate->nextState()`.
  * When undoing, you either need to have a valid transition to the previous state,
  or you need to use the undocumented `$this->gamestate->jumpToState($stateId)` call
  to got back to the previous state.
* __Grouping ActionCommand__: You can split your `ActionCommand` in smaller, reusable
classes which is great. But if you use more that one `ActionCommand` in one player action,
you will undo only part of the action if your undo only deletes the last row
of the `action_command` table. So create a `GroupActionCommand` class: you add your
`ActionCommand` instances to it and save only the `GroupActionCommand`. The `do()` of
this class only needs to loop on the added classes and call their `do()` functions.
The undo is the samething, but you call `undo()` in reverse order.
* __Upgrading ActionCommand__: Once the game is released, you might need to change
an `ActionCommand` class but you will not be able to use BGA's `upgradeTableDb()`
function. There are a few options:
  * You can create a new class. You leave the `ChooseTileActionCommand` class like
  it is and you create and start using `ChooseTileActionCommandVersion2` class
  instead. Old games will be able to undo the first class and when they choose a new
  tile, the new class will be created and saved.
  * You add a new property in your existing class and initialize it in the constructor.
  When the class is created in your new code, it will have that value. But if it's read
  back from the `action_command` table from a game that started before the new version,
  the property will be `null` because it won't be initialized by `rebuildAllPropertyValues()`.
  You can then react accordingly.

## A real implementation

There's a lot in this post but it's a real system that works and the game
[Bärenpark](https://boardgamearena.com/gamepanel?game=barenpark) uses it (the
game is in Alpha at the time of writing).

If you want to poke around and see the real implementation, see
[Action.php](https://github.com/bennygui/barenpark-public/blob/main/modules/php/BX/Action.php)
from Bärenpark's library. Here are a few things to know that will help you
browse all this code:
* `BaseActionRow` and `BaseActionRowMgr` are the base classes for rows
that are read from database tables but that can also be modified in memory
only. This replaces the ugly global `$modifiedShapes` and associated functions
in the example above.
* `BaseActionCommand` is a base class for all `ActionCommand`
* `ActionCommandRow` is a row in the `action_command` table.
* `BaseActionCommandNotifier` and all derived classes are the `Notifier` in the example above.
* `ActionCommandMgr` loads, saves and deletes rows from the `action_command` table. It also calls
`do()` and `undo()`.

You call also look at real `ActionCommand` classes from
[the game](https://github.com/bennygui/barenpark-public/blob/main/modules/php/BP/Action.php),
like `ChooseTileFromPlayerSupplyActionCommand`which is very simple.

Finally, you can check the function
[`chooseTileFromPlayerSupply`](https://github.com/bennygui/barenpark-public/blob/main/modules/php/BP/States/ChooseTileFromPlayerSupply.php)
which is called when the player clicks on a shape. The undo system allows the
function to be very short and easy to read.

## More features

One thing to note is that, in Bärenpark's implementation, what a player does is
private to that player until they confirm their turn. This is to allow the
player to _try_ some tile placements, like you can do in
[Patchwork](https://boardgamearena.com/gamepanel?game=patchwork) or [The Isle of
Cats](https://boardgamearena.com/gamepanel?game=theisleofcats).

This also allows some weirder things like preparing your next move even if it's
not your turn! For this, the game tracks _private states_, meaning that all
players are in the same state for BGA's framework but the game also has another
table to track states for each player. The idea of private states was taken from
[Welcome to](https://github.com/Syarwin/bga-welcometo) which uses it for a
_multipleactiveplayer_ state, but my implementation is pretty similar. [Private
parallel
states](https://en.doc.boardgamearena.com/Your_game_state_machine:_states.inc.php#Private_parallel_states)
based on Welcome's implementation are now available in BGA's framework, but they
work only when multiple players are active.

Finally, allowing players to prepare their next turn means that there might be
conflicts to resolve. If you are interessted in that, see
`ActionCommandMgr::getReevaluationArgs()` and `ActionCommandMgr::reevaluate()`.
I wouldn't recommend doing something more complicated than undoing the
conflicting actions, as trying to fix actions can become complicated very fast.
But Bärenpark's implementation does do a few tricks if you are eager to go down
the rabbit hole.

## The End!

![That's all](/blog/assets/img/do-the-undo/thats-all.jpg){: width="400"}

That's all for now! Until next time, have fun!