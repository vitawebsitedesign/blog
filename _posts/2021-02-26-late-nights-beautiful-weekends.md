---
title: Late nights, beautiful weekends
layout: post
author: Michael Nguyen
---

Reusable libraries are often created to reduce development workload.
They can also be used in an almost-infinite combination of scenarios (both current & future), which is a unique challenge.
To compensate, the library developers need to ensure the library ***consumers*** can actually use it **without** having to make unreasonable tradeoffs.

This means flexibility, which can be achieved via:
- Integrating with the consumer (e.g.: engaging in paid interop work between 2+ companies), or
- Intentionally making the library items simpler, & deferring extra functionality/options to the consumer (e.g.: Facebook has done a fantastic job of Reacts' [composition vs inheritance](https://reactjs.org/docs/composition-vs-inheritance.html))

# Simple example
Take for example, a reusable UI library. Lets say this library has a simple button component, that needs to be used across 3 very similar (but slightly different) products, with 3 different situations per product. The button also has hard-coded (magic numbers) to define spacing around & within it:

> 9 scenarios for a reusable component isn't alot, but its good to oversimplify the situation for example purposes

<pic>

As we can see above, the button can be reused fine in 1 situation, but the other 8 situations are a pain in the ass. Due to the hard-coded external & internal spacing, consumers are forced to apply hacky workarounds that severly deplete the [technical debt capacity](https://www.oreilly.com/library/view/the-pragmatic-programmer/9780135956977/f_0020.xhtml) of the entire company (which means that projects not using this library are still negatively impacted via staff availability):

<pic>

If consumers start facing serious business or techincal issues, the library obviously has a critical design flaw. It needs to be addressed at some point, before it spirals out of control. If you caught onto the design issue early, good on you. And if you have the time to fix it today, then even better.

But ignoring the importance of design flaws like this can lead to...

# Absolute disaster!
So what happens if critical design flaws are misunderstood & treated as "not important"?
### 1. Staff cant do their job to the best of their ability
Developers just end up creating their own reusable library, usually as a Skunkworks project, & start migrating over to it. Often, they are just frustrated by the business expense & take things into their own hands. Good developers dont put up with crap software, & will take initiative & do this. It can turn out surpisingly well (ignoring the business expense, stress from late nights & potential turnover). Then the old reusable library just gets left in the dust because it becomes an unattractice alternative during project planning.

Example: if a front-end "shared" component library requires designers to **change** their workflow, the time "saved" by developers using the resuable library **is actually just shifted** onto the designers. In this case, the reusable library does NOT actually save effort, because it just shifts extra work to someone else). From the library developers perspective, it saves time. From everyone elses perspective, its an ***absolute catastrophe***! The responsibility lies on the engineers in this case, which intentionally left design flaws in place for their own gain. An engineer using that approach is unfortunately unintentionally double-crossing their colleagues, although hopefully they eventually realize the impact of their design decision & turn the situation around.

### 2. The technical debt eventually become too visible, & starts hindering development agility to the point where all current projects get dropped
Put simply, the reusable library gets thrown out & redone:
- Re-architect
- Re-design
- Re-write
- Re-integrate
- Re-test
- Re-deploy

Plus all those meetings, turnover, ad-hoc dev chats & the inevitable political pressure from upper management. Hmm oh yeah i do love it when I have to stay back at 10pm on weekends, while getting screamed at upper management due to someone elses intentional negligience. I'm sure many good developers have been in this situation, & we all know those lost weekends will never be re-gained back :(

# "There's only a few problems so far, so lets keep going in this direction!"
Its important to think long & hard on design decisions that can be potentially disastrously expensive in the future.

An approach of *intentionally* pushing design flaws to the side because "it makes my life easier" isn't an acceptable approach. Companies rely on developers to be responsible & to act in the best interest of the company. Software is **hard**. Engineers make mistakes **all** the time. That is fine, it happens. I make mistakes too.

But ***intentionally making mistakes*** isn't exactly the best approach to development, ESPECIALLY if it comes at the expense of your team, & all the other colleagues that rely on you.
