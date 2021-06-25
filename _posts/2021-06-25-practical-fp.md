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

First post in a series covering practical functional-programming ideas for everyday LoB application developers

I've recently read the book [Functional programming in C#](https://www.manning.com/books/functional-programming-in-c-sharp) by Enrico Buonanno - I highly recommend reading it. This post (series?) will concentrate the ideas from the book that _I_ found most valuable, as well as some ideas from other sources.

## Introduction

You've probably heard of functional programming (FP) before, but perhaps you've been put off by complicated geeky terms like "Lambda Calculus", "Algebraic Data Type" or the dreaded m-word (...Monad 😱). 

<figure class="wp-block-image size-large"><img src="/images/posts/practical-fp-part-1/monad-monad-monad.png"/></figure>

Yes, the ideas are rooted in mathematics however there's still really valuable stuff you can draw on without paying too much attention to the theory. I'll try and present what I think are the most useful ideas.

What is FP, in a nutshell? 

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
* Opinion: Recoverable (unchecked) exceptions are evil. Out of memory, out of disk space, assertion exception
* The problem with null return values. Actually it's just a special case of the general problem of code might not handle all possible return values.
* Inverting control to get compile-time safety (basically, you can't get at the result unless you promise to deal with or at least acknowledge the edge cases) 
* Option aka Maybe
* Either - and brief segue into union types vs product types 

Reference https://github.com/louthy/language-ext 

> This library uses and abuses the features of C# to provide a functional-programming 'base class library' that, if you squint, can look like extensions to the language itself. The desire here is to make programming in C# much more reliable and to make the engineer's inertia flow in the direction of declarative and functional code rather than imperative

And also https://github.com/emmanueltouzery/prelude-ts 

> prelude-ts (previously prelude.ts) is a TypeScript library which aims to make functional programming concepts accessible and productive in TypeScript. It provides persistent immutable collections (Vector, Set, Map, Stream), and constructs such as Option, Either, Predicate and Future. 