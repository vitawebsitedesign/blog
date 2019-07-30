---
title: CSS Spritesheet Limitations
layout: post
author: Michael Nguyen
---
## For
The common argument FOR using CSS spritesheets is that they reduce the http requests (network congestion). This is important because of browser http persistent connection request limits (thereby speeding up page load times).

Another useful benefit is that it's an alternative to prevent "image flickering" (when an image needs to be swapped out at runtime, e.g. mouse hover, and the time between the new image request & response is large enough for the naked eye).

## Against
The common argument AGAINST is that they complicate CSS, they present a less malleable solution, but most importantly, represent a larger load time when the spritesheet needs to be first downloaded. While the latter pitfall can be mitigated through CDN hosting, this post assumes no CDN.

## Lost semantic meaning
CSS sprites are implemented via background-image, which likely means you wont be using an <img />.

Images have the alt attribute, which improve accessibility for both users (captions) and text-only screen readers.

If the alt attribute is a functional requirement, then CSS spritesheets are more of a pitfall. But if not, then they become an available option.

## Whats even the point if individual images are cached too?
Individual images get cached, so technically if you have 2 pages that use the same image url/domain, then yes, the 2nd http request wont be made the file will just be fetched from browser cache.

CSS spritesheets are more for the situation where you have 3 sprites, 1 used on 1 page, and 1 on another page. In this case, the 2nd http request wont be made. Whereas if you just used <img />, the 2nd http request would slow down 2nd page initial load speed.

## When should i use css spritesheets?
This aint a black and white question - you gotta analyze your priorities, order them based on business requirements and technical limitations, and ensure that the benefits significantly outweigh the dragbacks.
