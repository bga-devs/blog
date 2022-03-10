---
title: Why (I) use TypeScript
author: Guy Baudouin
date: 2022-03-09 19:00:00 +0800
categories: [Tips]
tags: [front,typescript,debug]
pin: true
---

Using TypeScript on my day-time job, I'm very used to it and I can see the advantages of it. As it is not integrated by default on BGA framework, I made a tutorial to set it up on your projects.

I'll show 2 examples, one using JavaScript, the other using TypeScript. In both cases, I just want to update a counter on the player panel when the player play a card (counter should go from 8 to 7).

Here is an example using only JavaScript :

<video controls width="100%" name="js">
    <source src="https://bga-devs.github.io/blog/assets/video/typescript/js.mp4" type="video/mp4">
</video>

At the end, no JS error is visible, but the code doesn't work. I need to do more debugging, but I skipped that part so the video isn't too long, but you got it, it's long and it's a pain.

Here is the example using TypeScript :

<video controls width="100%" name="ts">
    <source src="https://bga-devs.github.io/blog/assets/video/typescript/ts.mp4" type="video/mp4">
</video>

As you can see, as TypeScript got strong typing check, I can see errors and fix them when I'm writing my code, instead of waiting run-time to see/debug.

If you are interested, doc to setup TypeScript is here : https://en.doc.boardgamearena.com/Using_Typescript_and_Scss
