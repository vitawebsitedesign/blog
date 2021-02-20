---
title: Blindly applying defensive programming is a recipe for disaster
layout: post
author: Michael Nguyen
---

Depending on the system requirements, fault tolerance may be a priority for your product. This is especially true in mission-critical systems that need to consider high availability, data integrity, security & physical safety.

For instance, what happens when the [“cowboy approach”](https://en.wikipedia.org/wiki/Cowboy_coding) is applied when developing a physical medical device without considering safety of humans (e.g.: patient, nurse, doctor). What about an **offensive** programming approach instead?

Of course, on the flip side, an overly **defensive** approach that’s not necessary may introduce unnecessary excessive schedule risk & be a terrible choice.

This brings us to the approach: being careful
The official term for this is: **defensive programming**.

# Ah, yes, thems big words...
Basically, you just assume that all data is suss, & all the bad possibilities (such as module & subsystem crashes) can happen at any moment.

You, as a developer, assume that, and only THEN do you start touching code. “Defensive programming” is really just that, we don’t *need* to overcomplicate theories here. This blog is about pragmatism. Useless talk just to sound smart has no place here.

To help us understand how to be **defensive** & identify the consequences, let’s put on our imagination hat & assume we are developers on an autonomous trading system (mission critical). For simplicity, we’ll just pretend to freeze time, & put a magnifying glass over just 1 layer in just 1 subsystem.

Let’s say... a client-side user interface layer doing a simple HTTP AJAX call to 1 API endpoint, & the UI is used by an operator (sentinel role).

# Adventure time
When you think about it, a *lot* of suss things can happen between a client & server. That sexy boundary called the internet is a dangerous void. For instance:

- Xhr error (client side)
- Client and/or server Timeout
- All possible HTTP codes (including redirects)
- Auth issues
- Client-side middleware (e.g.: [Redux epics](https://redux-observable.js.org/docs/basics/Epics.html) failing)
- Dependency failures (client side)
- Client and/or server caching issues
- And of course, bugs!

But let’s slow down a bit, keep it simple, & just assume 1 of the factors listed above.
Lets be **defensive** by assuming that the API is contactable & it may return **500**, even if the API documentation doesn’t include a 500.

If our user interface did a simple AJAX call to 1 API in JavaScript via jQuery, it may look something like:
```javascript
$.post(someUrl, someData)
  .then(() => {…. });
```

If the API goes down, the promise will call `reject` internally. And gosh, we’re such brave programmers. Running naked here - no catch!

So we’re being defensive, great. The next step in the defensive programming approach, is to check the situation we’re knee-deep in. Just lean back in your chair, let out a sighful breath, and reflect. Our **CURRENT** situation is a client-side program, being used be an **operator**, that is contacting an API. As a **defensive programmer**, we should to notify the operator that the API is actually down, *right*?

You don’t want to leave the operator guessing or waiting, otherwise the trading algorithms could go unmonitored. And that presents significant financial risk to your firm. Software is hard, & risks will always be present & devs will always make mistakes. But dont intentionally make a mistake that can be cheaply avoided. That is a serious no-no.

## Defensive programming is mostly about complex, holistic self-reflection
Again, lets assess our situation. A financial mission-critical system with a human operator acting as a sentinel over trading algorithms. Now lets assume **all** data is suss, & **everything** possibly bad can happen.

> A productive way to defensively program is to think of the worst possible scenarios, then work backward to the "safer" scenarios. The worst scenarios have [low detectability, a high occurrence, & a high severity with significant consequences](https://en.wikipedia.org/wiki/Failure_mode_and_effects_analysis).

For instance, if a major market crash occurred (Japan, GFC, covid, commodities etc) & all of a sudden, our 1 API may experience a higher load. If there’s not enough company short-term funds (i.e.: pre-approved short-term liquidity) to elastically scale the backend to the very expensive levels necessary, the backend may get overloaded from all the data mining caused by the new information influx being posted on the internet (due to the financial crisis).

So in this case, the operator ABSOLUTELY NEEDS to know that somethings wrong, so **they** can then take appropriate action & keep the firm’s financial risk at bay during such an event.

## Ah mate, this is easy!
So as a defensive programmer, you’d simply catch the HTTP call… right?
```javascript
$.post(someUrl, someData)
  .then(() => {…. })
  .catch(() => console.error);
```

Nope, because whilst certain applications can show console-level errors, its usually hidden behind a menu. Our operator (who isn't a dev) probably wouldn't keep that pane open 24/7. Even if they did see the console message, they may not be technically capable to decrypting what the hell the message meant.

We thought we applied “defensive programming”. We thought we were smart! Oh how wrong we were, such foolishness. The humility is unbearable! Our thinking part is correct, but we just need to add a bit more sauce to **effectively** make use of defensive programming.

## Right, lets just show something in the UI then?
```javascript
$.post(someUrl, someData)
  .then(() => {…. })
  .catch(() => {
    console.error(…);
    toast.show(…);
  });
```

Nope, we’re still screwed here unfortunately. If the console error logging mechanism fails (e.g.: a weakly-typed referenced variable is passed in, but was then renamed), the toast doesn’t show. Keep in mind, this is a mission critical system where everything can turn to crap in a single millisecond.

```javascript
$.post(someUrl, someData)
  .then(() => {…. })
  .catch(() => {
    toast.show(…);
    console.error(…);
  });
```

There we go bois. Good defensive programming, rightttttttttt?

> Any failures in `toast.show` need to take a defensive programming approach. If it doesn't & the code is outside out of your control (e.g.: a third-party toast library), your mission-critical application should NOT use it. You absolutely need to find a **safe alternative**, or build one in-house. Otherwise you could be posing significant legal risk for your firm.

All things considered, this level of defensive programming is sufficient (for our very simple scenario of 1 HTTP Ajax call).

# Neato! I’m now gonna start being defensive. The words “Defensive programming” have a nice ring to it!
All things in software are contextual. If someone just says “lets do defensive programming” without actually describing a scenario, they probably need to have a closer look at the exact situation.

You can’t be defensive in all situations. It can be counterproductive when you’re NOT developing mission-critical applications. For instance, if you’re developing an internal-use-only library, the optimal approach may be **OFFENSIVE** programming, & defer additional handling to the consumers of your library. In this case, **DEFENSIVE** programming may actually *increase* schedule risk with negligible benefit (although at least you could say “I’m a defensive programmer, yay!”). Talk about tradeoffs, huh.

Analyse the specific situation at hand, use common sense, & take the right approach. Don’t just barge through the door like coolaid man & exclaim "we should be defensive in everything we do". Software engineering is complex & theoretical, so make sure your choice of approach makes sense before committing on it.
