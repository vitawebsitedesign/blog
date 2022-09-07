
---
title: Designing & programming an event-driven predictive model in Lua
layout: post
author: Michael Nguyen
---

This blog post details an 8-month journey in developing [predictive math algorithms](https://www.mathworks.com/discovery/predictive-modeling.html) in WoW & integrating them into the [TBC](https://wowpedia.fandom.com/wiki/Category:API_events) & [WeakAura](https://github.com/WeakAuras/WeakAuras2/wiki/API-Documentation) APIs. The WeakAuras integration requires deep knowledge of their API, often requiring experimentation to discover the full extent of their interface due to competing sources of incomplete & outdated documentation. 

Integrating with the 2 API’s also involved the [Lua programming language](https://www.lua.org/), and since the math we desire is predictive, it also required understanding the “hidden” math that drives the WoW product behind the scenes. 

For your convenience, these algorithms are now shared via [Wago](https://wago.io/t6on5CWbp) & you can also view this in [PDF format on my LinkedIn page](https://www.linkedin.com/in/michael-nguyen57/).

> ℹ️ Unlike my other posts which cover serious technical topics, such as Big O space & time complexity of various MS-SQL execution plans, this post is more casual.  It documents a problem that a programmer unexpecting discovers in their life outside of work, & realizes an opportunity to solve that problem using niche skills & knowledge outside of their usual domain of expertise. 

## The problem to solve
The shortest & over-simplified way to describe the problem I had was: 

“Given a specific encounter, what is the highest rank of [chain heal](https://tbc.wowhead.com/spell=1064/chain-heal) that I can use sustainably until the encounter ends” 

Since resto shamans suffer from severe mana issues, solving this problem would enable us to maximize [mana efficiency](https://wowwiki-archive.fandom.com/wiki/Healing_comparison#Introduction) in any dynamic encounter. 

To simplify this problem further, we can do an intentional (but invalid) assumption: more mana should always equals more healing output. If this were the case, solving the problem simply requires a math formula to ensure that mana usage remains **inversely correlated** to required output: 

![Mana efficiency comparison](https://i.imgur.com/P8LZ5TH.png "Mana efficiency comparison")
<sup>This diagram depicts mana usage (blue) & required output (red).  Ideally these remain inversely correlated regardless of encounter time.  During times 24-31, we can see that there is insufficient mana to meet output. In order to meet output, we ideally expend mana more efficiently during times 1-8, represented as the grey line. If we were to solve our problem reliably via predictive modelling, our expenditure (blue line) should be matching the grey line.</sup>

The above diagram illustrates the problem we are solving using a composite chart. If we look at real data (real charts), we begin understanding just how complex our problem really is. Below are non-composite charts, displaying real data that was [captured](https://classic.warcraftlogs.com/) during a Muru twins encounter.

![Mana level during a muru twins encounter](https://i.imgur.com/0c7l7BJ.png "Mana level during a muru twins encounter")
<sup>Mana level during the encounter (blue). Due to a large number of dynamic variables at play in different moments, mana expenditure rates differ throughout an encounter.</sup>

![Raid-wide damage taken](https://i.imgur.com/XFu2vy8.png "Raid-wide damage taken")
<sup>Healing output required (red). Due to damage being dealt non-uniformly during an encounter, some moments require low output whereas others require extremely high output. Ideally, mana is conserved during low-healing periods to then be spent when it’s needed the most. If we were to compare this chart vs our composite chart earlier on, a grey line (ideal mana expenditure) on this chart would always be inversely correlated to the red line.</sup>

![Healing output](https://i.imgur.com/tDHFdlx.png "Healing output")
<sup>Actual output - aqua, lime & red show different chain heal ranks being used.  
Due to mana being a scarce resource, various ranks need to be mixed together to sustainably meet required output throughout an encounter. It's difficult to identify the exact rank to use in each specific moment, which is the problem we aim to solve through our predictive modelling.</sup>

## Designing the predictive model
### Example scenario
To help understand our problem in more depth, we can run through a scenario:
> ℹ️ Real encounters are significantly more complicated. Hence, this upcoming first example is significantly over-simplified & “cut down” just to help illustrate the problem we are trying to solve. Later in this post, we will deep dive into developing the final formula that is more complex, accurate & pragmatic for a wide range of encounters.

*“Suppose we are in an encounter with a target having 100 remaining hit points. If the raid has 20 dps roles each outputting 0.5 damage per second, & assuming a current mana pool level of 1000 mana, can a chain heal rank $vn$ be sustainably casted for this situation?”*

A good first step to solving any math question is to identify the constants listed in the problem description. Then we can solve for $vn$:

| Constant                    | Value | Description                                                                                                                                                                                                                                                                                                                                                                                                                            |
|-----------------------------|-------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Remaining target hit points | 100   | - Simply put, the raid needs to output 100 damage total to end this encounter.                                                                                                                                                                                                                                                                                                                                                         |
| Number of dps roles         | 20    | - Number of roles in the raid that are specifically there to deal damage (instead of absorbing damage or healing).                                                                                                                                                                                                                                                                                                                     |
| Damage per dps raid role    | 0.5   | - Measured in seconds, this is the average output for each dps role. - Since this is a run-time variable that fluctuates throughout the course of an encounter, it therefore must be re-calculated at a high frequency to be reliably used as input to any predictive modelling.  - We will treat this variable as a constant just to simplify this first example, & we will dive deeper into event-based triggers later in this post. |
| Mana                        | 1000* | - Available mana to expend for chain heal                                                                                                                                                                                                                                                                                                                                                                                              |
| Chain heal cast mana cost   | 540   | - Mana cost to output 1 chain heal.  - Whilst this example uses a fixed constant of 540, the actual cost in real encounters is more complicated (we will dive into tidal focus & t6p4 modifiers' later in this post).                                                                                                                                                                                                                  |
| Chain heal cast time        | 2.5   | - Seconds required to output 1 chain heal.  - We will treat base cast as 2.5 seconds to simplify this first example, however the real cast time is more complicated (we will dive more into haste conversions later in this post).                                                                                                                                                                                                     |

*For simplicity in this first example, we are assuming the mana pool as static (rather not the effective mana pool). Effective mana pool needs to be considered instead to make our predictive model accurate & reliable. For sake of explanation, this example uses static, & we will dive into effective mana pool calculations later in this post)*

With our constants identified, we can solve for $vn$ to verify if it’s a viable rank of chain heal:
$vn =$ mana surplus at end $\geq 0$
$vn =$ casts required until end of encounter $-$ mana required for each cast $\geq 0$
where $m =$ static mana available
where $td =$ time to down
where $ct =$ chain heal cast time
where $cm =$ chain heal mana cost

$vn = m - \left( h \div (n . d) \over ct \right)cm \geq 0$
where $h =$ remaining health
where $n =$ number of raid members
where $d =$ damage per second for each raid member

$vn = 1000 - \left( 100 \div (20 \div 2) \over 2.5 \right)540 \geq 0$
$\therefore vn \lt 0 \implies vn$ is not a viable chain heal rank

### Problem solved? ...right?
Well, as I discovered during testing, not really. This equation only considers a subset of all factors that exist in real encounters. And for our predictive model to be pragmatic & applicable to a wider range of encounters, we have to get our hands dirty.

Additionally, the equation we’ve made so far makes all sorts of **incorrect assumptions**. To make the equation pragmatic for real encounters, we need to solve 2 more problems:

1. Real encounters involve significantly **more factors**, which change as the encounter moves along. For instance, one can expect to use [Mana Tide](https://tbc.wowhead.com/spell=16190/mana-tide-totem) once during an encounter – a certain talent that restores 24% of the max static mana pool. But it has a long cooldown & can usually only be used once. To consider this talent in our equation, it would need to consider mana tide when its **off** cooldown and also when it's **on** cooldown
2. We need to trigger the calculation based on an **event** (I.e.: integrate into WoW’s event-driven architecture). Having an event-based trigger is a **functional requirement**, due to this expensive formula needing to be re-calculated at a high frequency, & other factors leading us to having no other optimisation options.
    - We can’t use magnificent things like [lookup tables](https://www.mathworks.com/help/simulink/ug/about-lookup-table-blocks.html) due to the inputs being dynamic & ever-changing throughout an encounter
    - We can’t cache anything due to a sandbox technical restraint where variable scopes are severely limited when performing programmatic calculations or custom logic in WoW.

Let’s start with our 1st problem: identifying the other factors.

### Identifying all influencing factors
Due to this post already being too long for my liking, I'll jump straight to it - a couple weeks of soul-searching lead me to creating this diagram, containing our list of all “other factors” that a predictive model needs to be reliably applicable to a wide range of encounters.

Originally sketched out in pencil on the back of a scrap envelope, it has now been converted into a pristine diagram for your viewing pleasure:

![Overview of all influencing factors](https://i.imgur.com/w6JnP2t.png "Overview of all influencing factors")
<sup>All factors that need to be considered are listed in the middle, from “static mp5 data” to “cooldowns data”</sup>

**Static mp5 data**. Mana is actually restored over time, albeit at a very low & reduced rate (often referred to as the “casting mp5” rate). This is an equation in itself that involves additional sub-factors:
-   [Insightful Earthstorm Diamond](https://tbc.wowhead.com/spell=27521/mana-restore) (IED) - an item that has a chance of restoring a fixed amount of mana. More about this math in a second.
-   [Mana Tide](https://tbc.wowhead.com/spell=16190/mana-tide-totem) - a skill we already talked about that restores 24% of the max static mana pool
-   [Mana Potions](https://tbc.wowhead.com/item=22832/super-mana-potion) - consumables that restore mana. The amount restored is not fixed but rather a range & is therefore influenced by [randomness](https://en.wikipedia.org/wiki/Pseudorandomness).
-   [Water Shield](https://tbc.wowhead.com/spell=33736/water-shield) - a complex mechanic that modifies the “casting mp5” restoration rate. It also restores a fixed amount of mana based on a specific event (the caster needs to take damage to trigger this secondary mechanic), which also happens to have a hidden internal [GCD](https://wowpedia.fandom.com/wiki/Cooldown). Due to our encounters always involving long spans of time, we can thankfully consider this GCD impact negligible.

**Mana Tide** & **Mana Potions** can be assumed to have a fixed number of uses per encounter. Mana Tide oftens sits at 1 usage per encounter, whereas Mana Potions often sit at 2 usages per encounter. We will need to fetch this data somehow because our predictive model needs input that identifies whether these can be used at any moment during an encounter.

> ℹ️ When we consider all these additional factors, we are now calculating an "accurate" mana pool, compared to our over-simplified mana pool listed in the first example. This is referred to as the “effective mana pool”, which can be used as a reliable input to our predictive model.

### Expanding the equation
Previously, we developed an equation that only considered a subset of factors that exist in real encounters:
$vn = m - { h \div (n . d) \over ct } cm \geq 0$

The left-hand side basically asks the question: “how much mana surplus exists if a specific chain heal rank is casted continuously until end of the encounter”. But now that we’ve identified all influencing factors required to make our inputs reliable, we can now substitute these new inputs in lieu of $m$:

$vn = (mp + mf + mt + mi + mp) - { h \div (n . d) \over ct } cm \geq 0$
where $mp =$ current mana level
where $mf =$ static mana restored every 5 seconds
where $mt =$ absolute amount of mana restored from mana tide
where $mi =$ expected mana to be restored from ied
where $mp =$ mana restored from potion usage

Excellent. Now that we’ve expanded the equation, all ***high-level*** equation variables are now defined. We can now start decomposing the equation to identify all ***low-level*** equation variables. This allows us to identify any other remaining factors that are needed to implement our predictive model successfully, & to ensure we can actually source all of the input necessary for our model.

### Decomposing the equation (LHS)
$vn = mp + mf + mt + mi + mp$
$vn = mp + (mo . td) + (mmp . mtp) + \left( { 5 \over ct } . iedm . iedc \right) + { mpl + mph \over 2 }$
where $mp =$ current mana level
where $mo =$ static mana restored every **1** second
where $mmp =$ maximum mana level
where $mtp =$ percentage of maximum mana level restored from mana tide
where $iedm =$ mana restored from an IED proc
where $iedc =$ probability of IED proc
where $mpl =$ minimum mana potion restore
where $mph =$ maximum mana potion restore

### Decomposing the equation (RHS)
$vn = { td \over ct } cm$
$vn = { h_r \over dps } \div ct . m$
where $h_r =$ health remaining
where $dps =$ the raid's damage per second

$vn = h_r \div { d \over et } \div ct . cm$
where $d =$ the raid's damage so far as an absolute value
where $et =$ encounter time

$vn = h_r \div { h_x - h_c \over et } \div ct . cm$
where $h_x =$ target max health
where $h_c =$ target current health

$vn = h_r \div { h_x - h_c \over t_c - t_s } \div ct . cm$
where $t_c =$ current time in the encounter, expressed as unix timestamp
where $t_s =$ start time of the encounter, expressed as unix timestamp

### Re-composing the equation
With the left & right sides decomposed, we can re-compose the 2 sides to finalize our equation:
$vn = m_e - m_r \ge 0$
$m_e \implies$ effective mana
$m_r \implies$ required mana
where $m_e = mp + (mo . td) + (mmp . mtp) + \left( { 5 \over ct } iedm . iedc \right) + { mpl + mph \over 2 }$
where $m_r = h_r \div { h_x - h_c \over t_c - t_s } \div ct . cm$

$\therefore vn = \left( mp + (mo . td) + (mmp . mtp) + \left( { 5 \over ct } iedm . iedc \right) + { mpl + mph \over 2 } \right) - \left( h_r \div { h_x - h_c \over t_c - t_s } \div ct . cm \right) \ge 0 \implies$ chain heal rank $vn$ is viable

Where a positive number (greater than 0) indicates mana **surplus** (i.e.: the chain heal rank is viable), and a negative number indicates mana **deficit** (i.e.: not viable).

Fantastic. Now that we have our final formula, the remaining work is to substitute our new variables with constants.

### Researching the additional constants & run-time variables
Based on the above equation, we can identify the constant variables, & thankfully, their values can be researched using [the wowhead database](https://tbc.wowhead.com/):

|       Variable              |     Unit                              |     Constant `C` or Run-time `R` variable?  |     Value     |
|-----------------------------|---------------------------------------|---------------------------------------------|---------------|
|      $mp$                  |   Numeric                             |   `R`                                         |   n/a         |
|      $mo$                  |   Numeric                             |   `R`                                         |   n/a         |
|      $td$                  |   Seconds (expressed as a time span)  |   `R`                                         |   n/a         |
|      $mmp$                |   Numeric                             |   `R`                                         |   n/a         |
|      $mtp$                |   Percentage                          |   `C`**                                       |   0 to 0.24*  |
|      $ct$                  |   Numeric                             |   `C`**                                       |   TBD***      |
|      $iedm$              |   Numeric                             |   `C`                                         |   300         |
|      $iedc$              |   Percentage                          |   `C`****                                     |   .05         |
|      $mpl$                |   Numeric                             |   `C`                                         |   0 or 1800*  |
|      $mph$                |   Numeric                             |   `C`                                         |   0 or 3000*  |
|      $h_r$  |   Numeric                             |   `R`                                         |   n/a         |
|      $h_x$              |   Numeric                             |   `R`                                         |   n/a         |
|      $h_c$      |   Numeric                             |   `R`                                         |   n/a         |
|      $t_c$      |   Numeric                             |   `R`                                         |   n/a         |
|      $t_s$          |   Numeric                             |   `R`                                         |   n/a         |
|      $c_m$                  |   Numeric                             |   `C`                                         |   TBD*****    |

**Is 0 when item/spell already expended & not available. This means the variable functions as both a constant and a run-time variable.*
***The equation uses multiplication instead of addition for mana tide restoration, so whilst technically the mana amount restored is variable, it can be treated as a fixed constant in calculations*
****Calculating this run-time variable requires a bit more work, which we dive into the next section*
*****Encounters last a long-enough time span & hence; this can be reliably calculated using [expectancy](https://samuraitradingacademy.com/trading-expectancy/#:~:text=What%20is%20Trading%20Expectancy%3F,thirty%20to%20be%20statistically%20significant).), which is simply probability of proc chance multiplied by amount restored for one proc*
******Value depends on rank used*

After researching most of the variables, the remaining 2 fixed Variables we can try figure out are $c_m$ & $c_t$. Based on the wowhead database, the mana costs for each chain heal ranks ($c_m$) are:

|       Level for $vn$   |     Base value for  $c_m$   |     Actual value for  $c_m$*  |
|--------------------------|-----------------------------|--------------------------------|
|      $v_5$               |   [540](https://tbc.wowhead.com/spell=25423/chain-heal)                       |   459.00                       |
|      $v_4$               |   [435](https://tbc.wowhead.com/spell=25422/chain-heal)                       |   369.75                       |
|      $v_3$               |   [405](https://tbc.wowhead.com/spell=10623/chain-heal)                       |   344.25                       |
|      $v_2$               |   [315](https://tbc.wowhead.com/spell=10622/chain-heal)                       |   267.75                       |
|      $v_1$               |   [260](https://tbc.wowhead.com/spell=1064/chain-heal)                       |   221.00                       |
**85% reduction after [tier 6 4-piece bonus](https://tbc.wowhead.com/item-set=683/skyshatter-raiment#wh89j34tn97) & [5/5 tidal focus.](https://tbc.wowhead.com/spell=16217/tidal-focus) These “actual costs” are not displayed in-game in the tooltips, & instead must be manually calculated to obtain the correct values*

This gives us $c_m$, but we still need to solve for $ct$...

### Solving the hasted cast time formula
As of writing, tbc classic is at `2.5.4` (build `44171`).

This infers a [haste rating conversion rate](https://wowwiki-archive.fandom.com/wiki/Haste) of **15.77** inside of the game engine, which we were able to verify across multiple sources & also by comparing the modified cast time duration displayed on the in-game cast tooltips. Using this verified value, we can now use it to solve the adjusted cast time ($ct$):

$ct = {b \over e }$
where $b =$ base cast time
where $e =$ extra casts from additional haste

$ct = b \div \left( 1 + \left( { hr \over hc } \div 100 \right) \right)$
where $hr =$ runtime haste rating
where $hc =$ haste rating conversion rate

let $hr = 300$
$ct = 2.5 \div \left( 1 + \left( { 300 \over 15.77 } \div 100 \right) \right)$
$\therefore ct = 2.1004$ (4 dp)

...and that’s the last fixed variable!

### Yay or Nay?
We did it! We have successfully verified our predictive model as being implementable.

The remaining inputs are run-time variables which need to be fetched via either the TBC API or WA API. We can now start implementing this formula in code!

### Implementing the math equation
There are many different approaches to translating pseudocode into actual code.

For those fans of the [Code Complete](https://www.amazon.com/Code-Complete-Practical-Handbook-Construction/dp/0735619670) construction handbook, the ideal weapon of choice is often [the Pseudocode Programming Process](http://ps.informatik.uni-tuebingen.de/teaching/ss15/sct/StudentMaterial/09%20-%20Noah%20Doersing%20-%20pseudocode.pdf). And since we already have our list of inputs, the natural process is to start grouping the inputs into “purposes”, then each “purpose” gets translated into 1 method.

For reading convenience, the formula will be shown again below:
$\therefore vn = \left( mp + (mo . td) + (mmp . mtp) + \left( { 5 \over ct } iedm . iedc \right) + { mpl + mph \over 2 } \right) - \left( h_r \div { h_x - h_c \over t_c - t_s } \div ct . cm \right) \ge 0 \implies$ chain heal rank $vn$ is viable

|       Input                                               |     Purpose                              |
|-----------------------------------------------------------|------------------------------------------|
|      $vn$                                                |   get ideal chain heal rank              |
|      $ct$                                                |   get chain heal cast time               |
|      $td$                                                |   get time to down                       |
|      $h_r$ $h_x$ $h_c$ $t_c$ $t_s$ $cm$                  |   get mana required until target down    |
|      $mp$                                                |   get mana pool                          |
|      $mo$                                                |   get mana restoration from mp5 stat     |
|      $mmp$ $mtp$                                  |   get mana restoration from mana tide    |
|      $iedm$ $iedc$                              |   get mana restoration from ied gem      |
|      $mpl$ $mph$                                  |   get mana restoration from mana potion  |
If each table row is 1 method, we would then just need [a driver method](https://stackoverflow.com/questions/1451543/what-does-driver-program-mean#:~:text=A%20driver%20is%20generally%20a,outputting%20to%20CSV%20and%20HTML.) to call them in the right order. It doesn’t really matter what we call the driver method (aslong as it makes sense), & I eventually decided to call mine "get_ideal_chr".

This gives us a nice pseudocode skeleton to start filling in with actual code:
```
function get_ideal_chr() end
function get_cast_time() end
function get_ttd() end
function get_ttd_mana() end
function get_mana_pool() end
function get_mana_from_mp5() end
function get_mana_from_tide() end
function get_mana_from_ied() end
function get_mana_from_pot() end
```


## Developing the pseudocode and Lua
### get_ideal_chr
Firstly, we need a method that gives us the ideal chain heal rank. It essentially revolves around 2 main points:
-   ttd in seconds
-   mana cost until ttd is 0 seconds

Therefore, if there's enough mana, it is therefore a viable chain heal rank. My result of translating this into pseudocode is as follows:
```
begin get_ideal_chr 
  for each chain heal rank 
    calculate ttd in seconds 
    calculate mana required assuming ttd is 0 seconds 
    if available mana > mana required 
      return true 
    else 
      return false 
    endif 
  endfor 
end get_ideal_chr 
```

Based on our skeleton structured defined beforehand, we know that time-to-down method (get_ttd) will be part of our module too. This means we can accept it as a parameter instead of calculating it within this method for each chain heal rank.

With all the above, we can translate the pseudocode into Lua:
```lua
get_ideal_chr = function(chain_heals, chain_heals_count, cast_time, ttd, mana_pool) 
  -- ideal chain heal rank = max chain heal rank that can be used without going oom for an encounter 
  for chain_heals_index = 1, chain_heals_count do 
    local chain_heal = chain_heals[chain_heals_index]; 
    local ttd_mana = get_ttd_mana(ttd, cast_time, chain_heal.mana); 
    local can_use_rank = mana_pool - ttd_mana > 0; 
    if (can_use_rank or chains_heals_index == chain_heals_count) then 
      return { 
          rank = chain_heal.rank, 
          time_diff_formatted = get_time_diff_formatted(mana_pool, ttd_mana, chain_heal.mana) 
      }; 
    end 
  end 
end
```
Awesome. Next on the skeleton structure is **get_cast_time**.

### get_cast_time
Cast time in WoW is not a fixed number, and is affected by a characteristic called "haste", which is a positive integer that increases the number of casts that can be performed in a fixed time period:
```
begin get_cast_time 
  calculate base cast time 
  calculate spell haste rating 
  set haste rating conversion rate to 15.77 
  return base cast time / (1 + (spell haste rating / 15.77 / 100)) 
end get_cast_time 
```
Since the base cast time & haste rating conversion rate are fixed constants, we can simply convert them into parameters for this method.

One final factor to think about is that the spell haste rating is actually a **dynamic** variable which changes throughout the course of an encounter. For instance, different gear sets will result in different ratings, as well as buffs such as heroism, or procs from special trinkets like [Scarab of the Infinite Cycle](https://tbc.wowhead.com/item=28190/scarab-of-the-infinite-cycle). As such, this dynamic variable will need to come from the blizzard API and therefore it needs to be converted into a parameter.

With the above consideration, translating our pseudocode gives us the following Lua:
```lua
get_cast_time = function(base_cast_time, haste_rating, haste_rating_conversion_rate) 
  -- hasted cast time = baseCastTime / extraCastsFromHaste 
  return base_cast_time / (1 + (haste_rating / haste_rating_conversion_rate / 100)); 
end
```
Whilst a small function, it was incredibly important to spend the necessary time thinking to get this right. With the haste rating being a dynamic variable that changes during encounters, this method can return significantly different results, at a MINIMUM range difference of +/- 20%. This percentage is fed into other calculations which we'll get to soon, further creating inaccuracies & significantly impacting the final result to the point of the predictive model being ineffective.

With this out of the way, we move onto one of the most important methods: ttd.

### get_ttd
ttd = time to down. This is best explained through an example:
-   Assuming an encounter npc with `60,000/100,000` hp
-   Assuming an encounter start timestamp of `00:00:30`, with the current timestamp of `00:01:30`
-   We can therefore determine the damage per second (dps) by taking the missing health & dividing it by time spent in the encounter so far. Then given this dps, we can determine the remaining encounter time (ttd) as remaining health / damage per second. So for this example, it turns out to be `60000 / ((100000 - 60000) / 90)` which is 135 seconds

And given that this method contains almost solely of dynamic variables, we can write some approximate pseudocode to give us an idea of the Lua, which will definitely need a condition:
```
begin get_ttd 
  calculate encounter time 
  if encounter already finished then 
    return 1 
  endif 

  calculate damage per second 
  return remaining hp / damage per second 
end get_ttd
```
Some extra things to consider:

-   Encounter start time is not provided by the blizzard API, but rather the method we use to trigger the calculation. As stated earlier in this post, we have chosen WeakAuras as our calculation trigger, therefore we need to get the encounter start time from the WeakAura API as an input parameter.
-   In terms of time-to-down, the calculation may continue to run even after the encounter is finished. This method should gracefully handle this scenario
-   The hp stats are dynamic & change throughout an encounter. Therefore, they are dynamic variables & must be fetched using the blizzard API

Considering all the above, we can create the Lua equivalent:
```lua
get_ttd = function(aura_expiry_time) 
  -- time to down (in seconds) = remaining health / damage per second 
  local start_time = aura_expiry_time - aura_duration; 
  local combat_time = GetTime() - start_time; 
  if combat_time == 0 then return 1 end 

  local dmg_so_far = UnitHealthMax("focus") - UnitHealth("focus"); 
  local dps = dmg_so_far / combat_time; 
  return UnitHealth("focus") / dps;
end
```

Excellent. Now that we can calculate time-to-down in seconds, we can now move on to calculating the mana required (I.e.: assuming time-to-down reaches 0).

### get_ttd_mana
Since we calculated time-to-down in seconds, this one is thankfully fairly simple. We just need to calculate the number of chain heal casts until time-to-down, then multiply that by the mana cost for the given chain heal rank we are trying to calculate viability for (identified as $vn$ earlier in this post).

The final aspect to consider is to separate the static from the dynamic variables. In fact, time-to-down, cast time (calculated with spell haste rating) and chain heal rank mana cost are all dynamic variables!

The pseudocode turned out to be so short & self-explanatory, so let's jump straight into the juicy Lua:
```lua
get_ttd_mana = function(ttd, cast_time, chr_mana) 
  -- expected mana required until target downed = time to down / chain heal cast time * chain heal mana cost per cast 
  return ttd / cast_time * chr_mana; 
End 
```

### get_mana_pool
Aw yeah, this is the big one.

This is the method that the whole module revolves around, & as I discovered, it can get fairly complicated. In its simplest form, we need to know the "expected" mana available until time-to-down reaches 0.

This involves many factors, due to many mana restoration options available for resto shaman. In our specific usage scenarios, the variables are:

-   Current mana (at time of formulae trigger/calculation)
-   Current mp5. This is our passive mana regeneration
-   Mana restored from Mana Tide (24%)
-   Mana restoration expectancy from IED procs
-   Mana restoration from consumables

Given the above, we can now start the write the pseudocode:
```
begin get_mana_pool 
  calculate ttd 
  calculate chain heal cast time 
  calculate mana tide mana restore 
  calculate ied mana restore expectancy 
  calculate ied proc chance in percent 
  calculate mana potion restore 
end get_mana_pool
```
**All of the above are dynamic variables** & therefore need to be fetched using the TBC & WeakAura APIs. In addition, referring to our skeleton structure, a lot of our calculations have their own methods. Therefore, our Lua would just need to return the concatenated method return values:
```lua
get_mana_pool = function(ttd, cast_time, mana_tide_restore_pct, ied_proc_mana, ied_proc_chance_pct, mana_pot_min, mana_pot_max) 
  -- mana pool = the expected amount of remaining mana that is usable for an encounter 
  return UnitPower("player") 
  + get_mana_from_mp5(ttd) 
  + get_mana_from_tide(mana_tide_restore_pct) 
  + get_mana_from_ied(cast_time, ied_proc_mana, ied_proc_chance_pct) 
  + get_mana_from_pot(mana_pot_min, mana_pot_max); 
end
```
With our new driver method, we can now drill into each of these granular methods, which allow us to calculate the total mana available given time-to-down reaches 0.

### get_mana_from_mp5
MP5 is a passive & fixed metric, so we just need to convert the mp5 into mp1, then multiply that by time-to-down (which is also in seconds):
```lua
get_mana_from_mp5 = function(ttd)
  -- mana regenerated from mp5 = mana regenerated per second (while casting) * time to down seconds 
  local _, player_mp1 = GetManaRegen("player");
  return player_mp1 * ttd; 
end 
```

### get_mana_from_tide
As of time of writing this, mana tide restores exactly 24 percent of the base mana pool. Therefore, this method just returns that calculation.

One thing to consider is that mana tide can be on cooldown during an encounter, begging the question of how many mana tides can be used during a single encounter? Given a cooldown of 5 minutes, and SWP encounters lasting roughly 6-8 minutes, we can safely assume that only 1 mana tide can be used per encounter (including shorter encounters).

Therefore, our Lua method simply needs to assume 0 mana return when tide is on cooldown, otherwise return the calculation. Very simple & straight-forward, & we can jump straight into the Lua:
```lua
get_mana_from_tide = function(restore_pct) 
  -- mana restored from mana tide = max player mana * percentage of max player mana restored by mana tide 
  local _, cd, _ = GetSpellCooldown(16190); 
  if (cd == 0) then 
      return UnitPowerMax("player") * restore_pct; 
  end 
  return 0; 
end 
```

### get_mana_from_ied
["Expectancy"](https://invezz.com/invest/learn/trading-expectancy/) is a calculation theory that refers to determining an expected result occurring from a chance-based factor, over a large-enough sample size. The simplest way to explain expectancy is using an analogy such as a lotto ticket:
1.  Assuming a lotto ticket has a 1% chance of success
1.  Assuming a sample size of 50 (you scratch 50 lotto tickets)
1.  Given the above, what is the expected chance of winning the lotto?
1.  This would be: .01 * 50 = .5 = 50%
1.  Therefore, there is a 50% chance of winning the lotto

The above is obviously an over-simplified example, given our module. Basically, instead of lotto tickets, we are covering IED procs. And instead of sample size, we are covering chain heal casts until time-to-down reaches 0. On top of this, we are trying to calculate the IED proc chance plus the IED mana restoration expectancy.

With expectancy now explained, we can move onto developing the actual calculation itself. IED procs are triggered PER CAST. This means to calculate the mp5 restored from IED, we need to first calculate the number of chain heal casts per mp5 tick (every 5 seconds), then multiply that against the expected mana return if an IED were to successfully proc.
-   Number of chain heal casts per mp5 tick: `5 / cast_time`
-   Expected mana return if IED were to proc: `proc_mana * proc_chance_pct`
-   Therefore expected mana from equipping: `ied = (5 / cast_time) * (proc_mana * proc_chance_pct)`

Since all variables in the calculation are dynamic, we need to fetch them from the blizzard API and therefore they need to be developed as Lua parameters. With all the above finally considered, we can write the necessary Lua:
```lua
get_mana_from_ied = function(cast_time, proc_mana, proc_chance_pct) 
  -- expected mana from ied procs = number of chain heal casts per mp5 tick * expected ied mana restore per chain heal cast 
  return (5 / cast_time) * (proc_mana * proc_chance_pct);
end 
```
Geez, this is quite a lot to consider given the simplicity of the method, isn’t it?

If IED contributed little to the available mana given time-to-down reaches 0, then it would not be worth figuring out this equation. But since chain heal cast time is affected by spell haste rating, and with phase 5 offering an enormous amount of spell haste equipment, IED therefore contributes a notable amount of mana & therefore becomes an important consideration.

Thankfully for us, the last method in our skeleton structure is is a nice simple calculation to close off with.

### get_mana_from_pot
Mana potions have a bit of RNG, which restores mana in a range between X & Y. Important factors to consider are:
-   How many potions can be used for an encounter?
-   What should this method return when the item cooldown is active?

As of writing this, the cooldown is 2 minutes (120s). However, the cooldown is only activated if the current mana is less than or equal to the max amount of mana restored. In addition, different encounters last different durations, some are only 3 minutes, whereas others can last 8+ minutes. For simplicity, we can assume only 1 cooldown during the encounter, even if realistically 2 or 3 may be used during longer encounters (such as [Sunwell’s Brutallus](https://tbc.wowhead.com/guides/brutallus-sunwell-plateau-swp-strategy-burning-crusade-classic)).

The other factor to consider is what to return when cooldown is active. As I eventually figured out, this can simply be 0 - because as noted above, we can reliably assume only 1 active cooldown for the encounter (& most encounters only last long enough for 1 cooldown).

Lastly, the mana restoration amount is some number between a range of X & Y, & also depends on which type of mana potion is equipped. [Major mana potions](https://tbc.wowhead.com/spell=17580/major-mana-potion) restore a much lower amount than [their Super counterpart](https://tbc.wowhead.com/item=22832/super-mana-potion). Therefore, we want to keep X & Y as **dynamic** variables being passed into this method as input parameters.

With all the above considered, we simply take the average if cooldown non-active, otherwise 0 if active:
```lua
get_mana_from_pot = function(mana_pot_min, mana_pot_max) 
  -- expected pot restore per use = avg = (min restore + max restore) / 2 
  local _, cd, _ = GetItemCooldown(22832); 
  if (cd == 0) then 
    return (mana_pot_min + mana_pot_max) / 2; 
  end 
  return 0; 
end
```

With all the skeleton methods defined in Lua, we can now wire them all up! All we need now is just something to trigger the calculation, & as hinted earlier throughout this post, I chose to leverage WeakAuras to hook into WoW’s event-driven architecture.

## Finalities
### Triggering the Lua code with WeakAuras
In terms of the WeakAuras (WA) API, we only need it for the time-to-down calculation. The WA API offers “aura_expiry_time” & "aura_duration” as input parameters, which turns out to be exactly what we were looking for. This gives us the following trigger code:
```lua
function(aura_expiry_time, aura_duration)
  ...
end
```
Now we just need to place the methods inside of our trigger function.

The final factor to consider in Lua is whether we want to [forward declare methods](https://www.lua.org/pil/6.2.html) or not. In this instance, I chose to order the methods to require forward declaration, which allowed easier maintainability.

### Final Lua code
With our blizzard API variables, WeakAura API variables, constants, skeleton methods defined in Lua & our shiny trigger method, we can now connect it all together using forward method declarations, giving us the final Lua module:
```lua
function(aura_expiry_time, aura_duration) 
  local vars = { 
    base_cast_time = 2.5, 
    haste_rating = 296, 
    haste_rating_conversion_rate = 15.77, 
    chain_heals = { 
      { rank = 3, mana = 442 }, 
      { rank = 2, mana = 353 }, 
      { rank = 1, mana = 251 } 
    }, 
    chain_heals_count = 3, 
    ied = { 
      proc_chance_pct = .05, 
      proc_mana = 300 
    }, 
    mana_tide = { 
      restore_pct = .24 
    }, 
    mana_pot = { 
      min = 1800, 
      max = 3000 
    } 
  }; 
   
  local get_ideal_chr, 
  get_cast_time, 
  get_ttd, 
  get_ttd_mana, 
  get_mana_pool, 
  get_mana_from_mp5,  
  get_mana_from_tide, 
  get_mana_from_ied, 
  get_mana_from_pot, 
  get_time_diff_formatted, 
  get_time_diff_secs; 
   
  get_ideal_chr = function(chain_heals, chain_heals_count, cast_time, ttd, mana_pool) 
    -- ideal chain heal rank = max chain heal rank that can be used without going oom for an encounter 
    for chain_heals_index = 1, chain_heals_count do 
      local chain_heal = chain_heals[chain_heals_index]; 
      local ttd_mana = get_ttd_mana(ttd, cast_time, chain_heal.mana); 
      local can_use_rank = mana_pool - ttd_mana > 0; 
      if (can_use_rank or chains_heals_index == chain_heals_count) then 
        return { 
          rank = chain_heal.rank, 
          time_diff_formatted = get_time_diff_formatted(mana_pool, ttd_mana, chain_heal.mana) 
        }; 
      end 
    end 
  end 
   
  get_cast_time = function(base_cast_time, haste_rating, haste_rating_conversion_rate) 
    -- hasted cast time = baseCastTime / extraCastsFromHaste 
    return base_cast_time / (1 + (haste_rating / haste_rating_conversion_rate / 100)); 
  end 
   
  get_ttd = function(aura_expiry_time) 
    -- time to down (in seconds) = remaining health / damage per second 
    local start_time = aura_expiry_time - aura_duration; 
    local combat_time = GetTime() - start_time; 
    if combat_time == 0 then return 1 end 
      
    local dmg_so_far = UnitHealthMax("focus") - UnitHealth("focus"); 
    local dps = dmg_so_far / combat_time; 
    return UnitHealth("focus") / dps; 
  end 
   
  get_ttd_mana = function(ttd, cast_time, chr_mana) 
    -- expected mana required until target downed = time to down / chain heal cast time * chain heal mana cost per cast 
    return ttd / cast_time * chr_mana; 
  end 
   
  get_mana_pool = function(ttd, cast_time, mana_tide_restore_pct, ied_proc_mana, ied_proc_chance_pct, mana_pot_min, mana_pot_max) 
    -- mana pool = the expected amount of remaining mana that is usable for an encounter 
    return UnitPower("player") 
    + get_mana_from_mp5(ttd) 
    + get_mana_from_tide(mana_tide_restore_pct) 
    + get_mana_from_ied(cast_time, ied_proc_mana, ied_proc_chance_pct) 
    + get_mana_from_pot(mana_pot_min, mana_pot_max); 
  end 
   
  get_mana_from_mp5 = function(ttd) 
    -- mana regenerated from mp5 = mana regenerated per second (while casting) * time to down seconds 
    local _, player_mp1 = GetManaRegen("player"); 
    return player_mp1 * ttd; 
  end 
   
  get_mana_from_tide = function(restore_pct) 
    -- mana restored from mana tide = max player mana * percentage of max player mana restored by mana tide 
    local _, cd, _ = GetSpellCooldown(16190); 
    if (cd == 0) then 
        return UnitPowerMax("player") * restore_pct; 
    end 
    return 0; 
  end 
   
  get_mana_from_ied = function(cast_time, proc_mana, proc_chance_pct) 
    -- expected mana from ied procs = number of chain heal casts per mp5 tick * expected ied mana restore per chain heal cast 
    return (5 / cast_time) * (proc_mana * proc_chance_pct);         
  end 
   
  get_mana_from_pot = function(mana_pot_min, mana_pot_max) 
    -- expected pot restore per use = avg = (min restore + max restore) / 2 
    local _, cd, _ = GetItemCooldown(22832); 
    if (cd == 0) then 
        return (mana_pot_min + mana_pot_max) / 2; 
    end 
    return 0; 
  end 
   
  get_time_diff_formatted = function(mana_pool, ttd_mana, chr_mana) 
    -- time diff = seconds behind or ahead (based on current mana usage rate) 
    local time_diff_secs = get_time_diff_secs(mana_pool, ttd_mana, chr_mana); 
    local sign = nil; 
    if (time_diff_secs > 0) then 
        sign = "+"; 
    else 
        sign = ""; 
    end 
      
    return string.format("%s%.0f", sign, time_diff_secs); 
  end 
   
  get_time_diff_secs = function(mana_pool, ttd_mana, chr_mana) 
    -- current mana surplus or deficit = surplus mana / chain heal mana cost per cast 
    return (mana_pool - ttd_mana) / chr_mana; 
  end
end
```

### Bonus statistic: exact seconds until time to down (i.e.: how far ahead or behind the ideal rank will take us)
As a bonus piece of eye candy, we can also calculate the time differential, indicating how far our viable chain heal rank is ahead or behind, in seconds.

This is an ***incredible*** piece of useful information, & gives us a more accurate judgement on which rank to use. This is because whilst there are 3 chain heal ranks, often the most OPTIMAL rank is actually a mix of 2 ranks. For instance, 60% rank 4, 40% rank 5.

We can calculate the time delta as available mana given time-to-down reaches 0, divided by the mana cost of each chain heal cast:
$\Delta = {mmp - mttd \over cm}$
where $mttd =$ mana required for ttd to reach 0

With the equation defined, we can now write the Lua equivalent, & add the necessary code to drive the calculations upon trigger:
```lua
get_time_diff_formatted = function(mana_pool, ttd_mana, chr_mana) 
  -- time diff = seconds behind or ahead (based on current mana usage rate) 
  local time_diff_secs = get_time_diff_secs(mana_pool, ttd_mana, chr_mana); 
  local sign = nil; 
  if (time_diff_secs > 0) then 
    sign = "+"; 
  else 
    sign = ""; 
  end 

  return string.format("%s%.0f", sign, time_diff_secs);
end 


get_time_diff_secs = function(mana_pool, ttd_mana, chr_mana) 
  -- current mana surplus or deficit = surplus mana / chain heal mana cost per cast 
  return (mana_pool - ttd_mana) / chr_mana;
end 


if UnitHealth("focus") == 0 then 
  return nil, "..."; 

else 
  local cast_time = get_cast_time(vars.base_cast_time, vars.haste_rating, vars.haste_rating_conversion_rate); 
  local ttd = get_ttd(aura_expiry_time); 
  local mana_pool = get_mana_pool(ttd, cast_time, vars.mana_tide.restore_pct, vars.ied.proc_mana, vars.ied.proc_chance_pct, vars.mana_pot.min, vars.mana_pot.max); 
  local ideal_chr = get_ideal_chr(vars.chain_heals, vars.chain_heals_count, cast_time, ttd, mana_pool); 
  if (ideal_chr == nil) then 
    return nil, "?"; 
  end 
   
  return ideal_chr.rank, ideal_chr.time_diff_formatted;
end
```

### Real case studies
Finally, with our predictive model developed & kicking in action, a few (of the many) insightful moments that my formula provided are illustrated below.
![Illidari council](https://i.imgur.com/zwN1XgC.jpeg "Illidari council")
<sup>The Illidari Council, an encounter requiring high mental agility & capacity.<br>We can see the calculated viability in the bottom-left. This image dictates good mana efficiency management throughout the fight, with rank 1 chain heal being viable, & rank 2 chain being non-viable. The “-2” indicates a deficit of 2 seconds, meaning that a little bit of mana needs to be conserved at the current encounter stage. Since rank 1 is the lowest rank, this means that some casts need to be cancelled. Often healers require innervates due to difficulty in maintaining optimal mana efficiency during this encounter, but thankfully our formula allows us to easily maintain efficiency, thereby saving a critical innervate for another raid member.</sup>

![Illidan](https://i.imgur.com/A2xBw7t.jpeg "Illidan")
<sup>Illidan Stormrage, final boss of Black Temple.<br>This image illustrates rank 1 as the only viable rank, but also shows large “-107” second deficit, indicating a severe mana shortage at this stage of the fight. Since the formula also considers consumables, this deficiency highlights that over-healing was performed at an earlier stage of this encounter, & for a resto shaman, this is likely during Illidan’s aerial phase or too much single-target healing during the phase 2 fire elementals, which could have been deferred to a more efficient class such as a holy paladin.</sup>
![Reaver](https://i.imgur.com/RxGR7O3.jpeg "Reaver")
<sup>Fel Reaver in Tempest Keep.<br>An excellent use case for our formula, as the damage is linear throughout, thereby improving the value of our unique prediction-based formula. Here we see that whilst rank 1 is viable, rank 2 is shown as being possible & should therefore be used over rank 1 to maximize healing-per-second (HPS). We also see a surplus of only “+2” seconds, indicating near-perfect mana efficiency management.</sup>
![Kaelthas](https://i.imgur.com/qX60HoJ.jpeg "Kaelthas")
<sup>Kael’thas, final boss of Tempest Keep.<br>Another great use case for our formula, as the encounter rewards mana preservation during most moments, but extreme healing output during certain moments. Here we see our calculated rank viability in the bottom-left, indicating rank 1 as the only viable option.<br>Unfortunately, it also shows a very severe deficit of “-408” seconds, indicating that we won’t be able to sustain healing to the end of the encounter. Thanks to our formula, we are able to  identify this beforehand & communicate this mana restriction to our teammates ahead of time, which may involve getting a valuable innervate for an upcoming healing-intensive phase.</sup>

### Rank progression during code development of the formula
As we can expect, the predictive model significantly enhanced our healing performance over a very short period of time.

![Illidari Council progress](https://i.imgur.com/LocI3yj.png "Illidari Council progress")
<sup>Illidari Council progress</sup>
![Najentus progress](https://i.imgur.com/2Zljdcr.png "Najentus progress")
<sup>Najentus progress</sup>
![Kael’thas progress](https://i.imgur.com/WUtdaoZ.png "Kael’thas progress")
<sup>Kael’thas progress</sup>
![Void Reaver progress](https://i.imgur.com/GxObBOn.png "Void Reaver progress")
<sup>Void Reaver progress</sup>

## Conclusion
By leveraging the TBC & WA APIs, we were able to successfully implement a predictive model using math in a Lua module.

When I look back at these 8 months of iteration, it was full of excitement, exploration & experimentation. Sometimes the head scratching was too much some nights & I had to take a step away & just try again the next night after work.

But the simple land of Lua kept dragging me back in, & I simply can't get enough of the language's simplicity.

In software engineering, it's really hard to achieve a massive amount using only simple approaches & a small amount of code. Often the most difficult part of my life is writing less code to achieve more. Figuring out the math for this module was easy, but implementing it in the simplest & most optimized way was the real challenge that I enjoyed overcoming. As you can expect, the first iterations of the formula introduced latency & therefore drove the need for optimisation, which is what is listed in this blog post.

It was an iterate journey that was an absolute treat to develop, & whilst I'm sad that this journey has finished, I'm also proud of overcoming the logical & technical challenges it presented.