---
title: When and how Task.WhenAll improves performance
layout: post
author: Michael Nguyen
toc: true
---

Task.WhenAll() runs asynchronous tasks in parallel.

"Parallel" = multiple tasks running at the same time, "Asynchronous" = tasks dont block main thread.

Which code sample below is faster?

```c#
private static async Task TestAsyncSequential()
{
    await EmptyTask();
    await EmptyTask();
    await EmptyTask();
}
```

```c#
private static async Task TestAsyncParallel()
{
    await Task.WhenAll(EmptyTask(), EmptyTask(), EmptyTask());
}
```
[https://github.com/vitawebsitedesign/async-parallel-benchmark](https://github.com/vitawebsitedesign/async-parallel-benchmark)

## 10-run benchmark
### Non-parallel awaits
<blockquote class="imgur-embed-pub" lang="en" data-id="a/DkNrZZ3"  ><a href="//imgur.com/a/DkNrZZ3"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>
### Parallel awaits via Task.WhenAll()
<blockquote class="imgur-embed-pub" lang="en" data-id="a/IvsrYQD" data-context="false" ><a href="//imgur.com/a/IvsrYQD"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Some articles claim that code sample #1 is not faster, since returned Task are hot (i.e.: [Task-based Async Pattern](https://docs.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)

However this is incorrect. Even though hot tasks "start" as soon as theyre returned, the 2nd task isn't "returned" until the 1st await finishes.

Note: tasks created <a href="https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.-ctor?view=netframework-4.8">via constructor (i.e.: new Task(...))</a> start cold, not hot

If you're using the <a href="https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.whenall?view=netframework-4.8">Task.WhenAll(IEnumerable&lt;Task&lt;TResult&gt;&gt;) overload</a>, just ensure you dont blow up your computer & that the code remains scalable (e.g.: if number of tasks in list grows, or if task is updated in future to do more IO/CPU)

## Is it this simple?
Consider the below code:

```c#
var t1 = EmptyTask();
var t2 = EmptyTask();
var t3 = EmptyTask();
var result1 = await t1;
var result2 = await t2;
var result3 = await t3;
```

Instead of awaiting each returned task immediately, all 3 tasks are returned before an await.

If the tasks are fast, all 3 hot tasks may already be complete by the 1st await (await t1).

In this case, if we're ok with task #2 & #3 running before the 1st await, Task.WhenAll() needn't be used.
