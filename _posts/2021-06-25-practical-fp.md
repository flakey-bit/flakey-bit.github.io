---
title: 'Practical functional-programming part 1'
date: 2021-06-25T10:35:00+12:00
author: eddiewould
layout: post
permalink: /2021/06/25/practical-fp-part-1/
spay_email:
  - ""
categories:
  - Uncategorized
---

An introduction to functional-programming: ideas that can be applied to every-day LoB application development.

I've recently read the book [Functional programming in C#](https://www.manning.com/books/functional-programming-in-c-sharp) by Enrico Buonanno - I highly recommend reading it. This post concentrates on the ideas from the book that _I_ found most valuable, as well as some ideas from other sources. 

## Introduction
You've probably heard of functional programming (often abbreviated as "FP") before, but perhaps you've been put off by complicated geeky terms like "Lambda Calculus", "Algebraic Data Type" or the dreaded m-word (..."Monad" 😱). 

<figure class="wp-block-image size-large"><img src="/images/posts/practical-fp-part-1/monad-monad-monad.png"/></figure>

Yes, the ideas are rooted in mathematics however there's still really valuable stuff you can draw on without paying too much attention to the theory. I'll try and present what *I* think are the most useful ideas, without getting too bogged down.

### What is FP, in a nutshell? 
Admittedly a bit of a cop-out, but I'll start by contrasting functional programming (FP) with object-oriented (OO) programming - which I assume you're familiar with.

In the object-oriented world, our basic building-blocks (that we compose our applications from) are _object instances_. An object instance encapsulates both behaviour _and_ state (data) _together_. We call methods *on* such objects to 
* Modify the object's internal state
* Perform computations
* Trigger side-effects ("fire the missiles!") 

An object method is a function that is _bound_ to a given instance - that is to say, in addition to any parameters explicitly supplied to the method, the method can also utilize (and modify!) fields on the object itself.

By way of contrast, in the functional-programming world our building blocks are _functions_. Functions are not bound to an object - all they have to work with is the parameters they were explicitly supplied. This characteristic makes functions easier to safely modify than methods.

Note for C# programmers: an unbound function corresponds to a `static` method. 

In the object-oriented world, we frequently encounter methods that
* Accept objects (not just primitive values) as parameters
* Return an object (rather than a primitive value)

There is a symmetry in the functional-programming world - we have functions that
* Accept other functions (not just primitive values) as parameters
* Return a function (rather than a primitive value)

Such functions are known as _Higher order Functions_ (HoFs). HoFs are the primary means for code reuse in functional programming.

For the C# programmers out there, an every-day example of a HoF can be found in LINQ:

```csharp
var numbers = Enumerable.Range(1, 10);

// Create a function with the signature string → bool
// (i.e. takes a single string argument and produces a boolean return value)
Func<int, bool> isEven = theNumber => theNumber % 2 == 0;

// Invoke the LINQ Where method, passing the function in as an argument (the predicate)
var evenNumbers = numbers.Where(isEven);
```

Because `Where` takes a function as an argument, it is a HoF. 

By allowing the caller to pass a predicate function (e.g. `isEven`) to the `Where` method, the designers of LINQ have enabled significant extensibility; rather than trying to anticipate the filtering operations that might be needed up-front, they allow the user to "plug in" the filtering strategy - c.f. the [StrategyPattern](https://en.wikipedia.org/wiki/Strategy_pattern).

At this point you're probably thinking that dealing solely in terms of primitive data types and functions to operate on them would be hugely limiting - and indeed it would be! FP *does* have composite types, the distinction is that types don't have any _behaviour_ associated with them - they're *just data*. The section "Algebraic Data Types" covers composite types in more detail. 

It's interesting to note that in object-oriented programming, the principal of _encapsulation_ tells us to avoid creating types w/ *no behaviour* (see the [AnemicDomainModel](https://martinfowler.com/bliki/AnemicDomainModel.html) anti-pattern), whereas in functional programming, *it's the norm*. Encapsulation is a core pillar of object-oriented programming - a well-designed object
* Offers a minimal public interface (API)
* Hides implementation details
* Is responsible for protecting its own internal state & invariants
Encapsulation primarily helps with
* Enabling code reuse
* Reducing cognitive load for developers
* Ensuring correct behaviour / reducing bugs 

In functional programming, the same outcomes are achieved through _different_ means - primarily: immutable types, pure-functions & leaning heavily on the type system - these ideas are explored in the remainder of the post.

So to summarise (& grossly over-simplify) functional programming:
* It places a heavy emphasis on _data_ - how data flows and is transformed
* Functions are first-class things that we pass around like any other kind of parameter
* We don't mush-together state and behavior
* We build the overall behaviour by combining functions

### What are some of the core concepts from FP?

#### Immutability

To put it simply, immutability is the idea that once a value is created, _that_ (particular) value may never change; we may only create _new_ values. As a contrived example, in an OO program you might have a `FinancialAccount` object, with properties 


Ties in nicely with value-semantics

TODO TODO TODO

#### Prefer "value semantics" (even for reference types)

In C#, we have a dichotomy of "reference type" vs "value type"

For *reference types* (e.g. objects, arrays and strings):
* The data is stored on the heap (it could be large)
* Variables which contain a reference type really only contain a *reference* to the value (i.e. memory location)
  ```csharp
  int[] itemsA = { 1, 2, 3 };
  // itemsB refers to the same object in the heap as itemsA
  int[] itemsB = itemsA;
  // Will update itemsA[2] too, since they're the same array in the heap
  itemsB[2] = 5;
  ```
* "Reference equality" is used by default
  ```csharp
  int[] itemsA = { 1, 2, 3 };
  int[] itemsB = { 1, 2, 3 };
  Console.Out.WriteLine(itemsA == itemsB); // false
  ```

For *value types* (e.g. structs, most primitives):
* The data is stored on the stack (value types are typically small in size)
* Variables which contain a value type *actually contain* the value itself
  ```csharp
  int x = 5;
  int y = x;
  // Update x after the assignment of x to y. The update to x won't propagate to y
  x = 6;
  Console.Out.WriteLine(y); // 5
  ```
* "Value equality" is used by default
  ```csharp
  Guid guidA = Guid.Parse("1698135c-da61-4f6a-b8e8-506936632a66");
  Guid guidB = Guid.Parse("1698135c-da61-4f6a-b8e8-506936632a66");
  Console.Out.WriteLine(guidA == guidB); // true
  ```

As mentioned previously, in FP we avoid "mushing together state and behavior". As a consequence of that, the type *is* the data it is comprised of. Therefore, the most sensible definition of equality to use is value-based equality: If the constituent parts of two values are equal, then the two values should also be equal.

When we create an object in C# by "newing up" a class, the value we get back is (by definition) a reference type and (by default) will have reference-based equality. However, it is possible to define the class in such a way that it behaves more like a value type (primarily, by overriding `operator ==()` and friends) in terms of equality.

In C# 9.0, [records types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/equality-operators#record-types-equality) support the `==` and `!=` operators, automatically providing value equality semantics.


#### Pure functions
A function call is pure if you can replace the function call with the pre-computed result *without affecting behaviour*. For a function to be pure, it must adhere to the following:
* The function return value must be *entirely* based on the input parameters it receives
  * These input parameters may only be data or other pure functions
* It must not mutate (modify) any of its input parameters
* It must not trigger any side-effects (such as writing to disk, network calls etc)

NB: You might encounter a similar term "referential transparency" - which is essentially the same thing but a weaker guarantee as it allows _insignificant_ side-effects (such as writing to the console or logging).

* A function `md5Sum` that computes the MD5 hash of a given input string is pure as you can replace the function call with the pre-computed MD5 hash for the string.
* `DateTime.Now` (which returns the current system time in C#) and `Guid.NewGuid` (generates a new GUID) are *not* pure because each time you call them you get a different result.
* A function `createPerson` that takes a couple of strings (`firstName` and `lastName`) as input parameters and combines them into a data-structure including a GUID `personId` (generated with `Guid.NewGuid`) is *not* pure because `createPerson` _calls_ an impure function.
* A function `addToCart` which takes a shopping-cart data structure `cart`, a `productId` and `quantity` and updates the cart in-place is *not* pure, because it mutates the `cart` parameter.
  * If instead `addToCart` returned a *new* cart (rather than updating in-place) then it would be pure.
* A function `calculateRiskProfile` which transforms its input parameters, makes a HTTP `GET` web-service call and massages the response from the web-service is *not* pure because of the web-service call:
  * the web-service is a black-box and it's implementation could change at any point in time;
  * the web-service call goes over the network. The network could be down, the request could time-out etc. 

Pure functions are great because:
* They're super easy to test. You can literally treat them as a black-box "does the function do what it says on the tin?" - given these inputs, does it produce the correct output.
  * Additionally, intermediate results from a chain of pure functions makes finding a [seam](https://www.informit.com/articles/article.aspx?p=359417&seqNum=3) trivial!
* They make code easy to reason about. The signature of the function (inputs & output types) largely describes what the function does. Avoiding [primitive obsession](https://wiki.c2.com/?PrimitiveObsession) helps even further in this regard.
* They facilitate parallelization. If you have an array of items to be processed and a function `processItem` which takes a single such item as an input, you can spin up lots of threads/tasks and give them a chunk of items, not needing to worry about interactions between the calls. This kind of code is extremely scalable.
* They're easily reusable. Since the function is guaranteed not to have any unwanted side-effects (by definition!) you can reference it anywhere you need it.
* The results from a pure-function can be cached indefinitely! There's no need to worry about the results becoming stale.

See this [post](https://medium.com/@juntomioka/why-pure-functions-are-so-good-7f7759021c35) for more on the benefits of pure functions.

Note for C# programmers: You could consider decorating methods with the `[Pure]` attribute to indicate intent to other developers.

If you're a FP practitioner, _most_ of the code you write will be expressed in pure-functions - which leads us to the next section: Isolation of side-effects.

#### Isolation of side-effects
So you're following along, you've probably concluded
* Pure functions = good
* Side effects = bad

But as it turns out, side-effects are a necessary evil. All programs (except toy ones) need to do at least one (and often several!) of the following to be useful:
* Read input from disk / network / keyboard
* Write output to the screen / disk
* Communicate with another program or system (network)

We want to keep as much of our codebase functionally "pure" as possible (for the reasons/benefits listed in the previous section). The basic strategy is to *push side-effects to the very edges* (of the program). 

There's a great blog-post [clean and green](http://drocco007.github.io/2015_pytn/clean_and_green.html) which coins the term "functional core + imperative shell". The basic ideas being
* Separate policies from mechanisms and pass simple data structures between the two
* The imperitive-shell is procedural "glue" code that offers an OO interface & manages dependencies (mechanisms)
* The functional core (expressed in pure functions) implements all the decisions (policies)
* Never mix decisions and dependencies

The idea is that the *majority* of the application (_especially_ the complex business logic - the "core") is written in a functional style while the edges / interface to the outside world (the "ports") are written in an object-oriented or imperative style - keeping us in functional land as much as possible.

I've also previously [blogged](2019/10/17/writing-testable-software/) about a similar idea which I coined the "execution plan pattern":

> Essentially, the idea is to split figuring out "what needs to be done" from actually doing it. This pattern works particularly well when the “figuring-out” bit is complex / full of business logic. 

One trade-off is that you often end up fetching more data than you need - fetching data is a side-effect, and so we want to do it all at once (at the "top") to avoid the possibility of having to make a subsequent fetch later on.

The are of course other, more formal ways for isolating side-effects in FP (such as the "Effect" monad). But the principal is more important than the specifics of how to achieve it.

It should now be clear that adopting FP is not a case of "all or nothing". You'll often find that certain parts of a program will lend themselves more to FP than others. Be practical about it rather than dogmatic, but make sure it's clear which parts are written in a functional style and which are not (to help with maintainence).

#### Idempotency

TODO

#### Leaning on the type system

TODO: leaning heavily on the compiler (type system) to help prove the correctness of your program.



Now that we have the core concepts of FP out of the way, let's drill into some more detail.

### 

* Introduction (not "all-or-nothing", functional core with imperative shell. Yes it's rooted in maths)
* Functions as data
* Opinion: Recoverable (unchecked) exceptions for (flow control) are evil. Out of memory, out of disk space, assertion exception
* The problem with null return values. Actually it's just a special case of the general problem of code might not handle all possible return values.
* Inverting control to get compile-time safety (basically, you can't get at the result unless you promise to deal with or at least acknowledge the edge cases) 
* Option aka Maybe
* Either - and brief segue into union types vs product types (algebraic data types. This post has more info: https://jrsinclair.com/articles/2019/algebraic-data-types-what-i-wish-someone-had-explained-about-functional-programming/). Option and Either are both examples of ADTs. Useful in business domain too - preventing invalid states.
- https://jrsinclair.com/articles/2019/algebraic-structures-what-i-wish-someone-had-explained-about-functional-programming/ railway-oriented-programming: https://fsharpforfunandprofit.com/rop/#slides

#### Algebraic Data Types

We need a section on this since we refer to it


Reference https://github.com/louthy/language-ext 

> This library uses and abuses the features of C# to provide a functional-programming 'base class library' that, if you squint, can look like extensions to the language itself. The desire here is to make programming in C# much more reliable and to make the engineer's inertia flow in the direction of declarative and functional code rather than imperative

And also https://github.com/emmanueltouzery/prelude-ts 

> prelude-ts (previously prelude.ts) is a TypeScript library which aims to make functional programming concepts accessible and productive in TypeScript. It provides persistent immutable collections (Vector, Set, Map, Stream), and constructs such as Option, Either, Predicate and Future. 

Other articles / posts: 

https://www.yld.io/blog/the-not-so-scary-guide-to-functional-programming/
https://cscalfani.medium.com/why-is-learning-functional-programming-so-damned-hard-bfd00202a7d1