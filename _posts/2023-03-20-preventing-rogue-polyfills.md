---
title: 'Preventing rogue polyfills'
date: 2023-03-20T12:00:00+13:00
author: eddiewould
layout: post
permalink: /2023/03/20/preventing-rogue-polyfills/
spay_email:
  - ""
categories:
  - Uncategorized
---

ECMASCript properties to the rescue!

## Background

A large web application I was working on had a problem: _sometimes_, a dodgy polyfill would be loaded and would (occasionally) wreck havoc.

> A polyfill is a piece of code (usually JavaScript on the Web) used to provide modern functionality on older browsers that do not natively support it.

MDN: [Polyfill](https://developer.mozilla.org/en-US/docs/Glossary/Polyfill)

Non-realistic example to illustrate the point

```javascript
// Load the polyfill for random function if it's missing
if (typeof Math === "object" && typeof Math.random === "undefined") {
    Math.random = function () {
        // ... some function that approximates a (native) Math.random implementation
    }
}
```

The polyfill in question was for `JSON.stringify` and was (for the most part) coming in via an ancient version of core-js (2.6.12). That version of core-js was included an ECMASCript proposal [that was ultimately rejected](https://github.com/DavidBruant/Map-Set.prototype.toJSON).

The `JSON.stringify` implementation it provided was broken (see [Github issue 148](https://github.com/zloirock/core-js/issues/148)).

Because of the enormous size of the web app, variety of front-end tech stacks used and the nature of transient dependencies, it was very tricky to nail down all the possible places the polyfill could be entering the browser.

## What could we do?

It's been a long-time since `JSON.stringify` has been unsupported or necessary to polyfill - so clearly in the long term, we'd want to track down & remove all occurrences of the polyfill. But in the short-term I proposed the following:

```javascript
(() => {
    const stringify = JSON.stringify;

    Object.defineProperty(JSON, "stringify", {
        enumerable: false,
        configurable: false,
        writable: false,
        value: stringify
    });
})();
```

How does it work?

Well, if you run the following in your browser

```javascript
Object.getOwnPropertyDescriptor(JSON, "stringify")
```

You should see output similar to below:

```
  writable: true,
  enumerable: false,
  configurable: true,
  value: <function>
```

i.e. the `stringify` property on the `JSON` object is both `writable` and `configurable` which basically means that polyfills can mess with it. 

As long as we ensure our script runs **first**, we prevent polyfills from redefining `JSON.stringify`

Nice! 🍻