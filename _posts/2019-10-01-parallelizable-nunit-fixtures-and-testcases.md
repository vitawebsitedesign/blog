---
title: Exploring parallel behaviour in nUnit fixtures & testcases
layout: post
author: Michael Nguyen
---
Every build server has a limit, and once you hit it you'll find your nUnit tests slowing down the build process.

Thankfully, we can run nUnit tests in parallel by using the `Parallelizable` attribute. This tells the runner to run "stuff" in parallel mode.

I use the word "stuff", because theres [4 different parallel modes](https://github.com/nunit/docs/wiki/Parallelizable-Attribute) to choose from. The documentation explains it kind've ambiguously, so here's my take on it:
* `ParallelScope.All` 
All
Children
Fixtures
Self

Since this can get confusing pretty fast, i wanna explain this through examples. These are listed in a different order than the documentation, in order to aid understandability.

Before starting this party, the below examples use the Visual Studio nUnit runner (rather than the CLI runner). This is an important fact because:

> "Parallel execution is [the default behavior](https://github.com/nunit/docs/wiki/Engine-Parallel-Test-Execution) when running multiple assemblies together using the nunit3-console runner"

And this means that for the below examples:

> "By default, [no parallel execution takes place](https://github.com/nunit/docs/wiki/Framework-Parallel-Test-Execution)"

## ParallelScope.Children
Firstly, think of parallelable tests as 2 layers - test cases (the functions), and fixtures (the classes that hold the test cases). The nUnit docs on this attribute state:

> "child tests may be run in parallel with one another"

In other words: test cases in a fixture will run in parallel, but the fixture itself will NOT run in parallel with other fixtures`

For example, a test case would be a function:
```c#
	[Test]
	public async Task MyTestCase()
	{
		...
	}
```

And a test fixture would be the class that holds the function:
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

So by adding `ParallelScope.Children` attribute to the class, test cases inside the fixture would now run in parallel:
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

And this prints out:
```bash
a start
b start
b end
a end
```

Proving that they run in parallel. And to prove that the fixture does NOT run in parallel with other fixtures, lets add another fixture to the party:
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
```bash
a start
b start
b end
a end
c start
d start
d end
c end
```

Test case C did not start until the 1st fixture ended.

What about the flip side - what if we want both fixtures to run in parallel, but the test cases in each fixture to run non-parallel (i.e.: sequentially)?

## ParallelScope.Fixtures
> "fixtures may be run in parallel with one another"

Running our previous example again (except with `ParallelScope.Fixtures` above both fixtures, instead of `[Parallelizable(ParallelScope.Children)]`), we get:

```bash
c start
a start
c end
a end
d start
b start
d end
b end
```

This proves that the fixtures start at the same time (i.e.: were running in parallel), but the testcases inside each fixture run sequentially.

But what if we want to go full throttle - running all fixtures in parallel, AND the test cases in each fixture in parallel too. This means we want to fire all fixtures and all test cases immediately.

## ParallelScope.All
> "the test and its descendants may be run in parallel with others at the same level"

Running our previous example again (except with `ParallelScope.All` above both fixtures, instead of `[Parallelizable(ParallelScope.Fixtures)]`), we get:

```bash
a start
c start
d start
b start
d end
b end
a end
c end
```

This proves that everything was parallel; the fixtures & their test cases.

So thats all the cases right?

## Let the confusion begin! ParallelScope.All vs ParallelScope.Self
Ok so we've gone through parallel fixtures, parallel testcases, and the "all cylinders firing" option called `ParallelScope.All`.

What happens if we want finer control over whats parallel? Perhaps i want 2 fixtures to be parallel, with 1 of the test cases in parallel, and the other in non-parallel mode?

`ParallelScope.Self` is the fourth option, and is all about allows us to get what we want.

> "the test itself may be run in parallel with other tests"

> "valid on: classes, methods"

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

Here we expect both fixtures to start at the same time. But... do the parallelizable testcases run first, or do the non-parallel ones run first? *drumroll*:
```bash
c start
a start
b start
d start
b end
d end
c end
a end
```

Ah just as expecte... wait a sec!

```bash
a start
b start
```

What? B started before A finished? Its almost as if both test cases in each fixture ran in parallel! Wouldn't it have made sense for the parallel ones to get first dibs, then all the non-parallel peasants line up to run sequentially?

To get a clearer picture, let's add another non-parallel testcase to `MyFixture2`:
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
```bash
c start
d start
d end
e start
e end
c end
```

It seems that even though C & D start at the same time, the 2nd non-parallel testcase `E` only started after `D` ended.

When you mix parallel & non-parallel attributes into a bowl, you essentially get the equivalent of 2 queues - 1 for parallel fixtures/testcases, and 1 for non-parallel fixtures/testcases.

Other ways of thinking about this:
* Theres a line for the VIP's, and another line for the plebs. Both lines run side-by-side (much like 2 bouncers outside a club).
* Theres a fast road lane, and a slow road lane. Both lanes run side-by-side (much like a highway).

## Hitting the CPU core limit!
On my 4-core machine, nUnit will only be able to use 4 threads.

If exceed 4 threads, a test that is meant to be parallel is instead blocked until a thread gets freed.

To demonstrate this point, we need a more full-scale example: 2 parallel fixtures, each containing 2 parallel testscases & 2 non-parallel testcases
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

	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task C()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture1/TestCase3 start");
		TestContext.Progress.WriteLine("Fixture1/TestCase3 end");
	}

	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
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

	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task G()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture3/TestCase3 start");
		TestContext.Progress.WriteLine("Fixture3/TestCase3 end");
	}

	// Here we omit the Parallelizable attribute, so by default. it will run in non-parallel mode
	[Test]
	public async Task H()
	{
		await Task.Delay(3000);
		TestContext.Progress.WriteLine("Fixture2/TestCase4 start");
		TestContext.Progress.WriteLine("Fixture2/TestCase4 end");
	}
}
```

And the output (on my 4-core machine) is:
```bash
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
Core 1: Fixture1/TestCase1
Core 2: Fixture2/TestCase1
Core 3: Fixture1/TestCase3
Core 4: Fixture2/TestCase3

These first 2 are ones we marked as parallel, and the other 2 are the ones we marked non-parallel. Until one of them ends, a thread won't be free to run another testcase:
```c#
...
[8:40:46.657 PM] Fixture2/TestCase3 end
[8:40:49.698 PM] Fixture1/TestCase4 start
...
```

In fact, `Fixture1/TestCase2` & `Fixture2/TestCase2` are waiting for a free thread too, even though we marked them for parallel execution. Ah.. such is life.

## STA/MTA?
The nUnit documentation sometimes uses the abbreviation "STA" or "MTA". These just refer to "Single Threaded Architecture" & "Multi Threaded Architecture", essentially the non-parallel & parallel nUnit architectures.

## Deterministic?
No dice - parallel tests are non-deterministic. Everytime you hit the run button, parallel tests may end in different orders. This means your tests need to be thread-safe to produce determinsitic pass/fail results.

## Final thoughts
The nUnit team worked hard to give us parallelism. We should understand how it works so that we can leverage it to dramatically improve test execution scalability & developer productivity.

And lets face it, when devs start twiddling their thumbs because the build takes so damn long, its a sign to roll up your sleeves & start improving the efficiency of internal processes, such as the test suite execution speed.

Given the non-deterministic nature of tests, the nUnit team gave devs a viable option of improving test execution speed - incremental migration. And this is the `Parallelizable` attribute that we know of today.
