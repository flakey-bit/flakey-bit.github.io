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

Introduction to functional-programming ideas that can be applied to every-day LoB application development.

I've recently read the book [Functional programming in C#](https://www.manning.com/books/functional-programming-in-c-sharp) by Enrico Buonanno - I highly recommend reading it. This post concentrates on the ideas from the book that _I_ found most valuable, as well as some ideas from other sources. 

## Introduction
You've probably heard of functional programming ("FP") before, but perhaps you've been put off by complicated geeky terms like "Lambda Calculus", "Algebraic Data Type" or the dreaded m-word (...Monad 😱). 

<figure class="wp-block-image size-large"><img src="/images/posts/practical-fp-part-1/monad-monad-monad.png"/></figure>

Yes, the ideas are rooted in mathematics however there's still really valuable stuff you can draw on without paying too much attention to the theory. I'll try and present what I think are the most useful ideas, without getting too bogged down.

### What is FP, in a nutshell? 
Admittedly a bit of a cop-out, but I'll start by contrasting functional programming (FP) with object-oriented (OO) programming - which I assume you're familiar with.

In the object-oriented world, our basic building-blocks (that we compose our applications from) are _object instances_. An object instance encapsulates both behaviour _and_ state (data) _together_. We call methods *on* such objects to 
* Modify the object's internal state
* Perform computations
* Trigger side-effects ("fire the missiles!") 

An object method is a function that is _bound_ to a given instance - that is to say, in addition to any parameters explictly supplied to the method, the method can also utilize (and modify!) fields on the object itself.

By way of contrast, in the functional-programming world our building blocks are _functions_. Functions are not bound to an object - all they have to work with is the parameters they were explicitly supplied. This propety makes functions easier to reason about than methods (in particular, it makes modifying them easier!)

Note for C# programmers: an unbound function corresponds to a `static` method. 

In the object-oriented world, we frequently encounter methods that
* Accept objects (not just primitive values) as parameters
* Return an object (rather than a primitive value)

There is a symmetry in the functional-programming world - we have functions that
* Accept other functions (not just primitive values) as parameters
* Return a function (rather than a primitive value)

Such functions are known as Higher order Functions (HoFs). HoFs are the primary means for code reuse in functional programming - like the strategy pattern on steroids.

For the C# programmers out there, a common example of a HoF can be seen in LINQ:

```csharp
var numbers = Enumerable.Range(1, 10);

// Create a function with the signature string → bool
Func<int, bool> isEven = theNumber => theNumber % 2 == 0;

// Invoke the LINQ Where method, passing the function in as an argument (the predicate)
var evenNumbers = numbers.Where(isEven);
```

By allowing the caller to pass a predicate function (e.g. 'isEven') to the `Where` method, the designers of LINQ have enabled significant extensibility (rather than trying to anticipiate the filtering operations that might be needed).

At this point you're probably thinking that dealing solely in terms of primitive data types and functions to operate on them would be hugely limiting - and indeed it would be! FP does use composite types, the distinction is that the types don't have _behaviour_ associated with them - they're *just data*. See the section on Algebraic Data Types. 

So to summarise & grossly over-simplify:
* Functions are first-class things that we pass around like any other kind of parameter
* We don't mush-together state and behavior
* We build the overall behaviour by combining functions

### What are some of the core concepts from FP?

#### Pure functions
A function call is pure if you can replace the function call with the pre-computed result *without affecting behaviour*. For a function to be pure, it must adhere to the following:
* It's return value must be entirely derived from its input parameters
* It must not mutate (modify) any of its inputs
* It must not trigger any side-effects (such as writing to disk, network calls etc)

You might hear a similar term "referential transparency" - which is a little bit weaker as it allows _insignificant_ side-effects (such as writing to the console or logging). 

For example, a function that computes the MD5 hash of a given input string is pure as you can replace the function call with the pre-computed MD5 hash. As a counter-example, a function that returns the current time (e.g. `DateTime.Now` is *not* pure)


#### Isolation of side-effects
blah blah blah

How does this relate to "pure" functions?


What is it, in a nutshell?
pure functions
no side effects
separating data from behaviour
leaning heavily on the compiler (type system) to help prove the correctness of your program.

What do you get from adopting FP-style code? 
* Code that is easier to reason about
* Code that is easier to parallelize (scale)
* Code that is easier to test

One thing that's worth noting is that it's not a case of "all or nothing". You'll often find that certain parts of a program will lend themselves more to FP than others. Quite a common and reasonable approach is that of the "functional core + imperative shell" as coined in the blog post [clean and green](http://drocco007.github.io/2015_pytn/clean_and_green.html)

The idea is that the *majority* of the application (_especially_ the complex business logic - the "core") is written in a functional style while the edges / interface to the outside world ("ports") are written in an object-oriented or imperative style - the idea is staying in functional land as long as possible.

I've also previously [blogged](2019/10/17/writing-testable-software/) about a similar idea which I coined the "execution plan" pattern:

> Essentially, the idea is to split figuring out what needs to be done from actually doing it. This pattern works particularly well when the “figuring-out” bit is complex / full of business logic.

Introduction (not "all-or-nothing", functional core with imperative shell. Yes it's rooted in maths)

Topics:

* Introduction (not "all-or-nothing", functional core with imperative shell. Yes it's rooted in maths)
* Functions as data
* Pure functions
*   Referential Transparency, Side effect, 
* Opinion: Recoverable (unchecked) exceptions for (flow control) are evil. Out of memory, out of disk space, assertion exception
* The problem with null return values. Actually it's just a special case of the general problem of code might not handle all possible return values.
* Inverting control to get compile-time safety (basically, you can't get at the result unless you promise to deal with or at least acknowledge the edge cases) 
* Option aka Maybe
* Either - and brief segue into union types vs product types (algebraic data types. This post has more info: https://jrsinclair.com/articles/2019/algebraic-data-types-what-i-wish-someone-had-explained-about-functional-programming/). Option and Either are both examples of ADTs. Useful in business domain too - preventing invalid states.
- https://jrsinclair.com/articles/2019/algebraic-structures-what-i-wish-someone-had-explained-about-functional-programming/

Reference https://github.com/louthy/language-ext 

> This library uses and abuses the features of C# to provide a functional-programming 'base class library' that, if you squint, can look like extensions to the language itself. The desire here is to make programming in C# much more reliable and to make the engineer's inertia flow in the direction of declarative and functional code rather than imperative

And also https://github.com/emmanueltouzery/prelude-ts 

> prelude-ts (previously prelude.ts) is a TypeScript library which aims to make functional programming concepts accessible and productive in TypeScript. It provides persistent immutable collections (Vector, Set, Map, Stream), and constructs such as Option, Either, Predicate and Future. 