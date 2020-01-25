---
title: "Know your tools: JavaScript Promises"
date: "2019-08-07T07:40:48.892Z"
description: I am sure you have used Promises before, but if you avoid them because of their mystical behavior, this article is for you.
---

I am sure you have used Promises before, but if you avoid them because of their mystical behavior, this article is for you.

> A Promise is an object that allows us to work on (possible) async task, and notify others when the task is done

Initially, **a Promise looks just like a glorified callback**. If you are familiar with NodeJS callbacks you've seen this pattern before

```javascript
// callback version
asyncTask((error, result) => {
    if (error) {
        onError(error);
        return;
    }
    onSuccess(result);
});
```

If `asyncTask` returns a Promise instead, this would be the equivalent version

```javascript
// Promise version
asyncTask()
.then(onSuccess)
.catch(onError);
```

The latest code present some advantages from the former:

1. It is shorter: 3 vs 7 lines of code
1. _onSuccess_ and _onError_ functions are going to be called at most once, and only one of them, **guaranteed**
1. Promises are _microtasks_, which means they run before functions from _macrotasks_ like `setTimeout` or I/O

But one of the biggest benefits is for the _Developer Experience_: running multiples async tasks without the callback hell.

## Avoiding callback hell
What is the callback hell? Let's see an example by calling in series 4 async functions using callbacks

```javascript
// callback version
getCredentials((error, credentials) => {
    if (error) return console.error('Credentials not found');

    login(credentials, (error, accessToken) => {
        if (error) return console.error('Credentials are invalid');

        getHomeInformation(accessToken, (error, homeInformation) => {
            if (error) return console.error('There was an error getting your info');

            recordAnalytics('home loaded', () => {
                // we can keep adding indentation, forever... 
                // you will probably need a bigger monitor 
                // to avoid all the horizontal scrolling 😂
            })
        })
    })
})
```

Slowly, we start to increase indentation by each callback call — and we are not even properly handle errors!

Let's take a look of how to handle this case with Promises

```javascript
// Promises version
getCredentials()
.then(login)
.then(getHomeInformation)
.then(() => recordAnalytics('home loaded'))
.catch((error) =>{
    /* Handles each error properly */
});
```

The advantages are clear: keeping indentation under control. No need for buying a bigger monitor 😉

Now, if you are used to use callbacks you can have the _Promise callback hell_ as well

```javascript
// Promises + callback hell version
// 🚫 DO NOT TRY THIS AT HOME! 🚫
getCredentials()
.then((credentials) => {
    login(credentials)
    .then((accessToken) => {
        getHomeInformation(accessToken)
        .then(() => {
            recordAnalytics('home loaded')
            .then(() => {
                // for new async tasks, 
                // they will keep increasing indentation
                // you will need to buy a new monitor, again 😂
            })
        })
        .catch((error) => console.error('There was an error getting your info'));
    })
    .catch((error) => console.error('Credentials are invalid'));
})
.catch((error) => console.error('Credentials not found'))
```

Just by using Promises you don't avoid the callback hell: it is in your hands to use Promises as designed. Let's follow 3 simple rules that will help us improve our relationship with Promises.

## Rule #1: `onSuccess` and `onError` should always return (or throw)
Promise objects have two methods: `.then(onSuccess)` and `.catch(onError)`, and their received callback will be called when the Promise state changes from its initial value, _pending_, to a _fulfilled_ or _rejected_ state respectively. 

For both `onSuccess` and `onError` callbacks it is a good practice to always `return` (or to `throw`), to pass a value to the next chained `.then/.catch`. But what should we return?

- a sync value (a number, an object, a string...), or
- another Promise object 

In case of a Promise, its result **will be the value available for the next chained `.then/.catch`**. Let's look at some examples

```javascript
// always return/throw
fetch('https://api.domain.com/')
.then((response) => {
    return response.json();  // pass its result to the next `.then`
});
.then((json) => {
    if ('id' in json) {
        return json;         // pass the object to the next `.then`
    }

    // skip all following `.then` and pass error to next `.catch`
    throw new Error('id is not present');
})
.then((validatedJson) => {
    // we have an object with an `id`
});
```

And what happen if we don't return nor throw? Promises will not be chained at things will run in an order not guaranteed. Look at these anti-patterns

```javascript
// random things you can do inside a `.then`
// 🚫 DO NOT TRY THIS AT HOME! 🚫
fetch('https://api.domain.com/')
.then((response) {
    response.json();    // 🐛 function doesn't return, order isn't guaranteed!
}); 
.then((json) => {
    console.log(json);  // 🐛 received undefined from last `.then`, not what we expected!
})
.then();                // 🐛 no function at all, resolved value will pass to next `then`
.then(aPromiseObject);  // 🐛 expected a function, received an object!
```
    
## Rule #2: Prepare for errors, always `catch`

This rule is simple. Eventually, things will go wrong and you have to prepare for that case.

Always place a `.catch` at the end of your Promise chains and make sure you handle unexpected errors. If you don't, modern browsers will print an error in the console: that is really useful for you during development but gives users no clue if they press a button and nothing happens. 

Continuing with the previous example,

```javascript
// always catch
fetch('https://api.domain.com/')
.then((response) => response.json());
.then((json) => {
    if ('id' in json) {
        return json;
    }
    throw new Error('id is not present');
})
.catch((error) => {
    if (error.message === 'id is not present') {
        // handle the `id` was not present error
    } else {
        // other errors, like fetch failing
    }
})
```

Remember: think in your users and always add _at least one_ `.catch`.

## Rule #3: Create Promise with the right approach

Until this moment, Promise objects have appeared from existing APIs. But how can we create our own Promises?

### Create a Promise from a sync value

This is easy: all you have to do is call `Promise.resolve(value)` or `Promise.reject(message)` if you want a fulfilled or rejected Promise.

### Create a Promise from a callback

If you want to convert a callback-based function to return a Promise you will have to call the Promise constructor. Let's look an example:

```javascript
// from NodeJS callback to Promise
const asyncTaskPromise = () => {
    return new Promise((resolve, reject) => {
        asyncTask((error, result) =>
            (error) ? reject(error) : resolve(result)
        );
    });
};

asyncTaskPromise()
.then(...)
```

Now that you know the basic rules, let's take a look to a bit more advanced patterns you will be writing.

## Advanced Promise patterns

You have to be mindful and create Promise _only when you need them_. Consider this: you have a dynamic list of Promises you want to run in parallel, and other list to be run in series each after other. How would you approach each case?

### Running in parallel

Use `Promise.all(list)` to wait for a list of promises to be ready. If every Promise succeeded you will receive a list with their corresponding values in the next `.then`; if any of them fails you will get the error in the next `.catch`.

```javascript
// running in parallel ✈️️️️️️️️️️ ✈️️️️️️️️️️️️ ✈️️️️️️️️️️️️ ✈️️️️️️️️️️️️
const urls = ['/1', '/2', '/3', '/4'];
// fetch are triggered "in parallel"
const promises = urls.map(url => fetch(url));

Promise.all(promises)
.then((responses) => responses.map(res => res.json())
.then((jsons) => {
    // we have an array with the json for each call!
})
.catch((error) => {
    // the first rejected error will get here
})
```

### Running in series

What if we need to wait for the result of a previous Promise before calling the next one?

Running a `.map` will not work as the Promise is created in that moment. You need to create a function that will return a new Promise when needed (also called a _factory_).

Start the Promise series with an initial `Promise.resolve()`, just to have something where to chain to (call `.then`).

```javascript
// run in series 🚂🚃🚃🚃
const series = Promise.resolve();
urls.forEach(url => {
    series.then((previousText) => fetchFactory(url, previousText));
})

series
.then((finalText) => {
    // we have the last Promise result
})
.catch((error) => {
    // the first rejected error will get here
})

function fetchFactory(url, text) {
    if (text) {
        url = url + '?text=' + text;
    }

    return fetch(url)
    .then(response => response.text());
}
```

### Cancelling a Promise

Let's say you want to poll an API, and you want to have a way to cancel it. There is no `.cancel` method in the Promise class, but we can achieve something similar by creating our own `cancel` function by racing between Promises.

Here I used `Promise.race(list)`: it receives a list of Promises and resolves or rejects as soon as one of the Promise from its list resolves or rejects, with the value from that Promise.

In the next example, a `wait` Promise will race against the user cancelling the Promise. If the user never cancels, the system will keep polling until the response has a `status === 'ready'` and will fulfilled with the json response. If the user cancel, the Promise will reject with a message `'cancelled by user'` instead.

```javascript
function wait(delay) {
    return new Promise((resolve) => setTimeout(resolve, delay));
}

function cancellablePolling(url, ms) {
    let rejectCallback;
    const cancelPromise = new Promise((resolve, reject) => {
        rejectCallback = reject;
    });

    const poll = () => Promise.race([wait(ms), cancelPromise])
        .then(() => fetch(url))
        .then((response) => response.json())
        .then((json) => json.status === 'ready' ? json : poll());
    
    const cancel = () => rejectCallback('cancelled by user');
    const promise = poll();

    return {
        cancel,
        promise,
    };
}

const { promise, cancel } = cancellablePolling('/status', 5000);
promise.then(...);      // is a regular Promise
cancel();               // that will be rejected if `cancel` is called before 
```

## What's next?

Once you feel comfortable with Promises, a nice addition to the JavaScript language is the `await` operator and the `async` functions. This allows to call async functions in a more standard syntax, like sync functions with the regular `try/catch` statements.

By using `async function` you declare a function to be async, and always returning a Promise. The advantage is now you can "halt" the execution of your function and `wait` for Promises to fulfill. Let's take a look to the first example of this article and see how it is different:

```javascript
// callback version
const main = () => {
    asyncTask((error, result) => {
        if (error) {
            onError(error);
            return;
        }
        onSuccess(result);
    });
};
```

and now, using `async function + await`, assuming `asyncTaskPromise` is the Promise version of `asyncTask`:

```javascript
// Promise version, async function/await
const main = async () => {
    try {
        const result = await asyncTaskPromise();
        onSuccess(result);
    }
    catch (error) {
        onError(error);
    }
};
```

Currently, `Promises` are [widely supported](https://caniuse.com/#feat=promises) across browsers [as well as](https://caniuse.com/#feat=async-functions) `async functions`, both over 90%. The recommendation is to understand who are your users and polyfill and/or transpile if it makes sense.

Hope you have learn something today. Do you have your own rules or advances uses cases for Promises? Comment them below. Have a nice week!
