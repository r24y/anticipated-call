# anticipated-call

🔜 ☎️ Get a Promise that resolves when your function is called.

[![Build Status](https://travis-ci.org/ryaninvents/anticipated-call.svg?branch=develop)](https://travis-ci.org/ryaninvents/anticipated-call)
[![GitHub last commit](https://img.shields.io/github/last-commit/ryaninvents/anticipated-call.svg)](https://github.com/ryaninvents/anticipated-call/graphs/commit-activity)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)
[![license](https://img.shields.io/github/license/ryaninvents/anticipated-call.svg)](https://github.com/ryaninvents/anticipated-call/blob/develop/LICENSE.md)
[![npm](https://img.shields.io/npm/v/anticipated-call.svg)](https://www.npmjs.com/package/anticipated-call)

## Example usage

`anticipated-call` is intended for use in tests, though of course you may use it anywhere it's useful!

```js
const assert = require('assert');
const anticipatedCall = require('..');

class ExampleClass {
  runDelayedUpdate(value) {
    setTimeout(() => this.performUpdate(value), 1000);
  }

  performUpdate(value) {
    this.value = value;
  }
}

describe('Example test suite', function() {
  it('should perform the update', async function() {
    const ex = new ExampleClass();
    ex.performUpdate = anticipatedCall(ex.performUpdate);

    ex.runDelayedUpdate(37);
    await ex.performUpdate.nextCall;
    assert(ex.value === 37, 'ex.value should equal 37');
  });
});
```

> If you find a novel use case, create an issue on Github and it might get added to the README.

## Requirements

`anticipated-call` intercepts function calls with [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy), and returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise). To do this, the `Proxy` and `Promise` constructors must be available as globals.

If you're running Node 8, these are included in core, so you don't have to do anything. Similarly, if you're using `babel-polyfill` or similar, this is handled for you.

## API

```js
import anticipatedCall from 'anticipated-call';
```

### anticipatedCall(fn)

Wrap the given function to provide the `anticipated-call` methods.

```js
function foo(a, b) {
  return a + b;
}

const wrappedFoo = anticipatedCall(foo);

wrappedFoo.nextCall.then(() => console.log('foo was called!'));
```

If you wish, you may call it with no arguments if you simply need a hook for resolving a promise.

```js
const hook = anticipatedCall();

hook.nextCall.then(() => console.log('hook was called!'));

hook();
```

### nextCall

Wait for the next call of the given function.

```js
function foo(a, b) {
  return a + b;
}

const wrappedFoo = anticipatedCall(foo);

wrappedFoo.nextCall.then(() => console.log('foo was called!'));
```

### nthNextCall(n)

Like `nextCall`, but wait for the function to be called `n` times.

```js
let counter = 0;

const increment = anticipatedCall(() => {
  counter = counter + 1;
});

increment.nthNextCall(3).then(() => console.log(`counter value is ${counter}`));

increment();
increment();
increment();

// Prints `counter value is 3` since it waits for the 3rd call before resolving the Promise.
```

### nextCallDuring(fn)

Wait for the function to be called from a callback.

```js
let counter = 0;

const increment = anticipatedCall(() => {
  counter = counter + 1;
});

increment
  .nextCallDuring(() => {
    counter = 5;
    increment();
  })
  .then(() => console.log(`counter value is ${counter}`));
// Prints `counter value is 6`
```

### nthCallDuring(n, fn)

Like `nextCallDuring()`, but wait for the function to be called `n` times.

```js
let counter = 0;

const increment = anticipatedCall(() => {
  counter = counter + 1;
});

increment
  .nthCallDuring(3, () => {
    counter = 5;
    increment();
    increment();
    increment();
  })
  .then(() => console.log(`counter value is ${counter}`));
// Prints `counter value is 8`
```

## Usage note

In some cases, it's important to keep in mind that the returned Promise will resolve _at the end of the call frame_. Resolving a Promise is an asynchronous operation. Since JavaScript does one thing at a time (for the most part), when the Promise is resolved, it waits for the currently running function to stop executing (for a regular function, this happens on return; for an `async` function, this happens on `await`). If you're tracking state -- for instance, using a variable in a closure -- you might get unexpected results.

Here's an example:

```js
let counter = 0;

const increment = anticipatedCall(() => {
  counter = counter + 1;
});

increment
  .nextCallDuring(() => {
    increment();
    increment();
    increment();
  })
  .then(() => console.log(`counter value is ${counter}`));
// Prints `counter value is 3`... but why?
```

`anticipated-call` was told to wait for the next invocation, but it didn't return until `increment()` had been called _three_ times! This is because the callback inside `nextCallDuring` needed to complete execution before the Promise could resolve.

If the callback were an asynchronous function that yielded execution after the call, it would behave as might be expected:

```js
let counter = 0;

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

const increment = anticipatedCall(() => {
  counter = counter + 1;
});

increment
  .nextCallDuring(async () => {
    increment();
    await delay(0);
    increment();
    await delay(0);
    increment();
  })
  .then(() => console.log(`counter value is ${counter}`));
// Prints `counter value is 1`
```

The purpose of introducing `delay(0)` is to interrupt the call frame to allow the `anticipated-call` to have a chance to respond.

If you're interested in learning more, I suggest reading about the JavaScript event loop ([this article](https://hackernoon.com/understanding-js-the-event-loop-959beae3ac40) is a great start).

## Contributing

This project uses [ESLint-style commit messages](https://github.com/conventional-changelog/conventional-changelog/blob/master/packages/conventional-changelog-eslint/readme.md).
