---
id: 2142
title: Writing testable software
date: 2019-10-17T20:55:29+10:00
author: eddiewould
layout: post
guid: https://eddiewould.com/?p=2142
permalink: /2019/10/17/writing-testable-software/
ocean_gallery_link_images:
  - 'off'
ocean_sidebar:
  - "0"
ocean_second_sidebar:
  - "0"
ocean_disable_margins:
  - enable
ocean_display_top_bar:
  - default
ocean_display_header:
  - default
ocean_center_header_left_menu:
  - "0"
ocean_custom_header_template:
  - "0"
ocean_header_custom_menu:
  - "0"
ocean_menu_typo_font_family:
  - "0"
ocean_disable_title:
  - default
ocean_disable_heading:
  - default
ocean_disable_breadcrumbs:
  - default
ocean_display_footer_widgets:
  - default
ocean_display_footer_bottom:
  - default
ocean_custom_footer_template:
  - "0"
ocean_link_format_target:
  - self
ocean_quote_format_link:
  - post
categories:
  - Uncategorized
tags:
  - testing unit-testing solid pyramid side-effects functional design pattern refactoring state singleton idempotent immutable
---

Software designs/architecture that lead to testable software - and why that's desirable.

## Motivation

Why should we bother? There are several motivations for writing (unit) tested software. 

* It gives you confidence that what you're deploying to production will do what it's supposed to do under all conditions. Especially true if you're delivering (potentially) <a href="https://en.wikipedia.org/wiki/Therac-25">lethal doses of radiation</a> or something like that
* It gives you confidence to refactor (change the structure/design/implementation) of the software and not break things
* It provides living documentation of how the software is intended to be used (we all know how much devs love writing let-alone updating documentation)
* It encourages you to think about possible edge cases / failure modes

<figure class="wp-block-image is-resized"><img src="/wp-content/uploads/2019/10/thinking.jpeg" alt="" class="wp-image-2145" width="210" height="280"/></figure>

However it's worth mentioning that even if you don't bother writing any tests, *it's worth writing your code with testing / testability in mind*. Some benefits of doing so:

* It will tend to lead to code that adheres to the <a href="https://en.wikipedia.org/wiki/SOLID">SOLID</a> principles:
  * If you've got seemingly unrelated tests sitting next to each other, it's likely you've violated the single-responsibility principle
  * If you find setting up mocks/stubs unweildly, you've probably violated the interface segregation principle
  * If you find yourself testing/relying on implementation details of collaborators, you've likely violated the dependency inversion principle
* It generally encourages good designs / practices
  * Use of dependency injection
  * Use of immutable data structures
  * Isolation of side effects<
* It tends to produce code that is easy to understand and modify

## Eat your greens (unit tests)!

Growing up, you've probably been taught about the healthy-eating pyramid (eat lots of grains/vegetables, eat a moderate amount of meat/dairy and eat only a little fat/salt/sugar). 

We can apply a similar philosophy to testing:

<figure class="wp-block-image"><img src="/wp-content/uploads/2019/10/Drawing-1.png" alt="" class="wp-image-2144"/></figure>

The idea is that at the higher levels of the pyramid, tests become *increasingly difficult to write*/perform and also *increasingly difficult to maintain* / fragile. Thus, we should write:

* *Most* of our tests as "unit tests" - in the context of OOP that means testing individual classes so that they conform to their interface / spec. This would typically cover core business logic and algorithms.
* *Quite a few* integration / cross layer tests. For example tests that actually write to a database, execute a controller middleware pipeline etc. Typically this will be areas where we can no-longer avoid side-effects and/or we must pay attention to cross-cutting concerns (logging, authentication etc)
* *A small number of key* end-to-end system tests that demonstrate the system as whole integrating with the outside world / other systems. This should cover some key user stories, that the app functions as a whole, IoC container configuration and UI testing if you must.
* *Manual tests *only when other forms of testing are *infeasible*, or to provide a safety blanket.

I'll be focusing on unit tests for the remainder of this post.

## Unit testing - what makes code difficult to test?

In my experience, there are _three_ primary factors that drive the difficulty of writing unit tests. 

#### Object State

In the tradtional OOP methodology, a "object" combines _data_ (date of birth, weight, height, sex) with _functionality_ (`CalculateLifeExpectancy`) in an encapsulated package (a "Person"). The object is responsible for preventing its internal state from becoming invalid. The run-time behaviour of an object's functionality can depend on its internal state (for example as you put on weight, your life expectancy goes down).

<figure class="wp-block-image is-resized"><img src="/wp-content/uploads/2019/10/man-2288176_640.png" alt="" class="wp-image-2147" width="268" height="214"/></figure>

 From a testing perspective, that means we need to 

* Consider the implications that the internal state has on behavior
* Ensure we can get the object into the required states for testing (so that we can prove it behaves as expected)<

If the internal state is complicated and/or dificult to script, you'll have a hard time writing tests. 

#### Static Variables / Singleton / External State

In well architected OOP code, it should be possible to determine an object's collaborators by examining it's construtor and instance-method arguments.

Static variables / Singleton allows seemingly *disconnected parts of an application to communicate as-if by magic*! These invisible dependencies are problematic:

* It's easy to make changes that have unintended consequences in a completely unrelated area
* If multiple tests are run "in sequence", the order of the tests could impact the results (leading to flakey tests)
* The static-ness is like a zombie-apocalypse - *all data accessible* from a Singleton reference (even otherwise well-written, encapsulated objects) *effectively becomes static (!)*

<figure class="wp-block-image is-resized"><img src="/wp-content/uploads/2019/10/zombie-2799591_960_720.png" alt="" class="wp-image-2149" width="159" height="271"/><figcaption>Watch out, or the static-apocalypse will get you!</figcaption></figure>

It's also worth keeping an eye out for dependencies on external / global state. Examples include

* Data in a database
* Data returned from an external web-service call
* Data in a file

Unless care is taken to sure these are in a consistent, known state prior to running each test, you will be hurting. Tests depending on external state are really integration tests IMHO.

#### Side-effects

A "side effect" is an operation that makes a method/function "impure" from a functional programming perspective. 

A "pure" function takes inputs and returns an output (without mutating the inputs or any external state). 

Side effects boil down to

* I/O operations (reading/writing to disk/database, invoking web service, adding to a queue)
* Mutating external state (modifying arguments / static variables
* Calling another impure function

Code that causes (and in particular _depends on_) side effects becomes painful to test. It's also hard to reason about.

Side-effects are unavoidable (that spreadsheet is not going to be very useful if you can't save) but it's worth thinking about *isolating* and *limiting* side effects. 

## What can we do to make writing unit tests easier?

The meat of the blog post:

<figure class="wp-block-image is-resized"><img src="/wp-content/uploads/2019/10/ribs-1024x576.jpg" alt="" class="wp-image-2153" width="253" height="142"/></figure>

#### Avoid static variables / the Singleton "pattern" like the plague. 

A unit test should act as a <a href="https://blog.ploeh.dk/2011/07/28/CompositionRoot/">composition root</a>. All objects that are collaborators to the SUT (system-under-test) should be explicitly wired up *in the test*. If you've got the disease, systemically work to remove it.

#### Isolate side effects from business logic

One way to achieve this is what I call the "ExecutionPlan" Pattern. 

Essentially, the idea is to *split figuring out what needs to be done from actually doing it*. This pattern works particularly well when the "figuring-out" bit is complex / full of business logic.

```csharp
// Perform some initial side effects, fetch *all* required data up-front
var txn = _repo.BeginTransaction();
var data1 = _repo.FetchAllTheThings(); // data is DTO/POCO/POJO/POPO
var data2 = _repo.FetchMoreThings();

// Generate the plan. This is where all your complex business logic lives
// and is where your unit testing will be focused.
// Note that GeneratePlan is functionally pure and could be static.
var planner = new Planner(); 
var plan = planner.GeneratePlan(data1, data2); // plan is DTO/POCO
Console.Out.WriteLine(plan.ToString());

// Execute (commit) the plan (final side effects)
_repo.ExecuteTransaction(plan, txn);
```

Since the planner / plan / input data are just plain C# (Java/Python) objects, it's super easy to test that *the correct plan has been generated for various combinations of input data by inspecting it*. The exact nature of the plan will depend on the context of what you're doing, but will essentially be a *sequence of side-effects to execute*.

In an ideal scenario, the actual execution/commit of the (generated) plan is *so straight-forward* that it doesn't even warrant testing. But if it does, it can be covered with a small set of integration tests (covering the possible steps a plan might contain) - significantly reducing the number of pesky `Verify()` calls that you need.

Note that although the (pseudo-code) example relates to databases, the pattern is generally applicable (e.g. making an update via a web-service).

As an aside, I've found the <a href="https://docs.microsoft.com/en-us/dotnet/api/system.data.datatable?view=netframework-4.8">DataTable</a> classes in .NET super-helpful for implementing "execution plan" stuff around DB changes.

Even if you don't create an execution-plan as an entity, *try and limit side-effects & I/O to the "top" and "bottom" rather than sprinkling them throughout the method / call graph*.

#### Make side-effects idempotent for consumers if possible

Let's say your business logic needs to update the DNS entries in the <a href="https://en.wikipedia.org/wiki/Hosts_(file)">hosts</a> file in Windows. You look for a library to do it for you:

As the _consumer_ of the library, the ideal abstraction would _probably_ be a function that adds a host entry, only if the entry doesn't already exist - something like `EnsureHostsEntry(ipAddress, dnsName)`.

Idempotent functions such as this move complexity/conditional logic away and let you get on with the job.

#### Make setting up required state easy

* Make use of the <a href="https://martinfowler.com/dslCatalog/constructionBuilder.html">builder</a> and/or the <a href="https://martinfowler.com/bliki/ObjectMother.html">object mother</a> pattern for creating test data-objects
* Consider adding operations to an object to allow rehydrating complex state
  * Take care to ensure the object takes responsibility for validating its rehydrated state

#### Try to keep the most complex logic in "pure" functions

Pure functions are super easy to test because they don't have side effects or depend on external state. Or if not pure functions, at least try and "promote" I/O upwards. This <a href="http://drocco007.github.io/2015_pytn/clean_and_green.html">blog post</a> covers it fairly well.

#### Make use of immutable types wherever possible

Immutable types are great because they're easy to reason about and are a natural fit for working with pure functions (where we want to avoid mutating the inputs).

#### Watch out for code that needs SomeMethod.Verify() to test

This is almost-always a code-smell in my experience - a violation of the Liskov substitution & Dependency inversion principles from SOLID. 

Instead of checking for method calls, *focus assertions on data* (apply the "Execution Plan Pattern" as appropriate).

Hope this helps you to write better code!