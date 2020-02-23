---
title: "Retrying functions"
date: "2020-02-20T07:12:28.079Z"
description: "The problem statement is: you want to execute a function that may fail, and you want to retry it N times. How could you solve this?"
---

The problem statement is: you want to execute a function that may fail, and you want to retry it N times. How could you solve this?

An initial approach can be to program it for a specific number of times. Let's run `functionThatMayFail` and retry it 3 times

```javascript
try {
    functionThatMayFail();
} catch (e0) {
    try {
        // 1st retry
        functionThatMayFail();
    } catch (e1) {
        try {
            // 2nd retry
            functionThatMayFail();
        } catch (e2) {
            try {
                // 3rd retry
                functionThatMayFail();
            } catch (e3) {
                // give up
                handleException(e3);
            }
        }
    }
}
```

The previous snippet has some problems:
- it involves duplication, and
- the number of retries can't be modified at runtime

By using recursion we could write a second approach that may address those disadvantages —however I would advocate to avoid it if possible, as this is not a recursive problem _per se_ and the code will be harder to read and debug.

A third approach I'm more willing to follow is to make a utility function that takes care of both _calling a function and its retries_,

```javascript
function callAndRetry(fn, maxAmountOfRetries) {
    let error;
    for (let retries = 0; retries < maxAmountOfRetries; retries++) {
        try {
            return fn();
        } catch (err) {
            error = err;
        }
    }
    throw error;
}
```

In our example, this should be used as

```javascript
try {
    callAndRetry(functionThatMayFail, 3);
} catch (error) {
    handleException(error);
}
```

The presented approach solves the original statement and doesn't have the disadvantages of the previous methods. Another good thing about this pattern is that can be repurposed for asynchronous functions as well. How so?

Let's say `functionThatMayFail` now returns a Promise. Instead of hard-coding retries or going recursive, we can _slightly_ modify `callAndRetry` to wait for the function to fulfill, as

```javascript
// highlight-next-line
async function callAndRetry(fn, maxAmountOfRetries) {
    let error;
    for (let retries = 0; retries < maxAmountOfRetries; retries++) {
        try {
            // highlight-next-line
            return await fn();
        } catch (err) {
            error = err;
        }
    }
    throw error;
}
```

Did you notice how we now use an [async function](https://wiki.developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) and `await` for the `fn` function to fulfill? All the rest remains unchanged.

This new version can be used with the typical Promise chaining,

```javascript
callAndRetry(functionThatMayFail, 3)
.catch(handleException);
```

But I believe there is even more value when it is used like a sync call:

```javascript
// in the context of an async function, so we can `await`
(async () => {

    try {
        await callAndRetry(functionThatMayFail, 3);
    } catch (error) {
        handleException(error);
    }

})();
```

And the reasons are:
- we use regular `try/catch` for error exception, and
- <strong>this code can be used for both synchronous and asynchronous functions!</strong> Yes, `functionThatMayFail` may return a Promise or anything else and this would still works.

Another advantage of using a snippet to encapsulate the retry logic, instead of mixing it with the business logic, is that we can write it once and reused in your whole project as needed. You may even install one its versions available at [NPM](https://www.npmjs.com/search?q=retry) which include different retry strategies, if you need them.

I read your feedback in the comments below. Happy retrying!
