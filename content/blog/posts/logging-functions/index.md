---
title: "Logging functions in JavaScript"
date: "2020-05-18T01:26:26.780Z"
description: "Keep track of your functions and methods' inputs and outputs, cleanly."
---

Logs can be very useful to debug issues in production environments.

By logging when functions are called, with which parameters, it is possible to reproduce issues and find root causes.

I will explore the logging of functions and methods within classes.

## Functions
Let's say we have a function we'd like to know when it is being called, its parameters, and its result.

In this example, the function is called `range()` that when given two numbers it returns a list with the digits between them, excluding the last:

```javascript
function range(start, end) {
    return Array(end - start)
        .fill(start)
        .map((start, index) => start + index);
}

range(0, 5);    // returns [0, 1, 2, 3, 4]
```

An initial approach would be to edit the original function and add the logs needed:

```javascript
function range(start, end) {
    // highlight-next-line
    console.log(`Calling range(${start}, ${end})`);

    const result = Array(end - start)
        .fill(start)
        .map((start, index) => start + index);

    // highlight-next-line
    console.log(`range(${start}, ${end}) returns ${result}`);

    return result;
}

range(0, 5);    // returns [0, 1, 2, 3, 4]
// logs these two lines
//    Calling range(0, 5)
//    range(0, 5) returns 0,1,2,3,4
```

This works, however it makes the function harder to read by mixing the proper logic of the function and the boilerplate to make the logs. This may be okay for a 3 line function, however longer ones will become even longer and harder, making them harder to read and to make changes.

An alternative solution would be to move all the logging logic outside to its function:

```javascript
// let's keep the original version of `range()` without logging statements

// `logFunction()` creates a new function that handles the logging for us
// highlight-next-line
function logFunction(fn, thisArg) {
    return function() {
        const concatenatedArguments = Array.from(arguments).join(', ');
        console.log(`Calling ${fn.name}(${concatenatedArguments})`);

        const result = fn.apply(thisArg, arguments);

        console.log(`${fn.name}(${concatenatedArguments}) returns ${result}`);
        return result;
    }
}

// create a version of `range()` with logs
// highlight-next-line
const loggedRange = logFunction(range);

loggedRange(0, 5);    // returns [0, 1, 2, 3, 4]
// logs these two lines
//    Calling range(0, 5)
//    range(0, 5) returns 0,1,2,3,4
```

There are advantages to this approach:
1. now the business logic doesn't need to be mixed with logging, keeping our functions shorter, 
1. you can reuse this function into other functions that need logging as well
1. if you don't need the logs anymore, stop calling `logFunction`

Sweet. What about methods?

## Methods within classes

Let's take the example of a `LoginEmailService` class, that needs logging for its 2 methods:

```javascript
class LoginEmailService {
    lostPasswordFor(emailAddress, usersDB) { ... }
    verifyEmailAddressFor(user, emailAddress) { ... }
}
```

Following the idea of keeping logic apart from the logs, we can **create a new class** that extends the original one and makes the logging for us. A first attempt would be:

```javascript
class LoggedLoginEmailService extends LoginEmailService {
    // option 1: placing the logs directly
    lostPasswordFor(emailAddress, usersDB) {
        // highlight-next-line
        console.log(`Calling lostPasswordFor(${emailAddress}, ${usersDB})`);
        const result = super.lostPasswordFor(emailAddress, usersDB);
        // highlight-next-line
        console.log(`lostPasswordFor(${emailAddress}, ${usersDB}) returns ${result}`);
        return result;
    }
    // option 2: using `logFunction()`
    verifyEmailAddressFor() {
        // highlight-next-line
        const loggedVerifyEmailAddressFor = logFunction(super.verifyEmailAddressFor);
        return loggedVerifyEmailAddressFor.apply(this, arguments);
    }
}
```

In the example, it works fine to place the logs directly or reusing the `logFunction()` utility. However, neither of those options will scale well when adding more methods to the original class as it will require manual updates within `LoggedLoginEmailService` class.

Wouldn't it be nice if logs are added to all methods of a class, including future modifications? This is possible using the `Proxy` class, used to define custom behavior for operations like reading a property and function invocation. If you want to know more about Proxy [read this excellent article](https://javascript.info/proxy). Go on and read it, I'll wait here.

For the next example we will write 2 functions using Proxies: 
1. an external one, `loggingMethodsFor()`, will listen for reading the properties of an object, and will invoke...
1. the internal `logMethod()` to log when a method is executed

```javascript
// highlight-next-line
function loggingMethodsFor(instance) {
    return new Proxy(instance, {
        get(target, propertyName) {
            const property = target[propertyName];

            if (isMethod(property)) {
                return logMethod(property, target.constructor.name);
            }

            return property;
        }
    })
}

function isMethod(property) {
    return typeof property === "function";
}

// equivalent of `logFunction()` but for methods
// highlight-next-line
function logMethod(method, className) {
    return new Proxy(method, {
        apply(fn, thisArg, argumentsList) {
            const concatenatedArguments = Array.from(argumentsList).join(', ');
            console.log(`Calling ${className}.${fn.name}(${concatenatedArguments})`);

            const result = fn.apply(thisArg, argumentsList);

            console.log(`${className}.${fn.name}(${concatenatedArguments}) returns ${result}`);
            return result;
        }
    })
}

// highlight-next-line
const loggedLoginEmailService = loggingMethodsFor(new LoginEmailService());
loggedLoginEmailService.lostPasswordFor('a@b.c', 'db');
// log these two lines:
//    Calling LoginEmailService.lostPasswordFor(a@b.c,dbObject)
//    LoginEmailService.lostPasswordFor(a@b.c, db) returns <...>
```

Do you realize how similar both `logFunction()` and `logMethod()` are? By using the fact that there are no technical differences between methods and functions, we can modify `logFunction()` for logging _both methods and functions_.

```javascript
function logFunction(methodOrFn, className) {
    const isMethod = className !== undefined;

    return new Proxy(methodOrFn, {
        apply(fn, thisArg, argumentsList) {
            const concatenatedArguments = Array.from(argumentsList).join(', ');
            const fnPrefix = isMethod ? className + '.' : '';
            console.log(`Calling ${fnPrefix}${fn.name}(${concatenatedArguments})`);

            const result = fn.apply(thisArg, argumentsList);

            console.log(`${fnPrefix}${fn.name}(${concatenatedArguments}) returns ${result}`);
            return result;
        }
    })
}
```

## What about Promises?
You may be wondering what if the function or method returns a Promise. Currently, this function is printing `"[object Promise]"` which is not helpful at all. Let's modify the latest lines of `logFunction()` to change the way the output is logged.

```javascript
// inside `logFunction()`
// …
const result = fn.apply(thisArg, argumentsList);

const returnsPromise = result && result.then && typeof result.then === "function";
if (returnsPromise) {
    result.then(output => console.log(`${fnToString} returns ${output}`));
} else {
    console.log(`${fnPrefix}${fn.name}(${concatenatedArguments}) returns ${result}`);
}

return result;
// …
```

## Conclusion
We explored how functions and methods can log their arguments and output without mixing log statements with the business logic.

We reach to a single function that can be used for both functions and methods, handling Promises and non-Promises outputs.

Even if our `logFunction()` and `loggingMethodsFor()` functions work they may not be ready for production. For example, depending on how complex are the arguments received by your functions you may want to deep dive into how the `concatenatedArguments` variable is being built, by leveraging `JSON.stringify` or any other technique to convert variables to a string. You may also need to avoid logging every method within an object, by having a blocked or allowed list of methods.

I will write here the latest version of the two functions in case you want to play with them. Also split some logic into smaller functions to make it easier to read.

Do you have other ideas for logging?

```javascript
function loggingMethodsFor(instance) {
    return new Proxy(instance, {
        get(target, propertyName) {
            const property = target[propertyName];

            if (typeof property === "function") {
                return logFunction(property, target.constructor.name);
            }

            return property;
        }
    })
}

function logFunction(methodOrFn, className) {
    return new Proxy(methodOrFn, {
        apply(fn, thisArg, argumentsList) {
            const fnToString = functionAndArgumentsToString(argumentsList, className, fn.name);
            logFunctionStarts(fnToString);
            const result = fn.apply(thisArg, argumentsList);
            logFunctionEnds(fnToString, result);
            return result;
        }
    })
}

function functionAndArgumentsToString(argumentsList, className, functionName) {
    const concatenatedArguments = Array.from(argumentsList).join(', ');
    const functionPrefix = className ? className + '.' : '';
    return `${functionPrefix}${functionName}(${concatenatedArguments})`;
}

function logFunctionStarts(fnToString) {
    console.log(`Calling ${fnToString}`);
}

function logFunctionEnds(fnToString, result) {
    const isPromise = result && result.then && typeof result.then === "function";
    if (isPromise) {
        result.then(output => console.log(`${fnToString} returns ${output}`));
    } else {
        console.log(`${fnToString} returns ${result}`);
    }
}
```