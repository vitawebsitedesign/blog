---
title: Handling exceptions in non-async methods that return Task
layout: post
author: Michael Nguyen
toc: true
---

When a task isn't awaited, it becomes a *"fire & forget"* task. This means any exceptions thrown will be raised on the Task instead.

This introduces an interesting scenario where if a method directly returns a Task without `await`, it wont catch any raised exceptions. Instead, the method that **awaited** the Task catches them *instead*.

Consider:

```c#
public async Task SomeTask() {
    // ...
    throw new Exception();
    /// ...
}

public Task GetSomeTask() {
    try
    {
        return SomeTask();
    }
    catch
    {
        // ...
    }
}

public async void Main() {
    await GetSomeTask();
}
```

When `Main()` is called:
1. `SomeTask()` throws an exception
1. Since `GetSomeTask()` doesn't await the call, its simply fired & forgot
1. The exception gets raised on the `Task` instead of in `GetSomeTask()`
1. This means the exception handling needs to be in `Main()` instead of `GetSomeTask()`

Its quite an easy aspect to *forget* when visualising deep conditional call stacks. And currently, our little code sample is actually causing an unhandled exception on launch.

To fix this, we just need to **expect** exceptions in `Main()`:

```c#
public Task GetSomeTask() {
    try
    {
        return SomeTask();
    }
    catch
    {
        // ...
    }
}

public async void Main() {
    await GetSomeTask();
}
```

to:

```c#
public Task GetSomeTask() {
    return SomeTask();
}

public async void Main() {
    try
    {
        await GetSomeTask();
    }
    catch
    {
        // ...
    }
}
```

And now our unhandled exception is handled.

Its such an important & serious bug because:
- It can be a **high occurrence** scenario in a codebase. By default, its often better to not have nested `await` everywhere & return `Task` directly (where possible).
- Its a **low detectability** issue thats only detectable at run-time. And
- Since exceptions can unexpectingly go **unhandled**, it can cause serious issues depending on the context of the issue causing the exception.
