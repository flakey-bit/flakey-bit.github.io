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

## Introduction

In software engineering, there are no _absolute_ "rights" and "wrongs" - but we can still look for things that are **useful most of the time** (guidelines). 

This post shares _six principals_ (wisdom) that will help you produce testable, maintainable & _evolvable_ software - because software that people are afraid to change [quickly becomes legacy software](https://software.rajivprab.com/2019/11/25/the-birth-of-legacy-software-how-change-aversion-feeds-on-itself/).

### 1. Keep the business logic (rules) for a given "user flow" in a single process (component)
Importance: **CRITICAL**

This is basically another way of saying "don't create a distributed monolith". In fact, it _almost_ says "don't do microservices"🌶️. 

You need to decide which component "owns" the user-flow (from start to finish). 

If we need to collaborate with other components to get the job done, the collaborator components should act as "dumb data pipes". 

Each interaction with a "dumb data pipe" is limited to ONE of the following:
* Fetch some data
* Store some data
* Transform some data (in a _stateless_ way)
* Perform a single, well-defined side-effect (e.g. sending an email)

These operations should be **simple**, **stable** (very rarely need to change) & **general** (the collaborating components should **not** know the _specifics_ of the business domain).

Failing to adhere to this principal results in systems that are hard to reason about, brittle and difficult to evolve.

If you **do** adhere to it? You should be able to avoid E2E tests🎉 
* The dumb data pipe operations should be covered by functional tests by the team that maintains the component providing the operation
* You can check API compatibility through consumer-driven contract tests

#### "But what about event-driven architectures"? 
They're all the rage right now...

Nothing I've said here necessarily _precludes_ event-driven architectures:
* You can have two components listening for the same event and starting their own (independent) flows based on the event
* Upon completion of a flow, the component may dispatch an event. This event may trigger a _new_ flow **in another domain**.
  * Because the flow is in a _different domain_, it's reasonable (expected) that a **different** component responds to the new event

As a rule-of-thumb 👍, it shouldn't matter (to the business/user) **if there is a significant delay** between the event being dispatched and the new flow starting. 

If (on the other hand) such a delay would be unacceptable, the "separate flows" probably aren't separate at all.    

### 2. Push side-effects to the very edges of the application
Importance: **HIGH**

* Strive to model the "guts" (vast majority) of your application as _pure functions_ (no side effects, always return same output for given input)
* Make any side effects obvious (and ideally idempotent)
* Perform the side-effects & I/O at the edges (e.g. at the very beginning and very end of the interaction/flow ONLY)

The "[Humble Object](https://steven-giesel.com/blogPost/47acad0a-255c-489b-a805-d0f46bde23e5)" & "Execution plan" (which I've [previously blogged](/2019/10/17/writing-testable-software/) about) patterns can help here.

#### Humble Object
If you have some difficult-to-test code (usually because it's interacting with the outside world, e.g. I/O), **don't** mix that with business logic. 

Keep the [cyclomatic complexity](https://www.geeksforgeeks.org/cyclomatic-complexity/) as low as possible - that basically means avoid conditionals (`if`/`else`) and looping in that code and instead, **just** have your humble-object do the difficult-to-test thing.

If you feel the need, you can write **maybe one or two tests** against the humble object directly - but often, it won't be necessary. The rest of your tests can simply check that the humble-object was called with the expected arguments.

#### Execution Plan
The idea is to split figuring out "what needs to be done" (the _plan_) from actually **doing** it (the _execution_). 

The generated plan should be a sequence of _steps_ - each step is a simple description of an action to perform.

In real terms, that means for an interaction/flow:

1. Some dead-simple code to fetch the initial state/inputs (ideally, via a humble object)
2. Pass the inputs into a **pure function** which generates the plan. The planner should not perform side-effects or perform additional I/O!
3. Finally, _execute_ the plan. For each step, perform the action described using (you guessed it!) a humble object

The code to generate the "plan" (from data) is functionally pure (but potentially complex), while the code that executes the plan is _impure_ (but simple).

### 3. Take care with state
Importance: **HIGH**

State is a liability. Aim to minimize the amount of state your application has to deal with (if something can be computed, **don't** store it). Strive to keep operations stateless if possible.

Any state you are left with, make it obvious.

At the very least, embrace immutability in your application code. However, consider exposing it to the end-user too (they get versioning as a feature!)

### 4. Lean on the type-system, (compiler)
Importance: **MEDIUM**

To the extent possible, model the business domain in the type system using rich types (e.g. a `Money` type instead of a `decimal` primitive). 

If possible, make illegal states unrepresentable - union (sum) types are your friend here. Most of the time, `boolean` flags on types are **bad** (if `Foo=true` then `Baz` should be non-null 🤮)

If your type system is limiting you here, choose a different language (seriously).

Why? If your compiler can catch the mistakes for you, that's a bunch of **mistakes you can't make** (tests you don't have to write). See my post [A gentle introduction to functional programming
](/2022/09/26/a-gentle-introduction-to-functional-programming/) for more on this topic.

### 5. Test your software via the public surface
Importance: **MEDIUM**

> [The more your tests resemble the way your software is used, the more confidence they can give you](https://twitter.com/kentcdodds/status/977018512689455106)." - Kent C Dodds

**Unless** your software is a reusable library, your **users don't create objects/call methods/dispatch actions**.

Rather, the _surface_ they interact with is:
* Clicking buttons / filling in fields (if your software is a web-app)
* Calling HTTP endpoints (if your software is a REST API)

If your software **is** a reusable library, your users still only interact with the _exported_ (public) API of the library.

So long as we keep the observable behaviour at the surface stable/compatible, we're free to change anything **below** that surface (implementation details). This is a huge advantage in terms of being able to maintain the software. 

Where possible, strive to write tests that use the software like a user would. That ensures you're testing the actual user experience, but also gives you that refactoring freedom. 

If you find yourself changing the tests in lock-step with the implementation, it's usually a sign you're testing implementation details.

#### The sweet spot
Kent Beck [described](https://tidyfirst.substack.com/p/desirable-unit-tests) desirable properties of tests. I've listed the ones I think that are **most important** below:

- [Isolated](https://www.youtube.com/watch?v=HApI2cspQus) — Can run the tests 100% in parallel without impacting each other
- [Deterministic](https://www.youtube.com/watch?v=PwWyp-wpFiw) — If the code being tested hasn't changed, the test always produces the same result
- [Behavioral](https://www.youtube.com/watch?v=5LOdKDqdWYU) — If the behavior changes accidentally, a test should fail
- [Fast](https://www.youtube.com/watch?v=L0dZ7MmW6xc)
- [Predictive](https://www.youtube.com/watch?v=7o5qxxx7SmI) — Tests failing should give great confidence that the whole system is **not** working
- [Inspiring](https://www.youtube.com/watch?v=2Q1O8XBVbZQ) — A frequently-run unit test suite gives great confidence the programming is progressing

I've found that "integration" tests usually hit the sweet-spot: 
* For SPAs, render the SPA using jsDOM and interact with it via events + accessible role queries. Mock requests/responses with the backend
* For .NET APIs, test using `HttpClient` from `WebApplicationFactory` (full controller/filter/middleware pipeline) but with dependencies stubbed at the HTTP level

The _reason_ is that they exercise as much of the application as possible while keeping to a single thread/process.

As soon as multiple processes are involved, isolation/speed/determinism is compromised. 

#### A note on unit tests
I'm not saying "don't write unit tests", but rather **think more carefully about what the units are**. A common trap is that "unit" = class/method/function. I prefer the notion that a unit represents a behavior - for example

> Clicking a column header (e.g. "Total") sorts the table according to the values in that column (descending)

A "unit" **may involve a number of functions / classes** etc; anything below the top surface of the unit is still an implementation detail.

I think **a useful rule-of-thumb** is "Could I conceive exporting this functionality as a reusable library"? 
* If so, it's probably a good candidate for testing (as a unit)
* If not, it might be an implementation detail

I'll finish by stating that good tests are **as vague as possible** (whilst still protecting the user behaviour / business requirements from regressing).

### 6. Design for evolvability from the get go
Importance: **MEDIUM**

* If your application persists data, expect it to have to persist _different_ (or additional) data in the future (schema evolution) 
* If you're building an API, design to allow the API to be safely evolved (adding new capabilities) without breaking existing consumers
* If you're _consuming_ APIs, don't couple yourself excessively to the shape/behaviour of those APIs (most of the time, this means adding an anti-corruption layer)

## Summary
Software architecture can have a significant effect on how easy it is to evolve & maintain the system. The guidelines above go a reasonable distance towards the "pit of success".

