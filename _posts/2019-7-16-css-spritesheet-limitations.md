---
title: CSS Spritesheet Limitations
layout: post
---

## For
The common argument FOR using CSS spritesheets is that they reduce the http requests (network congestion). This is important because of browser http persistent connection request limits (thereby speeding up page load times).

Another useful benefit is that it's an alternative to prevent "image flickering" (when an image needs to be swapped out at runtime, e.g. mouse hover, and the time between the new image request & response is large enough for the naked eye).

## Against
The common argument AGAINST is that they complicate CSS, they present a less malleable solution, but most importantly, represent a larger load time when the spritesheet needs to be first downloaded. While the latter pitfall can be mitigated through CDN hosting, this post assumes no CDN.

## When should i use css spritesheets?
This aint a black and white question - you gotta analyze your priorities by order and ensure that the benefits outweigh the costs.

## Lost semantic meaning
CSS sprites are implemented via background-image, which likely means you wont be using an <img />.

Images have the alt attribute, which improve accessibility for both users (captions) and text-only screen readers.

If the alt attribute is a functional requirement, then CSS spritesheets are more of a pitfall. But if not, then they become an available option.
