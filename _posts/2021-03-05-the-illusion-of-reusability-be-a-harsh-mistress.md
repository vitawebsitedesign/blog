---
title: The illusion of reusability be a harsh mistress
layout: post
author: Michael Nguyen
toc: true
---

Reusable libraries are often created to reduce development workload across multiple teams.

Reusable items in the library can be used in an *almost-infinite* combination of scenarios (both currently known & possible new future scenarios), which presents a unique challenge - **how do we design to accommodate for all scenarios?**

And on top of the scenario challenge, devs also need to ensure the library ***consumers*** can actually use it **without** having to make unreasonable tradeoffs. **Flexibility**. That way it presents an attractable alternative in project plans.

To solve both the scenario & flexibility challenges, library devs often intentionally make the reusable items simpler, & defer extra behaviour/options to the consumer (e.g.: Facebook has done a fantastic job of Reacts' [composition vs inheritance](https://reactjs.org/docs/composition-vs-inheritance.html)).

This post explores an example of reusable library development hell - a reusable front-end component library.

# Simple example
You are part of the dev team that needs to deliver 1 component - a button:

```html
<button>My button</button>
```

[![Button](https://i.imgur.com/K9FWMzI.png "Button")](https://i.imgur.com/K9FWMzI.png)

No requirement for padding or margin yet.

The devs huddle around, & decide to maximize consumer dev speed by hard-coding options. In this case, the devs (us/we) have decided to hard-code padding & margin:

[![Button with padding and margin](https://i.imgur.com/iBvJ0fP.png "Button with padding and margin")](https://i.imgur.com/iBvJ0fP.png)

- Green = padding
- Orange = margin

Now the consumers can just copy-paste the component, right? They dont even ***need*** to set padding or margin themselves - what a time saver!
At the moment, it only needs to be used in `1` product with `3` specific scenarios.
We cross-check with wireframes The UI team sent us, just to ensure our button can be re-used in the necessary scenarios:

---
[![Scenario](https://i.imgur.com/9pjAYZA.png "Scenario")](https://i.imgur.com/9pjAYZA.png)

`Wrapper elements beside & below our button`

---
[![Scenario](https://i.imgur.com/yyVI0Lf.png "Scenario")](https://i.imgur.com/yyVI0Lf.png)

`Wrapper element beside our button`

---
[![Scenario](https://i.imgur.com/EbHgo6e.png "Scenario")](https://i.imgur.com/EbHgo6e.png)

`2 wrapper elements above our button, with vertical spacing between (to be consistent with the bottom margin of our button)`

---

It seems our consumers can use our reusable button in all 3 scenarios.
- We deliver the component library.
- The consumers copy-paste the button in their 1 product.
- Our component, ***great success!*** Pop the champagne!

No probs with the hard-coding the margin or padding.

In fact... the UI team praises how easy it was to use & how much time it saved them. Salary raises for all!

Fast forward a year. Now the company wants to ship a new product. Its competing against a potential competitor product, so time-to-market for this new product is *critical*. The UI & our (dev) teams choose our fancy reusable library (with our button) to speed things up. We just had a pleasant experience with it, why not?. 


Once again, the UI team sends us wireframes, & we cross-verify them to ensure our reusable button is still re-usable.
Analysis finds that our reusable button now needs to work with 1 **additional** product, containing 3 **NEW** scenarios:

---
[![Scenario](https://i.imgur.com/Hofcc2s.png "Scenario")](https://i.imgur.com/Hofcc2s.png)

`Button is used inside an element with padding. Due to both the buttons bottom margin & wrapper bottom padding, we have this weird extra gap between the button & the bottom of the wrapper.`

---
[![Scenario](https://i.imgur.com/I72leYH.png "Scenario")](https://i.imgur.com/I72leYH.png)

`Our "reusable" button is re-used twice & placed side-by-side. There is 2x the expected space between them.`

---
[![Scenario](https://i.imgur.com/3BvZ67U.png "Scenario")](https://i.imgur.com/3BvZ67U.png)

`The consumer wants to have a slightly thinner button (a real-world example of this is using a re-usable button with different inner text). Since the padding & margin is hard-coded, we see our button with larger-than-life spacing. "Way too much whitespace" the UI team says.`

---

Ah crap. Our button can't be re-used in these 3 new scenarios!

Eventually, the UI team gives feedback about not being able to use our so-called "reusable" component in the new concepts properly. Something about "ugly & inconsistent spacing".

As we can see above, the button can be reused fine in the 3 original situations, but the extra 3 situations are a pain in the ass. Due to us hard-coding external & internal spacing (in an attempt to improve the reusability of the button), our consumers are forced to apply hacky workarounds that severely deplete the [technical debt capacity](https://www.oreilly.com/library/view/the-pragmatic-programmer/9780135956977/f_0020.xhtml) of the entire company. Even projects not using this library may still be negatively impacted, especially if they need the staff resources which are now blocked on **our** current button issue.

> When a reusable library cannot be reasonably reused, its possible that using it poses more downside than upside, thus making it **not** reusable.

If our consumers cant use our reusable button in a pleasing way, will they:
- Continue using the library going forward into future products?
- Is it adding tech debt each time they use it? Are there complaints in the UI team that isn't being sent to us? (the dev team)
- Will the risk of company turnover in the UI department now increase due to our flaw?
- If someone chooses to use our library in their product, does it increase their schedule risk?
- Is our library actually a blocker? Perhaps this flaw is just getting started...
- If a consumer uses our library, is the cost worse than the potential benefit?
- And many other questions, including questioning the technical intelligence of our team.

As you can see, this is a **very critical** design flaw. Imagine a structural engineer designing a skyscraper, but it has a design flaw, such as [misjudging the structural integrity of the base](https://news.stanford.edu/pr/01/wtcpostmortem125.html), exposing it to potential collapse & making it inhabitable. In this case, the consumers (inhabitants) won't be able to inhabit the building...

# Illusions
Depending on the type of mission-critical software you develop, the structural analogy above may be understating or overexaggerated. However the analogy still stands - our library design flaw definitely has a serious flaw.

We tried our best & attempted to maximize reusability by allowing consumers to copy-paste our component and have it work "out of the box". Just like a plug-n-play Xbox controller with a PC.

We maximized reusability initially, but it was all an illusion. Our approach started backfiring when the 2nd product had to be supported. The problem, you see, is that **our approach to reusability design isn't scalable**. And because of this unforeseen matter, we now need to re-write our component, re-deploy, re-test both products etc.

It gets even worse. In reality, reusable components are used in more than `2` products (`6` scenarios). Developing successful reusable libraries is hard work, a tricky mistress. And we just created a prison without knowing it, & our team is now trapped inside, chewing on stale moldy bread & banging empty metal mugs against prison railings.

# Prison break
If you recall, the 2 *challenges* our team had to overcome were:
1. Support all necessary **scenarios** (current known future ones, as well as future unknown scenarios)
1. Support consumers so that they can work at their best **potential** (i.e.: flexibility + productivity + creativity)

And we tried to overcome both challenges by hard-coding spacing.
But due to our flawed approach to reusability design, we failed to overcome both challenges.
It is, however, indeed possible to overcome both challenges **SCALABLY**.

# Brilliance encased in simplicity
Think of [Bootstrap](https://getbootstrap.com). It is the epitome of reusable component web libraries (excluding native platform SDK's).
The [components](https://getbootstrap.com/2.3.2/components.html) the library provides are super-simple. In fact, the components are so simple, it sometimes gets questionnable.

They're so.... simple, right? They're just basic wrappers around HTML elements with a tiny CSS class slapped on to provide a default styling "out of the box". Stupid? No, not stupid. In fact, the simplicity is ABSOLUTELY **brilliant**.

> Challenge #1: "Support all necessary scenarios (current known future ones, as well as future unknown scenarios)"

By having stupidly simple components, Bootstrap components are so flexible that they can be used in the significant majority of scenarios.

> Challenge #2: "Support consumers so that they can work at their best potential (i.e.: flexibility + productivity + creativity)"

And since the html structure & styling is simple, consumers can apply their own styling to achieve their needs quickly. Need a different colour? A different font style? Different spacing? Different border? Different box sizing model? Bam, easy peasy.

Judging by the significant commercial adoption of Bootstrap, i'd say the Facebook devs used the **CORRECT** approach in designing reusable libraries, a *truly* scalable approach!

# I didn't hear no bell
Us & our team have been forced into a difficult situation - we now NEED to rewrite our reusable UI button to unblock our UI team, so that our company can beat the competitor to the market:
- With this new ***scalable*** reusability design approach in mind, lets approach the button design differently.
- Instead, we will make our button simpler, & defer extra options to consumers.

With this approach, we are hoping to overcome the 2 challenges in an inexpensive manner.

Again, lets revisit the 3 scenarios our button received complaints about:

---
[![Scenario](https://i.imgur.com/Hofcc2s.png "Scenario")](https://i.imgur.com/Hofcc2s.png)

Unscalable approach: `Button is used inside an element with padding. Due to both the buttons bottom margin & wrapper bottom padding, we have this weird extra gap between the button & the bottom of the wrapper.`

Scalable approach: our button **no longer** has extra bottom padding. The consumer has to write some extra lines of CSS, but they can use the button the exact way they want. They can now ACTUALLY USE our button!

---
[![Scenario](https://i.imgur.com/I72leYH.png "Scenario")](https://i.imgur.com/I72leYH.png)

Unscalable approach: `Our "reusable" button is re-used twice & placed side-by-side. There is 2x the expected space between them.`

Scalable approach: our button **no longer** specifies margin. The buttons neatly sit side-by-side & the UI team is relieved

---
[![Scenario](https://i.imgur.com/3BvZ67U.png "Scenario")](https://i.imgur.com/3BvZ67U.png)

Unscalable approach: `The consumer wants to have a slightly thinner button (a real-world example of this is using a re-usable button with different inner text). Since the padding & margin is hard-coded, we see our button with larger-than-life spacing. "Way too much whitespace" the UI team says.`

Scalable approach: our button **no longer** specifies padding or margin. The consumer is able to decrease the button width, & is able to define the button's padding & margin in several ways to achieve their exact vision (e.g.: percentage of REM based on the button's wrapper width)

---

Pats on the back for everyone. The consumers need to write **a bit of extra code**, but that overhead is so tiny considering the significant productivity advantage our "new" button provides. Now *THATS* a smart tradeoff. What a cracker!

We were able to apply our new scalable approach to reusability & overcome both challenges. Thank god!

# "Whew, that was a close one!"
If consumers start facing serious business or technical issues, the library obviously has a critical design flaw. It needs to be addressed at some point, before it spirals out of control. If you caught onto the design issue early, good on you. And if you have the time to fix it today, then even better.

But ignoring the importance of design flaws like this can lead to...

# Absolute disaster!
So what happens if critical design flaws are misunderstood & treated as "not important"?
### 1. Staff cant do their job to the best of their ability
Developers just end up creating their own reusable library, usually as a Skunkworks project, & start migrating over to it. Often, they are just frustrated by the business expense & take things into their own hands. Good developers dont put up with crap software, & will take initiative & do this.

It can turn out surprisingly well (ignoring the business expense, stress from late nights & potential turnover). Then the old reusable library just gets left in the dust because it becomes an unattractive alternative during project planning.

Example: if a front-end "shared" component library requires designers to **change** their workflow, the time "saved" by developers using the reusable library **is actually just shifted** onto the designers. In this case, the reusable library does NOT actually save effort (because it just shifts extra work to someone else). From the library developers perspective, it saves time. From everyone elses perspective, its an ***absolute catastrophe***! The responsibility unfortunately lies on the engineers in this case, which intentionally put those design flaws in place & let it be.

### 2. The technical debt eventually become too visible, & starts hindering development agility to the point where all current projects get dropped
Put simply, the reusable library gets thrown out & redone:
- Re-architect
- Re-design
- Re-write
- Re-integrate
- Re-test
- Re-deploy

Plus all those:
- meetings
- turnover
- ad-hoc dev chats
- the inevitable political pressure from upper management.

# Conclusion

> "There's only a few problems so far, so lets keep going in this direction!"

If you ever hear this, keep in mind that its important to think long & hard on design decisions that can be potentially expensive in the future.

An approach of *intentionally* pushing design flaws to the side because "it makes my life easier" isn't an acceptable approach. Companies rely on developers to be responsible & to act in the best interest of the company...
- Software is **hard**.
- Engineers make mistakes **all** the time. 
- That is fine, it happens. I make mistakes too.

But ***intentionally*** making mistakes isn't a responsible approach to development, ESPECIALLY if it comes at the expense of other teams, your company, & all your other colleagues that rely & put faith in you.
