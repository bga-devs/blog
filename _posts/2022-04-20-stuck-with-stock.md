---
title: Stuck with Stock?
author: Guillaume Benny
date: 2022-04-20 8:00:00 -0400
categories: [Tips]
tags: [front,css]
pin: true
---

If you are on the [bga-developer discord](https://en.doc.boardgamearena.com/Studio#Other_resources),
you will often see this answer when someone asks about
[Stock](https://en.doc.boardgamearena.com/Stock):

> Don't use stock.
>
> --- <cite>Everyone (almost)</cite>

You might want to use Stock in some special cases but most of the time you can
have more control and use less code with a pure CSS solution. Let's look
at how we can do that... and more.

## CSS Sprite

Let's create a [CSS
sprite](https://en.doc.boardgamearena.com/Game_art:_img_directory#Use_CSS_Sprites),
a single image with the 4 (ugly) tiles of our game:

![Tiles](/blog/assets/img/stuck-with-stock/tiles.png)

A good way to assign the tiles is to use _data-attributes_:

```css
.tile-supply {
    /* Nothing for now */
}

.tile {
    width: 80px;
    height: 80px;
    background-image: url("tiles.png");
    background-size: 160px 160px;
}

.tile[data-tile-id="1"] { background-position:   0px   0px; }
.tile[data-tile-id="2"] { background-position: -80px   0px; }
.tile[data-tile-id="3"] { background-position:   0px -80px; }
.tile[data-tile-id="4"] { background-position: -80px -80px; }
```

And then we can use the `data-tile-id` attribute to create tiles:

```html
<div class="tile-supply">
    <div class="tile" data-tile-id="4"></div>
    <div class="tile" data-tile-id="2"></div>
    <div class="tile" data-tile-id="1"></div>
    <div class="tile" data-tile-id="3"></div>
</div>
```

It's not pretty but it works:

![Tiles 1](/blog/assets/img/stuck-with-stock/tiles-1.png)

## Flex layout

The simplest way to have a nicer supply of tiles is to use flex:

```css
.tile-supply {
    display: flex;
    flex-wrap: wrap;
}

.tile-supply .tile {
    margin: 10px;
}
```

![Tiles 2](/blog/assets/img/stuck-with-stock/tiles-2.png)

Or you could display them in a column by adding `flex-direction: column`
in `.tile-supply`.

You might want to always have the tiles in the same order in the supply.
Or maybe they take too much space and you want to stack them. To do that
nicely for both cases, we'll first talk about `var()`.

## What is var()?

When I saw CSS variables for the first time, I saw this kind of example:

```css
:root {
  --primary-color: blue;
}

.example {
  color: var(--primary-color);
}
```

So all elements with the `example` class will have blue text. This is very
useful to avoid duplicating information in the CSS but since
[SCSS](https://en.doc.boardgamearena.com/Using_Typescript_and_Scss) can do the
same kind of things, I didn't think about it for a while.

![Afraid to ask](/blog/assets/img/stuck-with-stock/afraid-to-ask.jpg){: width="300" }


But CSS variables can do so much more: they are inherited, they can be
overridden in a child class or style attribute, and more! The most important
thing to know is that when you write `var()`, the value will be read from the
style of the element; if it's not defined in the style, it will be read from
the class; if it's not defined in the class, it will be read from the parent,
and so on.

Let's come back to our example. We want our tiles to always be in the same
order. We already have an id for each tile, so we'll use it as our order. Since
`attr()` can be used to access the value of other attributes, this works:

```css
.tile-supply .tile::after {
    content: attr(data-tile-id);
    color: white;
    font-size: 40px;
}
```
![Tiles 3](/blog/assets/img/stuck-with-stock/tiles-3.png)

And so, you might be tempted to write this:

```css
.tile-supply .tile {
    margin: 10px;
    /* Sorry, this does not work! */
    order: attr(data-tile-id);
}
```

But it does not work. This is because the `order` property requires a number but
data attributes are always strings. But CSS variables to the rescue: they
are typed! So this works:

```css
.tile[data-tile-id="1"] {
    background-position: 0px 0px;
    /* Added */
    --supply-order: 1;
}
/* And also for ids 2, 3 and 4... */

.tile-supply .tile {
    margin: 10px;
    /* Now this works! */
    order: var(--supply-order);
}
```
![Tiles 4](/blog/assets/img/stuck-with-stock/tiles-4.png)

## Stacking tiles

Flex is nice if we need to see them all but, in a lot of games, the tiles are
stacked. How can we do that? With absolute positioning:

```css
.tile-supply .tile {
    position: absolute;
    top: 0px;
    left: 0px;
}
```

![Tiles 5](/blog/assets/img/stuck-with-stock/tiles-5.png)

OK... But the one on top is supposed to be the one with the highest id.
Let's fix that:

```css
.tile-supply .tile {
    position: absolute;
    z-index: var(--supply-order);
    top: 0px;
    left: 0px;
}
```
![Tiles 6](/blog/assets/img/stuck-with-stock/tiles-6.png)

It works but we can't see that there are multiple tiles on top of each other.
To solve that, we can give each tile a slight offset.

## Calculating with calc()

Once you have `var()`, the `calc()` function becomes very handy since it allows
us to do calculations with our variables. Just watch your spacing around
operators or else `calc()` will be very confused.

Back to our offset:

```css
.tile-supply .tile {
    position: absolute;
    z-index: var(--supply-order);
    /* calc() can work with units */
    top: calc(10px * (var(--supply-order) - 1));
    left: calc(10px * (var(--supply-order) - 1));
    /* Not required but nicer! */
    box-shadow: 1px 1px 5px 0px black;
}
```
![Tiles 7](/blog/assets/img/stuck-with-stock/tiles-7.png)

The tiles are stacked!

## Viewing all stacked tiles

We haven't adressed the issue of the id of the tiles being off center:

```css
.tile {
    width: 80px;
    height: 80px;
    background-image: url("tiles.png");
    background-size: 160px 160px;
    /* New lines: this will affect childs
       of .tile, so only ::after in our case */
    display: flex;
    justify-content: center;
    align-items: center;
}
```
![Tiles 8](/blog/assets/img/stuck-with-stock/tiles-8.png)

But now, if the id is an important information, we can't see it
anymore. Since everything is in CSS, we can add a new rule when
we hover over the tiles to spread them:

```css
.tile-supply .tile {
    position: absolute;
    z-index: var(--supply-order);
    top: calc(10px * (var(--supply-order) - 1));
    left: calc(10px * (var(--supply-order) - 1));
    box-shadow: 1px 1px 5px 0px black;
    /* New */
    transition: left 0.5s ease-in-out, top 0.5s ease-in-out;
}

/* New */
.tile-supply:hover .tile {
    top: 0px;
    left: calc(90px * (var(--supply-order) - 1));
}
```
![Tiles 9](/blog/assets/img/stuck-with-stock/tiles-9.gif)

Of course, it doesn't have to be on hover: you can also use a button and add a
css class (in javascript) to the `.tile-supply` element when you click on the
button.

## Dynamic CSS variables

One final thing to show the usefulness of CSS variables.

On our supply board, our tiles are now stacked. Let's say that if the player
takes a tile, it goes in his player supply and it should not be stacked but it
should be placed in a particular order.

When creating the tiles in javascript, we can set different CSS variables:

```javascript
createTile(tileId, playerOrder) {
    const tile = document.createElement('div');
    tile.classList.add('tile');
    tile.dataset.tileId = tileId;
    tile.style.setProperty('--supply-order', tileId);
    tile.style.setProperty('--player-order', playerOrder);
    return tile;
}
```

Note that I prefer to use vanilla javascript to create HTML elements instead of
`this.format_block`/`dojo.place`: string substitution is good for complex HTML
but harded to read for simple cases.

And in your CSS you can use the variables to place the tiles:

```css
/* No change here, still using --supply-order */
.tile-supply .tile {
    position: absolute;
    z-index: var(--supply-order);
    top: calc(10px * (var(--supply-order) - 1));
    left: calc(10px * (var(--supply-order) - 1));
    box-shadow: 1px 1px 5px 0px black;
    transition: left 0.5s ease-in-out, top 0.5s ease-in-out;
}

/* New */
.player-supply {
    display: flex;
    flex-wrap: wrap;
}

/* New */
.player-supply .tile {
    order: var(--player-order);
}
```

This way, when moving tiles from the general supply to the player supply, the
tiles will automatically be placed perfectly, without having to do anything
special in javascript.

That's all for now! Until next time, have fun!