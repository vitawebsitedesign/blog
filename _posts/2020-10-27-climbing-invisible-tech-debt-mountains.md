---
title: Climbing invisible tech debt mountains
layout: post
author: Michael Nguyen
---

We need 10% time, but startups don’t work that way. In fact, startups are bloody hard.

10% time is a strategic initiative where software teams get dedicated “playtime” to reduce [technical debt](https://en.wikipedia.org/wiki/Technical_debt) (i.e.: stuff that is a pain in the ass).

Think of unnecessary duplicated code that makes it really expensive & hard to maintain. Think of legacy tooling that just doesn’t have the productivity features of [modern tools](https://www.jetbrains.com). Think of performance issues that lie in spaghetti code.

Think of the those times you sketched data flow of API's on paper. You looked at it, & questioned the meaning of life.

[![1854 - Leonardo Da Vinci's remastered renaissance artwork of The Last Supper](https://i.imgur.com/VnM6qbm.jpeg "1854 - Leonardo Da Vinci's remastered renaissance artwork of The Last Supper")](https://i.imgur.com/VnM6qbm.jpeg)

These are all examples of technical debt, **and that’s ok**. And you know what? Startups often have *very high* levels of technical debt, a tradeoff between software team morale & satisfying stakeholders with flashy products that sell like hotcakes.

But even though they have the most technical debt, they are the least likely to introduce measures to manage it. And eventually the startup (more specifically, the software engineers) hit a brick wall - a big (no… HUGE) mountain of tech debt, and they stare at the mountain, and feel the fear. It makes them consider rewriting code from scratch, which is usually what happens anyway since its cheaper than fixing legacy code.

And then management comes around. They simply won’t allow “playtime” to fix issues. It's just acknowledged & pushed to the side. “We need feature X Y Z by yesterday”. And this is ok - but what can software engineers do in this situation?

How do we manage technical debt when 0 time is dedicated to it?

# Lets find that missing puzzle piece...
## Solution 1: just refactor more!…. right?
An often approach is to trade off [sprint velocity](https://www.scruminc.com/velocity/) for non-visible technical debt reduction. This is usually either approved by the managers (e.g.: the product owner may approve because it's a smaller overhead than rewriting entire codebases), or not approved by managers. If it's **NOT** approved, good software engineers often take the initiative themselves, knowing that even though it's not visible by management, technical debt kills.

This is equivalent to a chef cleaning their table to prepare for the next dish.

## Solution 2: moonlighting
When not approved, software engineers can slip in some tech debt refactoring. This can happen during work hours (e.g.: 30 mins during a 9-5 window some days) or outside work hours (e.g.: day ends at `5pm`, they stay back & refactor to `5:30 - 6:00pm`).

This introduces a problem of burnout, but often if its:
- non-mandatory
- the initiative is **self-motivated** (e.g.: something is really itching them & fixing the tech debt brings them more happiness than spending the 30 mins at home watching Netflix)
- they are working on *“their baby”* (i.e.: the project is something that they deeply care about, they gave birth to the repo, & they are the kings/queens of that domain)

Then it's often a good approach. The software engineer is happy, their colleagues are happy, & managers are usually happy to let someone overwork a bit. **EVERYONES HAPPY!** Additionally, good managers will know of [the Dead Sea effect](https://en.everybodywiki.com/Dead_Sea_Effect) & the importance of turnover. They will approve overworking, but also drop a hint to them after slapping their back: “…but make sure to not burn out”.

## Solution 3: out with the old, in with the new!
Sometimes the tech debt is just so bad that its actually cheaper to just rewrite the entire repo(s) from scratch.

Considered a high-risk low-reward move, its difficult to justify this option to management. Often bad engineers just do it without approval, expecting a big congratulations upon presenting their work to their colleagues. BACK SLAPS ALL AROUND.

Unfortunately getting rewrites to replace the existing test & prod instances is complicated, & usually rewrites don’t have good test coverage (*naughty word*).

[![Software decisions is all about prioritising tradeoffs. Free lunches are welcome, but rare](https://i.imgur.com/vcia2Ny.jpg "Software decisions is all about prioritising tradeoffs. Free lunches are welcome, but rare")](https://i.imgur.com/vcia2Ny.jpg)

I’ve seen this work before, but the incredible expense is always an important factor to consider before creating those Jira epics.

Expense, with a capital "E".

## Solution 4: the dead sea effect
When all 3 above approaches fail, software engineers get low morale. Its like world war, & the engineers are in the trenches. They got no clean clothes, there’s bullets flying over their head on a daily basis, there’s not much food available (& what is available is infested with vermin). Oh god no not again!

[![We've all been like this at some stage in our lives. But a company that intentionally does this will have impressive amounts of turnover](https://i.imgur.com/R84oL6o.jpeg "We've all been like this at some stage in our lives. But a company that intentionally does this will have impressive amounts of turnover")](https://i.imgur.com/R84oL6o.jpeg)

It's just a terrible situation to be in. And it often results in turnover.

And of course, seeing comrades leave makes you question whether you should stay or step up to the next level in your life.

[![The Dead Sea effect doubles as a domino effect. 1 engineers leaves, then 2, then 4](https://i.imgur.com/xgHt9QN.jpeg "The Dead Sea effect doubles as a domino effect. 1 engineers leaves, then 2, then 4")](https://i.imgur.com/xgHt9QN.jpeg)

This is the end-game that all startups risk when they simply ignore technical debt, pretend that it doesn’t exist, & just whip software engineers even when they’re down. Often to stop the incredible turnover, they have to increase monetary compensation. And even then, thats not guaranteed to solve the problem.

Often, happiness is more preferred than money. In other words, a good workplace (daily happiness) is more important than compensation, aslong as the compensation is “decent” enough for the individuals standard of living & financial/tangible goals.

# What is the secret sauce?
If you’re at ground zero, solutions 1 & 2 are the good choices.

You can refactor more by changing your approach to each ticket. For example, before opening each PR you review it yourself & do a final “pass” to cleanup any remaining code. You can refactor code that’s not related to your ticket, whilst still staying out of [the rabbit hole traps](https://www.dictionary.com/e/slang/rabbit-hole/) that often come out of the tall grass & snatch you.

You can also moonlight. Even though a lot of software companies have a dedicated work window (e.g.: `9am to 5pm`, `8am to 6pm`), software engineers often perform additional work. Weekend production issues, late night team releases, early morning “all hands on deck” situations for critical releases the previous night. And since moonlighting is a self-initiative on your own “baby” (repos), its easy to be motivated to make it nice & pretty.

Startups have the **0 time, we want everything** problem. Theres just not enough time to do *everything* (even if you increase headcount). In the end, there are some good approaches to tackling tech debt, & whilst the 3 solutions above don’t suit all situations, there should be at least 1 that is correct, efficient, effective & a perfect match for your depression.

I wish you the best of luck, you'll need it.

