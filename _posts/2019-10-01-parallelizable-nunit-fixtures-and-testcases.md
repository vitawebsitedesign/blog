---
title: Exploring the parallel options in the nUnit execution framework
layout: post
author: Michael Nguyen
---
Every build server has a cpu limit, and once you hit it you'll find your nUnit tests taking forever to finish, ultimately slowing down your build process.

Thankfully, we can run nUnit tests in parallel by using the `Parallelizable` attribute. This tells the runner to run "stuff" in parallel mode.

I use the word "stuff", because theres multiple "levels" (i'll get to this later) we can configure, using [4 different parallel modes](https://github.com/nunit/docs/wiki/Parallelizable-Attribute) to choose from.

And hence, this can get confusing fast. I'll explain this through examples, which are listed in a different order than the documentation in order to aid understandability.

Before starting this party, the below examples use the Visual Studio nUnit runner (rather than the CLI runner). This is an important fact because:

> "Parallel execution is [the default behavior](https://github.com/nunit/docs/wiki/Engine-Parallel-Test-Execution) when running multiple assemblies together using the nunit3-console runner"

And since we aren't using the CLI tool to run our tests, then:

> "By [default](https://github.com/nunit/docs/wiki/Framework-Parallel-Test-Execution), no parallel execution takes place"

Note: in order to understand all of this, try think of parallelable tests as 2 layers - your test cases (i.e.: the functions), and your fixtures (i.e.: the classes that hold the test cases).

## Parallelizable
The `Parallelizable` C# attribute is something you can put on test fixture classes and/or test case functions. It makes it run in parallel on a separate, new thread. Example:

```c#
[Parallelizable()]
```

It takes in 1 argument, which happens to be an enumeration that allows us to use 4 possible "parallel modes". And the first one we will look at is `ParallelScope.Children`, which is usually used on test fixtures (the class that holds your test functions). Example:
```c#
[Parallelizable(ParallelScope.Children)]
```

## ParallelScope.Children
The official [nUnit description](https://github.com/nunit/docs/wiki/Parallelizable-Attribute#user-content-parallelscope-enumeration) for this "parallel mode" is:

> "child tests may be run in parallel with one another"

In english: test cases in a fixture will run in parallel, but the fixture itself will NOT run in parallel with other fixtures`

For example, given a test case:
```c#
[Test]
public async Task MyTestCase()
{
	...
}
```

A test fixture for this test case would be a class:
```c#
internal class MyFixture
{
	[Test]
	public async Task MyTestCase()
	{
		...
	}
}
```

By using `ParallelScope.Children`, we make test cases inside the fixture run in parallel:
```c#
[Parallelizable(ParallelScope.Children)]
internal class MyFixture1
{
	[Test]
	public async Task A()
	{
		TestContext.Progress.WriteLine("a start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("a end");
	}
	
	[Test]
	public async Task B()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("b start");
		TestContext.Progress.WriteLine("b end");
	}
}
```

In this example, we expect A to start first, then after 3 seconds, B starts too. Running this fixture prints out:
``` 
a start
b start
b end
a end
```

And this output proves that they run in parallel.

To prove that the fixture does NOT run in parallel with other fixtures, lets add another fixture to the party:
```c#
[Parallelizable(ParallelScope.Children)]
internal class MyFixture1
{
	[Test]
	public async Task A()
	{
		TestContext.Progress.WriteLine("a start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("a end");
	}
	
	[Test]
	public async Task B()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("b start");
		TestContext.Progress.WriteLine("b end");
	}
}

[Parallelizable(ParallelScope.Children)]
internal class MyFixture2
{
	[Test]
	public async Task C()
	{
		TestContext.Progress.WriteLine("c start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("c end");
	}
	
	[Test]
	public async Task D()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("d start");
		TestContext.Progress.WriteLine("d end");
	}
}
```

Which prints out:
``` 
a start
b start
b end
a end
c start
d start
d end
c end
```

Test case C (from fixture 2) did not start until the 1st fixture ended.

What about the flip side - what if we want both fixtures to run in parallel, but the test cases in each fixture to run non-parallel (i.e.: sequentially)?

## ParallelScope.Fixtures
From [nUnit documentation](https://github.com/nunit/docs/wiki/Parallelizable-Attribute#user-content-parallelscope-enumeration):

> "fixtures may be run in parallel with one another"

If we run our previous C# example again (except with `ParallelScope.Fixtures` in place of `ParallelScope.Children`), we get:

``` 
c start
a start
c end
a end
d start
b start
d end
b end
```

With `A` & `C` starting at the same time, this proves that both fixtures start at the same time (i.e.: were running in parallel). Also, `B` only starts after `A` ends, & `D` only starts after `C` ends. Hence, the testcases inside each fixture ran sequentially.

But what if we want to go full throttle - running all fixtures in parallel *AND* the test cases in each fixture in parallel too. This means we want to fire all fixtures and all test cases side-by-side.

## ParallelScope.All
From nUnit documentation:

> "the test and its descendants may be run in parallel [with others at the same level](https://github.com/nunit/docs/wiki/Parallelizable-Attribute#user-content-parallelscope-enumeration)"

*same level* means that if you place the `Parallelizable` on a fixture, then that fixture will run parallel with other fixtures. And if you place it on a testcase, then that testcase will run parallel with the other testcases in that fixture.

Running our previous example again (except with `ParallelScope.All` in place of `ParallelScope.Fixtures`), we get:

```
[9:47:23.362 PM] c start
[9:47:23.362 PM] a start
[9:47:26.365 PM] b start
[9:47:26.368 PM] d start
[9:47:26.369 PM] b end
[9:47:26.371 PM] d end
[9:47:31.369 PM] c end
[9:47:31.372 PM] a end
```

`B` and `D` started 3 seconds after `A` and `C` started, and they started BEFORE `A` and `C` ended. This proves that everything was parallel; both the fixtures & their test cases.

And now, i must explain the final "parallel mode": `ParallelScope.Self`

## Let the confusion begin! ParallelScope.All vs ParallelScope.Self
So far we've gone through:

* Parallel fixtures (`ParallelScope.Fixtures`)
* Parallel testcases in a fixture (`ParallelScope.Children`)
* And the "all cylinders firing" option (`ParallelScope.All`)

What happens if we want more fine-grain control over whats parallel and whats not parallel? For example, 2 parallel fixtures, where each fixture has 1 parallel testcase & 1 non-parallel testcase?

Enter `ParallelScope.Self`. From the nUnit documentation:

> "the test itself may be run [in parallel with other tests](https://github.com/nunit/docs/wiki/Parallelizable-Attribute#user-content-parallelscope-enumeration)"

and

> "valid on: [classes, methods](https://github.com/nunit/docs/wiki/Parallelizable-Attribute#user-content-parallelscope-enumeration)"

This "parallel mode" is basically `ParallelScope.Fixtures` & `ParallelScope.Children` rolled into 1, and we can put it on fixtures and/or testcases. Here's the code i'm gonna run to test this "parallel mode" out:
```c#
[Parallelizable(ParallelScope.Self)]
internal class MyFixture1
{
	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task A()
	{
		TestContext.Progress.WriteLine("a start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("a end");
	}
	
	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task B()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("b start");
		TestContext.Progress.WriteLine("b end");
	}
}

[Parallelizable(ParallelScope.Self)]
internal class MyFixture2
{
	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task C()
	{
		TestContext.Progress.WriteLine("c start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("c end");
	}
	
	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task D()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("d start");
		TestContext.Progress.WriteLine("d end");
	}
}
```

Based on the above, we expect both fixtures to start at the same time. With both test fixtures having a parallel & non-parallel testcase, do the parallelizable testcases run first, or do the non-parallel ones run first? *drumroll*:
``` 
c start
a start
b start
d start
b end
d end
c end
a end
```

Ah just as expecte... wait a sec! The hell's goin on over here?

``` 
a start
b start
```

What? `B` started before `A` finished? But we marked `A` as parallel and `B` as non-parallel!

Its almost as if both test cases in each fixture ran in parallel! Wouldn't it have made more sense for the parallel ones to get first dibs, then all the non-parallel peasants line up after it?

![queue](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/queue.jpg "queue")

To get a clearer picture on what's going on here, let's add another non-parallel testcase to `MyFixture2`, then only run that fixture (instead of both fixtures):
```c#
[Parallelizable(ParallelScope.Self)]
internal class MyFixture2
{
	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task C()
	{
		TestContext.Progress.WriteLine("c start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("c end");
	}

	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task D()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("d start");
		TestContext.Progress.WriteLine("d end");
	}

	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task E()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("e start");
		TestContext.Progress.WriteLine("e end");
	}
}
```

And when running just this 1 fixture, the output is:
``` 
[10:19:17.405 PM] c start
[10:19:20.404 PM] d start
[10:19:20.406 PM] d end
[10:19:23.435 PM] e start
[10:19:23.447 PM] e end
[10:19:25.412 PM] c end
```

It seems that `E` (the 2nd non-parallel testcase we added), only started after the 1st non-parallel testcase (`D`) ended.

The point here is that when you mix parallel & non-parallel ingredients into a bowl, you essentially get the equivalent of 2 queues - 1 for parallel execution, and 1 for non-parallel execution.

Some analogies to help understand this are:
* Theres a line for VIP's, and another line for plebs/normies. Both lines run side-by-side (much like 2 bouncers each guarding 1 queue each outside of a club).
* Theres a fast road lane, and a slow road lane. Both lanes run side-by-side (much like a highway).

![fast lane slow lane](https://raw.githubusercontent.com/vitawebsitedesign/blog/master/assets/overtaking-lane.jpg "fast lane slow lane")

## Hitting the CPU core limit
Finally we can move on to exciting stuff - hitting the thread limit in nUnit!

I have a 4-core machine. From nUnit documentation:

> NUnit uses the processor count or 2, whichever is greater. For example, on a four processor machine [the default value is 4](https://github.com/nunit/docs/wiki/LevelOfParallelism-Attribute)

So technically speaking, nUnit will only be able to use up to 4 threads on my machine. Once it needs more, all bets are off the table. Any "extra" tests which are meant to run in parallel will instead be blocked until a thread becomes free.

To demonstrate this point, we need a bigger example. Hows about... 2 parallel fixtures, each containing 2 parallel testcases & 2 non-parallel testcases:
```c#
[Parallelizable(ParallelScope.Self)]
internal class MyFixture1
{
	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task A()
	{
		TestContext.Progress.WriteLine("Fixture1/TestCase1 start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("Fixture1/TestCase1 end");
	}

	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task B()
	{
		TestContext.Progress.WriteLine("Fixture1/TestCase2 start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("Fixture1/TestCase2 end");
	}

	// Here we omit the Parallelizable attribute, so by default, it will run in non-parallel mode
	[Test]
	public async Task C()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture1/TestCase3 start");
		TestContext.Progress.WriteLine("Fixture1/TestCase3 end");
	}

	// Here we omit the Parallelizable attribute, so by default, it will run in non-parallel mode
	[Test]
	public async Task D()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture1/TestCase4 start");
		TestContext.Progress.WriteLine("Fixture1/TestCase4 end");
	}
}

[Parallelizable(ParallelScope.Self)]
internal class MyFixture2
{
	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task E()
	{
		TestContext.Progress.WriteLine("Fixture2/TestCase1 start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("Fixture2/TestCase1 end");
	}

	[Parallelizable(ParallelScope.Self)]
	[Test]
	public async Task F()
	{
		TestContext.Progress.WriteLine("Fixture2/TestCase2 start");
		await Task.Delay(8000);
		TestContext.Progress.WriteLine("Fixture2/TestCase2 end");
	}

	// Here we omit the Parallelizable attribute, so by default, it will run in non-parallel mode
	[Test]
	public async Task G()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture3/TestCase3 start");
		TestContext.Progress.WriteLine("Fixture3/TestCase3 end");
	}

	// Here we omit the Parallelizable attribute, so by default, it will run in non-parallel mode
	[Test]
	public async Task H()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture2/TestCase4 start");
		TestContext.Progress.WriteLine("Fixture2/TestCase4 end");
	}
}
```

Awwww yeah, this is more like it. I feel the blood coarsing through my veins. The output on my 4-core machine is:
```
[8:40:44.906 PM] Fixture1/TestCase1 start
[8:40:44.907 PM] Fixture2/TestCase1 start
[8:40:46.635 PM] Fixture1/TestCase3 start
[8:40:46.640 PM] Fixture2/TestCase3 start
[8:40:46.642 PM] Fixture1/TestCase3 end
[8:40:46.657 PM] Fixture2/TestCase3 end
[8:40:49.698 PM] Fixture1/TestCase4 start
[8:40:49.700 PM] Fixture2/TestCase4 start
[8:40:49.702 PM] Fixture1/TestCase4 end
[8:40:49.704 PM] Fixture2/TestCase4 end
[8:40:49.714 PM] Fixture1/TestCase2 start
[8:40:49.716 PM] Fixture2/TestCase2 start
[8:40:51.629 PM] Fixture1/TestCase1 end
[8:40:51.633 PM] Fixture2/TestCase1 end
[8:40:57.715 PM] Fixture1/TestCase2 end
[8:40:57.725 PM] Fixture2/TestCase2 end
```

Pay attention to the first 4 lines (representing the 4 threads running on my 4-core machine):

* CPU Core 1: `Fixture1/TestCase1`
* CPU Core 2: `Fixture2/TestCase1`
* CPU Core 3: `Fixture1/TestCase3`
* CPU Core 4: `Fixture2/TestCase3`

The first 2 (`Fixture1/TestCase1` & `Fixture2/TestCase1`) are ones we marked as parallel, and the bottom 2 are the ones we marked non-parallel. Until one of these 4 ends, a thread won't be free to run another testcase, as proven by the lines:
```c#
...
[8:40:46.657 PM] Fixture2/TestCase3 end
[8:40:49.698 PM] Fixture1/TestCase4 start
...
```

In fact, `Fixture1/TestCase2` & `Fixture2/TestCase2` are waiting for a free thread too, even though we marked them for parallel execution. Ah... isn't parallelism quite an interesting specimen to observe?

## Closing notes
### STA/MTA?
The nUnit documentation sometimes uses the abbreviation *STA* or *MTA*. These refer to *Single Threaded Architecture* & *Multi Threaded Architecture*; the non-parallel & parallel nUnit architectures.

### Deterministic?
No dice - parallel tests are non-deterministic! Everytime you hit the run button, parallel tests may end in different orders. This means your tests need to be thread-safe to produce determinsitic pass/fail results.

### Conclusion
The nUnit team worked hard to give us parallelism. We should understand how it works so that we can leverage it to dramatically improve test execution scalability & developer productivity.

And lets face it - when devs start spinning in their chairs & twiddling their thumbs because the build takes so damn long, its a sign to roll up your sleeves & start improving the efficiency of internal processes, such as test suite execution speed.

Given the non-deterministic nature of tests, the nUnit team gave devs a business-viable migration option when it comes to improving their test execution speed; incremental migration. And this is the `Parallelizable` attribute that we know of today.
