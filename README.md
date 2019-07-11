---
layout: default
---
## Welcome

Task.WhenAll() runs asynchronous tasks in parallel.

Which code sample below is faster?

https://github.com/vitawebsitedesign/async-parallel-benchmark

Results:

![alt text](https://github.com/adam-p/markdown-here/raw/master/src/common/images/icon48.png "Logo Title Text 1")

Some articles claim that code sample #1 is not faster, since returned Task are hot (i.e.: Task-based Async Pattern).

However this is incorrect, because the 2nd task isn't "returned" until the 1st await finishes.

If you're using the Task.WhenAll(IEnumerable<Task<TResult>>) overload, just ensure you dont blow up your computer & that the code remains scalable (e.g.: if number of tasks in list grows, or if task is updated in future to do more IO/CPU)
