---
title: TDD aint a silver bullet - its a bronze bullet
layout: post
author: Michael Nguyen
---

You ever hear someone relay a quote from a famous person, apply it to a situation while disregarding context, & then discover they got burned? Example:
> *"It’s far better to buy a wonderful company at a fair price, than a fair company at a wonderful price"*

This is a Warren Buffet quote regarding "overinflated" share prices that dont justify their price-to-earning ratio.

In the modern era of trading (& especially market situations involving public participants), most of the money is actually made on speculative momentum-based short-term [trend](https://www.investopedia.com/terms/t/trendtrading.asp) plays involving incomplete information, rather than **"wonderful"** companies such as McDonalds:

---

[![Love their nuggets though](https://i.imgur.com/O6Cfk1T.png "Love their nuggets though")](https://i.imgur.com/O6Cfk1T.png)

----

[![One of the big winners for 2020. Good momentum-based short-term incomplete information play here](https://i.imgur.com/Rg9dRJB.png "One of the big winners for 2020. Good momentum-based short-term incomplete information play here")](https://i.imgur.com/Rg9dRJB.png)

---

The point: humans naturally want complex things to be simple, so that their cognitive load decreases. But certain things are complicated. And oversimplifying something can often hold negative effects.

And this applies to [TDD](http://agiledata.org/essays/tdd.html). TDD is fantastic, but there’s certain situations where it just doesn't cut the cheese.

# Unintentionally taking on unnecessary schedule risk
Most software companies (especially those with limited resources such as Start-ups), have certain exposure to something called [schedule risk](http://acqnotes.com/acqnote/tasks/schedule-risk). This is basically when time-to-market becomes a priority for the business.

In this case, anything that threatens time-to-market needs to be pushed down the priority list. And this includes **test coverage**.

It doesn't matter if its TDD, [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development), [ATDD](https://en.wikipedia.org/wiki/Acceptance_test%E2%80%93driven_development) or "tests after code". During right situations, priorities need to be aligned *correctly* to maximise project success. And in this case, doing TDD "just because that’s what good software engineers do" is harmful. Moreso because TDD has a higher upfront cost than tests-after-code.

# High requirements volatility
Depending on the company domain, certain products just have volatile requirements. It can't be avoided - its just the nature of the biz.

This is especially true for businesses building good [customer feedback loops](https://www.atlassian.com/company/events/summit-us/watch-sessions/2012/archives/scrum-kanban/building-an-effective-customer-feedback-loop). These are quick-turnaround, highly competitive markets, & involve multiple products continually try to out-beat each other & come on top of the pack. The cycle never ends.

In this case, TDD is *still* beneficial. But if the project doesn't already have decent test coverage, going from 0% to 50% can introduce **significant** risk for the product. Theres no point to TDD if the product needs to be disbanded because of business failure.

# "I thought i was a genius - my bad"
The software world is full of "[unknown unknowns](https://www.pmi.org/learning/library/characterizing-unknown-unknowns-6077)". Sometimes you just don’t have that squeaky-clean picture of the exact code & changes needed to get that meaty ticket to prod in fast, qualitiative & safe manner.

An engineer applying TDD in this scenario will encounter multiple rounds of TDD. They create an initial set of tests, then halfway through the ticket they discover additional information, need to change interfaces/semantics & need to update their tests again.

Depending on the number of "TDD rounds", this can be quite expensive. This is especially true for ambitious projects where requirements aren't clear. Perhaps the company is testing the waters with a new product, & they just want to see if it garners "interest" before investing additional resources into the project.

# Oh god, will the refactoring ever end?
In the end-game of TDD, test coverage is quite high, the release cycle gets fast & tight. Just the way we like it.

But test coverage doesn't ***have*** to be 100%. In fact, for very large & complex products in a company with limited resources, it’s better to leave out that other 10%. If the other uncovered 10% is low-impact features that you know isn’t gonna break, then the overhead of maintaining continually-breaking tests might not be worth the coverage benefit.

It’s good to *strive* to be perfect, just don’t *expect* perfect. Otherwise you'll overcommit to a practice, lose situational context & start inefficiently investing time & money.

# Eh, not a silver bullet. but bronze bullets are still good for most business products anyways.
Software is complicated & involves many scenarios. Use that brain & apply the right tool for the right job. TDD is great, just be sure that you're **winning** from it, not *losing* to it.

In the end of the day, the highest priority is business success, not the egotistic feeling of being a "good software engineer". Theres more to being a good software engineer than smashing out code. In fact, applying solid practices that maximize business success is often more valuable that being a code monkey.
