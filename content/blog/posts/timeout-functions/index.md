---
title: "Time-out functions in JavaScript"
date: "2020-06-14T05:33:42.226Z"
description: "Is it possible to stop a function if it has been running for a long time?"
---

I've been thinking about the problem of _time-out a function_, this is, try to run a function until a maximum amount of time, so it can either complete in time or be interrupted.

So I rolled up my sleeves and wrote this

```javascript
function runWithTimeout(fn, timeoutInMs) {
    let timeoutId;
    const timeout = new Promise((resolve, reject) => {
        timeoutId = setTimeout(() => reject('timeout'), timeoutInMs);
    });

    const fnResult = Promise.resolve(fn());

    return Promise.race([fnResult, timeout])
        .finally(() => clearTimeout(timeoutId));
}
```

It is meant to be used by passing a function with no parameters, plus how many milliseconds should be allowed to run. And by wrapping it in a Promise it can either support synchronous and asynchronous values.

However, `runWithTimeout()` doesn't work as expected.

The rest of the article I'll explain why this function doesn't work and why it is impossible to solve this problem in JavaScript.

Consider these examples,

```javascript
function uninterruptibleFaultySync() {
    while (true) {  }
}

function uninterruptibleFaultyAsync() {
    if (true) { queueMicrotask(uninterruptibleFaultyAsync); }
}
```

both will never be interrupted by the `runWithTimeout()` function. How so?


## JavaScript is a single-threaded language
If JavaScript is a single-threaded language, this means a single function is being executed at any given point in time. But then how do we have asynchronous functions? The answer is in the **event loop**.

The event loop is a list of functions that the JavaScript engine will be running, in order. If there are no functions in the event loop, the engine will be doing nothing until a new function is placed on the list.

The functions in the event loop are called **tasks** or **macrotasks**. How are tasks created? Functions run when a file is loaded, and functions placed with `setTimeout()`, `setInterval()` are the most common examples.

The interesting thing about macrotasks is, once a task starts running it will run until completion with no interruptions. That's the reason why `uninterruptibleFaultySync()`, which never ends, will run forever. But for other functions that do complete, they allow the event loop to execute the next function in its queue.

This is not the whole story. There are other types of functions that are considered by the event loop which have more priority than the macrotasks: they are called **microtasks**. Microtasks are run when the current macrotask completes, and the next macrotask will not be taken until the list of microtasks is empty. How are microtasks created? With Promises and calls to `queueMicrotask()`.

And because microtasks functions will be run until completion with no interruptions, calling `uninterruptibleFaultyAsync()` will keep putting elements in the microtask list, never allowing pending macrotasks to be started. But for other microtasks that do complete, without creating infinite microtasks, they allow the event loop to keep running the macrotasks in its queue.


## Is there a way to timeout a function?
I am afraid it is not possible, at least for the general case. You can see this [related question in Stack Overflow](https://stackoverflow.com/questions/8778718/how-to-implement-a-function-timeout-in-javascript-not-just-the-settimeout) to read other people answers.

Now, if the function you want to timeout is not a buggy one, like the uninterruptible ones presented here, `runWithTimeout()` will indeed help you trigger a timeout error if necessary.

How can you avoid writing uninterruptible functions? Here I bring some ideas:
- always return a Promise
- ensure your Promises are catching errors
- ensure, for loops and recursion, that the function will stop