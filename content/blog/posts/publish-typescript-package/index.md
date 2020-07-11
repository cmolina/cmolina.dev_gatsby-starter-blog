---
title: "Publish a TypeScript package in npm"
date: "2020-07-11T19:23:28.593Z"
description: "Write once, re-use everywhere, by publishing your code in npm!"
---

Would you like to reuse that fancy function from a previous project? I do. All the time. And what I always do is to copy and paste it, and call it a day. But… wouldn't it be nice to _import_ the function?

The advantages are:
- Don't Repeat Yourself (DRY)
- Fixes fixed in one place get propagated everywhere
- New features are just an `npm update` away

So, I'll proceed with 3 steps:
1. Create a TypeScript package
1. Publish a module in NPM
1. Publish an update

Let's see exactly how to do this. FYI, I am following these instructions on the current NodeJS LTS, version 12.


## Create a TypeScript package
I will be creating a new git repository, to then set it up as an npm and TypeScript project.

### Create a git repository
Create a directory for your package and start a git repository in it
```sh
mkdir <your-package-name> && cd <your-package-name>
git init
```

Instruct git to ignore two directories, `node_modules` and `dist` for the dependencies and compiled files, respectively.
```txt:title=.gitignore
node_modules
dist
```

### Initialize an npm package
Initialize an npm package by running `npm init` and answering the questions. Ensure your new `package.json` file contains at least this configuration:
```json:title=package.json
{
    "name": "<your-package-name>",
    "version": "0.1.0",                         /* follow semver.org */
    "description": "<what-this-package-do?>",
    "files": "dist",                            /* these are the files to be published */
    "main": "dist/index.js",                    /* the .js file to be loaded when importing this package */
    "types": "dist/index.d.ts",                 /* the .d.ts file with the type for your package  */
    "scripts": {
        "prepare": "tsc",                       /* compile the files just before publishing */
        "test": "mocha test/**/*.spec.ts"       /* run unit tests */
    },
    "author": "You <you@email.com> (https://you.com/)",
    "license": "MIT"
}
```


This is also a good time to install your dependencies. There are 2 types:
- `devDependencies`, i.e. the packages needed to compile and test your library, and
- `dependencies`, i.e. the packages that need to be deployed alongside your library.

For this example, no `dependencies` will be required, but I will need some `devDependencies` for compiling and running the tests:

```sh
npm i -D /    # install packages as devDependencies
    @types/chai /   # @types/* for better TypeScript experience
    @types/mocha /
    chai /          # BDD assertion library
    mocha /         # test framework
    ts-node /       # transpile TypeScript files for testing
    typescript      # transpile TypeScript files for production
```

After running these commands, make sure to keep track of these changes with git: `git add . && git commit -m "Initialized npm package"`.


### Initialize a TypeScript project
Let's initialize a TypeScript project by running `npx tsc --init`. Ensure your new `tsconfig.json` file contains at least this configuration:

```json:title=tsconfig.json
{
    "include": ["src/**/*"],
    "ts-node": {
        "transpileOnly": true   /* Use TypeScript's faster transpileModule */
    },
    "compilerOptions": {
        "declaration": true,    /* Generates corresponding '.d.ts' file. */
        "outDir": "./dist/",    /* Redirect output structure to the directory. */
    }
}
```

Again, make sure to keep track of these changes with git: `git add . && git commit -m "Initialized TypeScript project"`.


### Write the library
At this point, we can start writing code and its tests. About time!

Create a new file, `src/index.ts`, and write the code. For example, export a function that sums two numbers:

```typescript:title=src/index.ts
export function sum(a: number, b: number): number {
    return a + b;
}
```

For the tests, you can create a spec under the `test` directory, one spec file for each source code.

```typescript:title=test/index.spec.ts
import { expect } from 'chai';
import { sum } from '../src/index';

describe('sum', () => {
    it('should return the sum of 2 positive numbers', () => {
        const result = sum(1, 2);

        expect(result).to.equal(3);
    });
});
```

Before running the tests, let's configure mocha so it understands TypeScript files. Create a new `.mocharc.jsonc` file with this configuration:

```json:title=.mocharc.jsonc
{
    "require": "ts-node/register",  /* load the TypeScript compiler */
    "extension": "ts"               /* consider *.ts files */
}
```

Now you are ready to run `npm test` and see your test passing. If everything went well make a new commit: `git add . && git commit -m "Initial code, unit test passing"`.


## Publish a module in NPM
You will need to log in with your npm account (and create one if you don't have it yet), ensure everything looks good, and finally publishing your package.


### Login to npm CLI
If you do not already have an npm account, go to https://www.npmjs.com/signup and fill the form.

In a terminal run `npm login` and fill with your username and password when requested.


### Update your package name (optional, recommended)
Every time a package is published in npm it goes directly into the global scope. And because every package needs to have a unique name, it can be challenging to find an available name for your new package.

To avoid this problem I recommend publishing your package under your username scope (or an org scope if you belong to one). To do this, go back to your `package.json` and change the `name` property,

```diff:title=package.json
{
-    "name": "<your-package-name>",
+    "name": "@<your-username>/<your-package-name>",
    // ...
}
```


### Ensure you have correctly emitted types
Now that you have your code written and tested, it is time to publish it. To see if everything is in order,

1. run `npm link` to make your unpublished package available locally
1. in another directory, run `npm link @<your-username>/<your-package-name>`
1. create an `index.ts` file, and try to import the functions from your package

If everything went well, you will see the editor consuming the types you had specified 😁


### Add a tag
People usually add tags to their git commits associated with a new version of a package so it can be easily referenced later: `git tag v0.1.0`


### Publish it!
Run `npm publish --access public`. This `access` parameter indicates you will publish your package for everyone with access to npm.

If no errors were published in the terminal, you should be able to see your package listed under `https://www.npmjs.com/package/@<your-username>/<your-package-name>`.

For example, I followed the instructions from this post to publish a logging library mentioned in [a previous post](/posts/logging-functions/) and this is the URL of the package: https://www.npmjs.com/package/@cmolina/log


## Publish an update
After publishing the first version of the package you may be wondering how to send an update. Well, you have to commit your changes, ensure every test is passing, compiling, bumping the version, adding a tag… it is pretty easy to forget a step.

Good news we have `np`, an npm package for publishing to npm.

After committing your changes, run `npx np` and answer the questions. The first time I run it I had to update `npm` and made me have a remote repository as well (I chose GitHub).

## Conclusion
You learned how to create a TypeScript library from scratch, how to manually publishing a new npm package, and to keep it up to date.

Happy publishing!
