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

An introduction to functional-programming - the low-hanging fruit 🍒🍍🍏

## Introduction

This post is intended to be a gentle introduction to functional programming (FP) for C# developers working in the object-oriented (OO) paradigm - no prior knowledge of functional programming is assumed.

I hope that after reading it, you'll have some useful tools & techniques 🔨 in your belt that you can apply to day-to-day software development 👷. It's worth remembering that FP isn't a case of all-or-nothing - you can use the ideas in isolated areas of the codebase (where it makes sense).

Although functional programming is rooted in mathematics 🧮, I've tried to keep the post practical - if you're interested in the _theory_, there are plenty of other posts out there.

Finally, I should mention that I'm entirely self-taught (no formal training in functinal programming) and thus I'm very much still learning myself!

## What are the key ideas from functional programming?

### Software is built by composing & reusing functions

Admittedly a bit of a cop-out, but I'll start by contrasting functional programming (FP) with object-oriented (OO) programming.

In the object-oriented software world, our basic building-blocks 🧱 are _classes_. We use classes to create _object instances_ (or just "objects").

An object instance combines behaviour _and_ state (data) _together_ (encapsulation). Objects expose methods which (when called)  
* Modify the object's internal state ("increase line-item quantity")
* Perform computations ("calculate order shipping cost")
* Trigger side-effects ("fire the missiles!" 🚀) 

The program as a whole can be viewed as an object [graph](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)) - there is a root object in the program's entrypoint (composition-root)
* The root object has references to other objects (it's collaborators)
  * Each of those objects have references to _other_ objects (_their_ collaborators)
    * ...and so on and so forth 

A request/message comes in from the outside world (button click, HTTP request, stdin...), the root object calls methods[^1] on _it's_ objects (which in turn call methods) - and thus the program springs to life. 

By way of contrast, functional-programming languages don't use
* Classes
* Objects
* Methods

Primarily, they deal with 
1) Lumps of data 
2) Functions

(...and some other things like ADTs & typeclasses, which I'll ignore for now)

Avoiding (for the time being) a more nuanced discussion of what a [function](https://en.wikipedia.org/wiki/Function_(mathematics)) is and is not, just think of a function as an _unbound_ method (i.e. a `static` method). So unlike a method (which is bound to a particular object), a function just kind of "floats around". Because the function isn't tied to an object, it can only utilise the parameters that were passed in when it was called.

In the functional programming paradigm, the program as a whole can be viewed as a [computation](https://en.wikipedia.org/wiki/Model_of_computation):
* A request to perform computation comes in from the outside world
* The request is represented as a lump of data / values
* The data is passed through a _pipeline_ of functions
  * The output from an upstream function is used as the input to downstream functions
  * The data may change _shape_ as it passes through the pipeline
  * Once a result has been produced (by a function in the pipeline) that result is never modified (instead, a _new_ result is computed based on the inputs)   
* Finally, the result of the computation pops out at the other end 🏭

Code written in a functional style often treats functions as data too - think along the lines of [reverse-polish-notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation) (RPN) where `calculationToPerform = [2, 4, 8, sum, mult];` represents `2 * (4 + 8)` - the functions `sum` and `mult` have been included alongside operands (numbers).

[^1]: The original proponents of object-oriented programming [didn't really intend for it to work like this](http://lists.squeakfoundation.org/pipermail/squeak-dev/1998-October/017019.html) - it was supposed to be about actors sending messages - closer to how actor-based models like [Akka.NET](https://github.com/akkadotnet) work.

##### A quick note on encapsulation
In the world of object-oriented programming, the principal of _encapsulation_ warns us against creating types that are "just data" (i.e. don't have behaviour) - see the [AnemicDomainModel](https://martinfowler.com/bliki/AnemicDomainModel.html).

Encapsulation is a core pillar of object-oriented programming - according to OO best-practice, a well-designed object
* Offers a minimal public interface (API)
* Hides implementation details
* Is responsible for protecting its own internal state & invariants 

Encapsulation is primarily intended to help with
* Enabling code reuse
* Reducing cognitive load for developers
* Ensuring correct behaviour / reducing bugs

In functional programming, the same outcomes are achieved through _different_ means - primarily: 
* Algebraic Data Types ("leaning on the type system")
* Immutability
* Pure-functions / isolation of side effects
* Typeclasses

some of these ideas are explored in the remainder of the post.

### Data is immutable

Immutability is the idea that once a value is created, _that_ (particular) value never changes; it is only possible to create _new_ values.

As an example, imagine some software that deals with lists of people (perhaps a 'friends list'):

* Under a traditional (_mutable_) design, "adding a friend" would change the data structure **in-place** 
  * Any code that has a reference to the friend list would _automatically_ "see" (observe) the updated list
* Under an _immutable_ design, "adding a friend" would create a **new** data structure which is a shallow-copy of the previous friend list **with the new friend added at the end**
  * Any code that has a reference to the original friend list (as it was prior to adding the friend) would continue to see the same list of friends
  * Only code that has deliberately been passed the new friends list will observe the changes 

When programming in an object-oriented (OO) or mixed paradigm (part OO, part FP) style, it is possible to make a class immutable although it takes some care/rigour to do so:
* The class should not expose any property setters or public fields
* Any mutating operations (methods) should return a new instance ([copy constructors](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/how-to-write-a-copy-constructor) is the come in handy)
* Take not to reuse collections when mutating
* Ideally, all dependencies of the class (constructor arguments) should be immutable also (transitively) 

C# [records](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records) make writing immutable types substantially easier

### Almost all functions are pure
A function is "pure"[^2] if you can replace all calls to that function with pre-computed results (without affecting the program behaviour).

For a function to be pure, it must adhere to the following:
* The return value must depend *solely* on the function inputs
* The function must not mutate (modify) any of its input parameters
* The function must not trigger any side-effects (such as writing to disk, network calls etc)
* It may only call other pure functions

[^2]: You might encounter a similar term "referential transparency" - which is essentially the same thing but a weaker guarantee as it allows _insignificant_ side-effects (such as writing to the console or logging).

Some examples:
* A function `sha1sum` that computes the SHA1 hash of a given input string is pure as you can replace the function call with the pre-computed hash for the string
* `DateTime.Now` (which returns the current system time in C#) and `Guid.NewGuid` (generates a new GUID) are *not* pure because each time you call them you get a different result.
* A function `createPerson` that takes a couple of strings (`firstName` and `lastName`) as input parameters and combines them into a data-structure including a GUID `personId` (generated with `Guid.NewGuid`) is *not* pure because `createPerson` _calls_ an impure function.
* A function `addToCart` which takes a shopping-cart data structure `cart`, a `productId` and `quantity` and updates the cart in-place is *not* pure, because it mutates the `cart` parameter.
  * If instead `addToCart` returned a *new* cart (rather than updating in-place) then it would be pure.
* A function `calculateRiskProfile` which transforms its input parameters, makes a HTTP `GET` web-service call and massages the response from the web-service into a return value is *not* pure because of the web-service call:
  * the code executing in the external web-service is entirely out of our control and thus must be assumed to be impure
  * the web-service call goes over the network. The network could be down, the request could time-out etc

In "proper" functional programming languages (like Haskell), all functions are pure by default (i.e. unless explicitly stated otherwise).

#### Advantages of pure functions
* They're super easy to test because you can treat them as a black-box
    
    "Does the function do what it says 🏷️ on the tin 🥫?" - given these inputs, does it produce the correct output? Also, [seams](https://www.informit.com/articles/article.aspx?p=359417&seqNum=3) are obvious

* They make code easy to reason about. The signature of the function (inputs & output types) largely describes what the function does - see also "leaning on the type system"

     Have you ever worked on a codebase with a method innocuously named `GetOrderDetails`, only to discover that function sometimes deletes data?

* They're easy to debug - simply examine the intermediate results as the data flows through the pipeline

* They make parallelization easy

     If you have a large array of items to be processed and a function `processItem` taking a single item as a parameter, it's trivial to parallelize the work across multiple threads/processes

* They're easily reusable

     Since the function is guaranteed not to have any unwanted side-effects (by definition!) you can reference it anywhere you need it.
* The results from a pure-function can be cached indefinitely! There's no need to worry about the results becoming stale.

See this [post](https://medium.com/@juntomioka/why-pure-functions-are-so-good-7f7759021c35) for more on the benefits of pure functions.

Note for C# programmers: The `[Pure]` attribute can be used to indicate _intent_ to other developers (unfortunately, the compiler doesn't enforce anything).

### Isolation of side-effects
So you're following along, you've probably concluded
* Pure functions = good
* Side effects = bad

But as it turns out, side-effects (at least I/O) are a necessary evil. Real programs need to do at least one of the following to be useful:
* Read input from disk / network / keyboard
* Write output to the disk / screen
* Communicate with another program or system (network, pipe etc)

Again, in "proper" functional programming languages, [the compiler prevents us from performing I/O unless we're in a special context](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/IO) (IO Monad) - similar to how the `await` keyword can't be used unless you're already in an `async` method in C#. 

Unlike Haskell, in C# the compiler can't prevent us from performing I/O in arbitrary code, so the best we can hope for is a "Gentleman's Agreement" (with the other developers on our team) around when and where to perform I/O.

We want to keep as much of our codebase functionally "pure" as possible (for the reasons/benefits listed in the previous section). The basic strategy is to *push side-effects to the very edges* (of the program). 

There's a great blog-post [clean and green](http://drocco007.github.io/2015_pytn/clean_and_green.html) which coins the term "functional core + imperative shell". The basic ideas being
* Separate policies from mechanisms and pass simple data structures between the two
* The imperitive-shell is procedural "glue" code that offers an OO interface & manages dependencies (mechanisms)
* The functional core (expressed in pure functions) implements all the decisions (policies)
* Never mix decisions and dependencies

The idea is that the *majority* of the application (_especially_ the complex business logic - the "core") is written in a functional style while the edges / interface to the outside world (the "ports") are written in an object-oriented or imperative style - keeping us in functional land as much as possible.

I've also previously [blogged](2019/10/17/writing-testable-software/) about a similar idea which I call the "execution plan pattern" - the idea is to split figuring out "what needs to be done" from actually doing it - the code to _generate_ the "plan" (from data) is functionally pure (and possibly complex), but the _execution_ of the plan is impure (but simple).

At any rate, I'd suggest structuring your code so that the bit that actually performs the I/O or side-effects has very low [cyclomatic complexity](https://www.geeksforgeeks.org/cyclomatic-complexity/) - in other words, avoid branching (`if`/`else`) and looping in that code.

For some other ideas, see [the Effect monad (Eff & Aff) in the language-ext library](https://github.com/louthy/language-ext/wiki/How-to-deal-with-side-effects#aff-and-eff-monad).

### Idempotency

An action is said to be idempotent if performing the same action **multiple times** yields the same outcome as performing the action **once**.

As an example, consider a "withdraw money" operation acting on a bank account balance:

```
{
  balance: "10239.45",
  asOfDate: "2022-19-08"
}
```

By default, a "WITHDRAW $100" action is **not** idempotent, because performing the action once yields a balance of $10,139.45 whereas performing the action three times yields a balance of $9,939.45.

_One possible way_ to support idempotent actions acting on an account balance is as follows:

```javascript
{
  balance: "10239.45",
  asOfDate: "2022-19-08",
  version: "747"
}
```  

When performing the action we include the version number alongside (or as part of) the action. The code handling the action knows to check the current version number on the balance; if the version number is not the expected version then the action-handler ignores/discards the action. If the version **is** as-expected then the balance is updated and the version is incremented.

```javascript
{
  balance: "10139.45",
  asOfDate: "2022-19-08",
  version: "748"
}
```

Note that as far as _functions_ go, **all "pure" functions are idempotent** (because pure functions _by definition_ do not produce side-effects & do not mutate their inputs) but **not all idempotent functions are pure** - idempotent functions can (and often do) cause side-effects.

As a _consumer_, idempotent APIs and functions & APIs are typically safer (and therefore easier) to use. Consider the difference between consuming a `createDirectory()` function and a similar `ensureDirectoryExists()` function: 
* When consuming `createDirectory()`, your application code is forced to deal with the possibility that the directory might _already_ exist
* When consuming `ensureDirectoryExists()`, your application is shielded from that particular possibility (_other_ I/O problems can still occur however)

Idempotency is usually more applicable to API design & systems architecture than it is to general programming, but it's a useful concept to know about. 

### Leaning on the type system

In my opinion, this is probably the most powerful and easily-adopted aspect of functional programming - the lowest hanging fruit of all 🍉.

As a object-oriented programmer, you might be used to a workflow such as this:
1. Write some code
2. Fix any errors reported by the compiler
3. Run your program & interact with it manually (so-called "exploratory" testing)
   1. If that yields any issues, then go back to step #1 
4. Write some automated unit tests for your code
5. Rinse and repeat

If you follow the [TDD](https://martinfowler.com/bliki/TestDrivenDevelopment.html) (test-driven-development) methodology, your approach would look a little different: you'd write the tests earlier and interleaved with writing the production code.

, however that difference isn't important for the point I'm about to make:

As a typical object-oriented developer, a successful compilation (step #2 above) **doesn't give you very high confidence** that your program works correctly, or that it does so for all edge-cases and inputs. You need to write a bunch of tests (and have those tests pass) before having any semblance of confidence. 

The eutopia that functional-programmers strive for is "If my program compiles without errors, it's probably correct" - I refer to this as "leaning on the type system". 

Can't forget to write the tests.

IsShipped


talk about the order example e.g. ShippedOrder, ConfirmedOrder c.f. boolean props

typescript e.g. 

```typescript
type Country = "England" | "USA" | "France";
const myCountry: Country = "New Zealand"; // compile error
```



Avoiding [primitive obsession](https://wiki.c2.com/?PrimitiveObsession) taken further - algebraic data types

TODO: leaning heavily on the compiler (type system) to help prove the correctness of your program.

* Opinion: Recoverable (unchecked) exceptions for (flow control) are evil. Out of memory, out of disk space, assertion exception
* The problem with null return values. Actually it's just a special case of the general problem of code might not handle all possible return values.
* Inverting control to get compile-time safety (basically, you can't get at the result unless you promise to deal with or at least acknowledge the edge cases)
* Option aka Maybe
* Either - and brief segue into union types vs product types (algebraic data types. This post has more info: https://jrsinclair.com/articles/2019/algebraic-data-types-what-i-wish-someone-had-explained-about-functional-programming/). Option and Either are both examples of ADTs. Useful in business domain too - preventing invalid states.
- https://jrsinclair.com/articles/2019/algebraic-structures-what-i-wish-someone-had-explained-about-functional-programming/ railway-oriented-programming: https://fsharpforfunandprofit.com/rop/#slides



### Higher Ordered Functions

TBD: IS THIS SECTION WORTHWHILE? Yes, HoF are a core part of FP. On the other hand, strategy pattern (interface) achieves a similar thing. Use HoF where it makes sense 🤷‍♂️.

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





## Further reading

Reference https://github.com/louthy/language-ext 

> This library uses and abuses the features of C# to provide a functional-programming 'base class library' that, if you squint, can look like extensions to the language itself. The desire here is to make programming in C# much more reliable and to make the engineer's inertia flow in the direction of declarative and functional code rather than imperative

And also https://github.com/emmanueltouzery/prelude-ts 

> prelude-ts (previously prelude.ts) is a TypeScript library which aims to make functional programming concepts accessible and productive in TypeScript. It provides persistent immutable collections (Vector, Set, Map, Stream), and constructs such as Option, Either, Predicate and Future. 

Other articles / posts: 

https://www.manning.com/books/functional-programming-in-c-sharp (Functional Programming in C#: How to write better C# code)

https://github.com/hemanth/functional-programming-jargon

https://www.yld.io/blog/the-not-so-scary-guide-to-functional-programming/
https://cscalfani.medium.com/why-is-learning-functional-programming-so-damned-hard-bfd00202a7d1
https://mikhail.io/2018/07/monads-explained-in-csharp-again/
https://buttondown.email/hillelwayne/archive/making-illegal-states-unrepresentable/
https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/