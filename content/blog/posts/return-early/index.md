---
title: "Small Things Matter: Returning early"
date: "2020-01-16T05:11:32.897Z"
description: Small improvements add up. Sadly, the same thing happen with less fortunate decisions. Keep your code with less indentation with this trick.
---

At the begging of my career as software developer I remember writing really long functions, full of nested conditionals.

I asked a teacher what could you do about this, and this was the first time I was introduced to the "return early" pattern, which improved the readability of my code by a great extent.

## The rule
Every time when you are faced with a conditional in a function:

```javascript
function() {
    if (condition) {
        doA();
    } else {
        doB();
    }
}
```

it is possible to rewrite it in a way you **return** after the condition, making the else block redundant.

```javascript
function() {
    if (condition) {
        doA();
        // highlight-next-line
        return;
    }
    // highlight-next-line
    // look mam, no else!
    doB();
}
```

This effectively allows you to:
1. write code with less indentation (i.e. less horizontal scrolling)
2. reduce cognitive load, as when you return you don't need to keep in mind what happen in the other branch of the condition

## The example
So this seems like a small change that may not have a lot of sense. However, when small improvements are put in place together, the change is noticeable.

Let's take a look of a function in charge of handling an HTTP request for getting scheduled classes of a gym's customer:

```javascript
function getScheduledClasses(req, res) {
    if (req.user) {
        if (req.user.hasPaidMembership) {
            const classes = getScheduledClassesForUser(user);
            if (classes.length > 0) {
                res(200, { classes });
            } else {
                res(204); // No content
            }
        } else {
            res(403, { reason: 'not_paid_membership' });
        }
    } else {
        res(401); // Unauthorized
    }
}
```

By swapping the conditional to **always return early the logic with less processing**, we can rewrite this as follow:

```javascript
function getScheduledClasses(req, res) {
    if (!req.user) {
        return res(401);
    }

    if (!req.user.hasPaidMembership) {
        return res(403, { reason: 'not_paid_membership' });
    }

    const classes = getScheduledClassesForUser(user);
    if (classes.length === 0) {
        return res(204); // No content
    }

    res(200, { classes });
}
```

This code presents less indentation, which improves readability. In the first snippet, indentation is correlated to conditionals; in the latest, conditions are presented in a procedural way.

Notice that both snippets have the same amount of lines, however the second one has way more new lines to separate the intention of each block.

What do you think of the technique? Have you used before? Do you think it may make sense applied to your codebase?
