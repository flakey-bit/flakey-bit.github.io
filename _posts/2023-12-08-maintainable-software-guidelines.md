---
title: 'Guidelines for maintainable software'
date: 2023-12-08T12:00:00+13:00
author: eddiewould
layout: post
permalink: /2023/12/08/maintainable-software-guidelines/
spay_email:
  - ""
categories:
  - Uncategorized
---

Or "how not to make a mess of things"🥴

## Background

In software development, there are no "rights" and "wrongs", however this post shares _guidelines_ that I've found helpful for keeping your sanity / being able to maintain software over time.

### 1. Keep the business logic (rules) for a given "user flow" in a single process (component)
Importance: **CRITICAL**

This is basically another way of saying "don't create a distributed monolith". In fact, it almost says "don't do microservices"

You need to decide which component "owns" the user-flow (from start to finish). 

If we need to collaborate with other components to get the job done, the collaborator components should act as "dumb data pipes". 

Each interaction with a "dumb data pipe" is limited to ONE of the following:
* Fetch some data
* Store some data
* Transform some data (in a stateless way)
* Perform a single, well-defined side-effect (e.g. sending an email)

These operations should be **simple** and **stable** (very rarely need to change). 

Failing to adhere to this guideline results in systems that are hard to reason about, brittle and difficult to evolve.

If you do adhere to it? You should be able to avoid E2E tests: 
* The dumb data pipe operations should be covered by functional (component) tests by the team that maintains the component
* You can check API compatibility through consumer-driven contract tests

### 2. Push side-effects to the very edges of the application
Importance: **HIGH**

* Strive to model the "guts" (vast majority) of your application as pure functions (no side effects, always returns same output for given input)
* Make any side effects obvious (and ideally idempotent)
* Perform the side-effects & I/O at the edges (e.g. at the very beginning and very end of the interaction ONLY)

The "Execution plan" & "[Humble Object](https://steven-giesel.com/blogPost/47acad0a-255c-489b-a805-d0f46bde23e5)" patterns can help greatly here.

### 3. Take care with state
Importance: **HIGH**

Aim to minimize the amount of state your application has to deal with. Strive to keep operations stateless if possible.

Any state you are left with, make it obvious.

Ideally, embrace immutability *everywhere* (even expose to the end-user) - prefer creating a new version of a record over mutating.

### 4. Lean on the type-system, (compiler)
Importance: **MEDIUM**

To the extent possible, model the business domain in the type system using rich types (e.g. a `Money` type instead of a `decimal` primitive). 

If possible, make illegal states unrepresentable - union (sum) types are your friend here. Most of the time, `boolean` flags on types are bad (if `foo=true` then `baz` should be non-null 🤮)

If your type system is limiting you here, choose a different language (seriously).

Why? If your compiler can catch the mistakes for you, that's a bunch of mistakes you can't make / tests you don't have to write.

### 5. Adhere to command/query responsibility separation (CQRS)
Importance: **MEDIUM**

A given operation should either be a command ("do something", "update something") OR a query ("get something") but never both

### 6. Test your application the same way that it is used (don't over-specify tests)
Importance: **MEDIUM**

"The more your tests resemble the way your software is used, the more confidence they can give you." - Kent C Dodds

The key here is writing tests that give you the freedom to significantly refactor the internal workings (implementation details) without having to update the tests at the same time.

I'm not saying "don't write unit tests", but rather think more carefully about what the units are (usually, they should be significantly bigger than single classes/functions).

* For SPAs, the sweet-spot is integration tests that render the SPA to jsDOM and interact with it via events + accessible role queries.
* For .NET APIs, the sweet-spot is tests using WebApplicationFactory (full controller/filter/middleware pipeline) but with dependencies stubbed at the HTTP level

Good tests are **as vague as possible** while still protecting the user behaviour / business requirements from regressing. Avoid over-specified tests.

### 7. Design for evolvability from the get go
Importance: **MEDIUM**

* If your application persists data, expect it to have to persist _different_ (or additional) data in the future (schema evolution) 
* If you're building an API, design to allow the API to be safely evolved (adding new capabilities) without breaking existing consumers
* If you're _consuming_ APIs, don't couple yourself excessively to the shape/behaviour of those APIs (most of the time, this means adding an anti-corruption layer)

## Summary
Software architecture can have a massive effect on how easy it is to evolve & maintain the system. The guidelines above go a reasonable distance towards the "pit of success".

