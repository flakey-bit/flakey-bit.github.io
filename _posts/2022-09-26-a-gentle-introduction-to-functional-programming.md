---
title: 'A gentle introduction to functional programming'
date: 2022-09-26T10:35:00+12:00
author: eddiewould
layout: post
permalink: /2022/09/26/a-gentle-introduction-to-functional-programming/
spay_email:
  - ""
categories:
  - Uncategorized
---

A gentle introduction to functional-programming - the "low-hanging fruit" 🍒🍍🍏

## Introduction

This post is intended to provide a _gentle_ introduction to the world of functional programming (FP) for C# developers working in the object-oriented (OO) paradigm - no prior knowledge of functional programming is assumed.

My goal is to convince you that FP has some cool things to offer - hopefully whetting your appetite for further learning (I've tried to link to resources that I found easy to digest). You might also add some tools 🔨 & techniques to your belt that you can apply to day-to-day software development 👷 

Although functional programming is rooted in mathematics 🧮, I've tried to keep the post practical & approachable - if you're interested in the _theory_, there are plenty of other posts out there.

Finally, I should mention that I have no formal qualifications in FP so take what you read with a grain of salt 🧂

## What are the key ideas from functional programming?

### Software is built by composing & reusing functions

Admittedly a bit of a cop-out, but I'll start by contrasting functional programming (FP) with object-oriented (OO) programming.

In the object-oriented software world, our basic building-blocks 🧱 are _classes_. We use classes to create _object instances_ (or just "objects").

An object instance combines behaviour _and_ state (data) _together_ (encapsulation). Objects expose methods which (when called)  
* Modify the object's internal state ("increase order line-item quantity")
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

_Primarily_, they deal with 
1) Lumps of data 
2) "Pure functions" (calculations)
3) "Side effects" (actions) 

Avoiding a more nuanced discussion of what a [function](https://en.wikipedia.org/wiki/Function_(mathematics)) is and is not, just think of a function as an _unbound_ method (a `static` method).
* A **method** is bound to a particular object (instance)
* A **function** just kind of "floats around" (it isn't tied to any particular object).
  * As a consequence, a function **can only utilise the parameters that were explicitly passed in** when it was called.

If that distinction doesn't make sense, don't worry about it for now - just use the words "method" and "function" interchangeably.

In the functional programming paradigm, the program as a whole can be viewed as a [computation](https://en.wikipedia.org/wiki/Model_of_computation):
* A request to perform computation comes in from the outside world
* The request is represented as a lump of data / values
* The data is passed through a _pipeline_ of functions
  * The output from an upstream function is used as the input to downstream functions
  * The data may change _shape_ as it passes through the pipeline
  * Once a result has been produced (by a function in the pipeline) that result is never modified (instead, a _new_ result is computed based on the inputs)   
* Finally, the result of the computation pops out at the other end 🏭
  * Any side-effects that the computation needs (saving to disk, responding to the request) are typically triggered at the _edges_ of the program (right at the start, or right at the end)  

Code written in a functional style often treats functions as data too - think along the lines of [reverse-polish-notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation) (RPN) where 

```calculationToPerform = [2, 4, 8, sum, mult];```

represents `2 * (4 + 8)` - the functions `sum` and `mult` have been included _alongside_ the operands (numbers).

Things to note:
* Languages that allow functions to be passed around like this (treated as data) are said to have "first-class functions". A language with first-class functions is basically a pre-requisite for functional programming. Fortunately, C# has first-class functions with `delegate` / `func` / `action` etc
* A function that _takes_ other functions as data or _returns_ a function as its result is known as a Higher-order Function (HoF)

[^1]: The original proponents of object-oriented programming [didn't really intend for it to work like this](http://lists.squeakfoundation.org/pipermail/squeak-dev/1998-October/017019.html) - it was supposed to be about actors sending messages - closer to how actor-based models like [Akka.NET](https://github.com/akkadotnet) work.

#### A quick note on encapsulation
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
* Any mutating operations (methods) should return a **new** instance ([copy constructors](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/how-to-write-a-copy-constructor) come in handy)
* Take care not to reuse collections when mutating
* Ideally, all dependencies of the class (constructor arguments) should be immutable also (transitively) 

C# [records](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records) make writing immutable types substantially easier

### Almost all functions are pure
A function is said to be "pure"[^2] if you can replace all calls to that function with pre-computed results (without affecting the program behaviour). 

A good litmus test is "If it matters **how many times** we call the function (or **whether we call it at all**), it's not pure".

By that logic, a `SendEmail` function is clearly not pure:
* If we don't call it at all, the customer doesn't receive an email
* If we call it 100 times, the customer gets 100 emails (oops 🤭)

For a function to be pure, it must adhere to the following:
* The return value must depend *solely* on the function inputs
* The function must not mutate (modify) any of its input parameters
* The function must not trigger any side-effects (such as writing to disk, network calls, updating global state etc)
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

It's worth emphasising that impurity spreads like a zombie 🧟 virus - imagine the call stack:

```bash
SendEmail()
GenerateAndSendConfirmationEmail()
ProcessCustomer()
ProcessCustomers()
Main()
```

* Because `SendEmail` is impure and `GenerateAndSendConfirmationEmail` calls `SendEmail`, `GenerateAndSendConfirmationEmail` is impure
* Because `GenerateAndSendConfirmationEmail` is impure and `ProcessCustomer()` calls `GenerateAndSendConfirmationEmail`, `ProcessCustomer()` is impure
* ...
* therefore the whole call stack (right the way to `Main()`) is impure

In functional programming languages (like Haskell), all functions are pure by default (i.e. unless explicitly stated otherwise).

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

Functional programmers aim to write as much of their application in terms of pure functions as possible - keeping actions simple & **at the edges** (bottom of the call stack).

### Isolation of side-effects
So if you're following along, you've probably concluded
* Pure functions = good
* Side effects = bad

But as it turns out, side-effects (at least I/O) are a necessary evil. Real programs need to do at least one of the following to be useful:
* Read input from disk / network / keyboard
* Write output to the disk / screen
* Communicate with another program or system (network, pipe etc)

Again, in functional programming languages, [the compiler prevents us from performing I/O unless we're in a special context](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/IO) (IO Monad) - similar to how the `await` keyword can't be used unless you're already in an `async` method in C#. 

Unlike Haskell, in C# the compiler can't prevent us from performing I/O in arbitrary code, so the best we can hope for is a "Gentleman's Agreement" (with the other developers on our team) around when and where to perform I/O.

We want to keep as much of our codebase functionally "pure" as possible (for the reasons/benefits listed in the previous section). The basic strategy is to *push side-effects to the very edges* (of the program). 

There's a great blog-post [clean and green](http://drocco007.github.io/2015_pytn/clean_and_green.html) which coins the term "functional core + imperative shell". The basic ideas being
* Separate policies from mechanisms and pass simple data structures between the two
* The imperitive-shell is procedural "glue" code that offers an OO interface & manages dependencies (mechanisms)
* The functional core (expressed in pure functions) implements all the decisions (policies)
* Never mix decisions and dependencies

The idea is that the *majority* of the application (_especially_ the complex business logic - the "core") is written in a functional style while the edges / interface to the outside world (the "ports") are written in an object-oriented or imperative style - keeping us in functional land as much as possible.

I've also previously [blogged](/2019/10/17/writing-testable-software/) about a similar idea which I call the "execution plan pattern" - the idea is to split figuring out "what needs to be done" from actually doing it - the code to _generate_ the "plan" (from data) is functionally pure (and possibly complex), but the _execution_ of the plan is impure (but simple). 

Another way to think about it is that your "functional core" produces a list of action _descriptions_ (side effects) to execute. Those actions are finally executed near the entrypoint to your program. The code that finally executes the side-effects or I/O should have very low [cyclomatic complexity](https://www.geeksforgeeks.org/cyclomatic-complexity/) - in other words, avoid branching (`if`/`else`) and looping in that code.

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

In my opinion, this is probably the most powerful and easily-adopted aspect of functional programming - the lowest hanging fruit of all 🍉

As a object-oriented programmer, you might be used to a workflow such as this:
1. Write some code
2. Fix any errors reported by the compiler
3. Run your program & interact with it manually (so-called "exploratory" testing). If that yields any issues, then go back to step #1 
4. Write some automated unit tests for your code
5. Rinse and repeat 🚿

If you follow the [TDD](https://martinfowler.com/bliki/TestDrivenDevelopment.html) (test-driven-development) methodology, your approach would look a _little_ different: you'd write the tests earlier and you'd write them as you write the production code.

Many proponents of TDD claim it is superior because it yields better (software) designs, however I feel a more _compelling_ reason to adopt TDD is that it offers faster feedback.  

Regardless of whether you follow TDD or not, as a typical object-oriented software developer, a successful compilation (step #2 above) **doesn't give you very high confidence** that your program works correctly, or that it does so for all inputs & edge-cases. You need to write a bunch of tests (and have those tests pass) before having any semblance of confidence. 

The utopia that functional-programmers strive for is "If my program compiles, it's probably correct" - I refer to this as "leaning on the type system". Does that mean functional programmers don't write tests? Of course not. But I'd argue they write _fewer_ tests - as an object-oriented programmer many of your tests will fall into the category of 
* Ensure all edge-cases are handled &
* Ensure invalid states are prevented (e.g. "an order can't be out for delivery if it's waiting on an item to arrive in the warehouse")

The idea is that we get the compiler to do the work for us (ensuring edge cases are handled & invalid states are prevented) so that we don't have to do it in our application code. If we don't have to write code to prevent these problems, then it's less important to write _tests_ showing that the problems have been prevented.

As a massive simplification, we want the compiler to prevent our program from compiling if 
* It has failed to deal with an edge-case
* It is creating an invalid state

#### Prefer parsing over validating (make illegal states unrepresentable)

One key ways functional programmers lean on the type system is by preferring parsing over validation. What is the difference between the two?
* When we _validate_ a value, **all we do** is check that it meets our expectations (for example: value must be non-negative; value must be less than 256)
  * If the validation fails then we typically halt further processing (return an error, throw an exception etc)
  * If the validation succeeds then the value is permitted through and processing continues
* When we _parse_ a value **we make it fit into a more rigid structure** - such a structure will often have multiple parts (with different shapes), repeating groups etc.

As an example, consider an Australian bank account number. A validation-based approach would probably represent an account number as a `string` - and add a method to check that an account number is valid:
```csharp
public static void CheckValidAccountNumber(string accountNumber) {
    if (... || ... || .../* various checks */) {
        throw new ArgumentException("Invalid account number", nameof(accountNumber));
    }
}
```

The main problems with this approach are:
* It's not always clear whose responsibility it is to validate a value
  * Where should `CheckValidAccountNumber` be called from?
* It's not always clear whether a value has already been validated
  * Assume we're deep in a business-service somewhere and we receive a string containing an account number, should we validate it before making REST API call to a 3rd party webservice, or should we assume that has already been done elsewhere?

At best, this leads to "shotgun parsing":

> Shotgun parsing is a programming antipattern whereby parsing and input-validating code is mixed with and spread across processing code—throwing a cloud of checks at the input, and hoping, without any systematic justification, that one or another would catch all the “bad” cases.

At worst, we might forget to perform the validation **at all** (or remove it by mistake).

What would a parsing-based approach look like? Well, we know that Australian bank account numbers are composed of a "BSB" (Bank State Branch number) & a six digit account number ("namespaced" to the BSB).

So you could model it like this:

```csharp
public record AustralianBankAccountNumber(AustralianBsb Bsb, string LocalAccountNumber);
public record AustralianBsb(string Value);
```

Digging a bit further still, a BSB `XXY-ZZZ` is comprised of three parts:
* `XX`: The parent financial institution (e.g. "03" for Westpac Banking Corporation
* `Y`:  The state where the branch is located (e.g. "3" for Victoria)
* `ZZZ`: The branch location (e.g. "547" indicates the Westpac branch at 360 Collins Street) 

So our implementation might look at follows:

```csharp
public record AustralianBankAccountNumber(AustralianBsb Bsb, string LocalAccountNumber);

// Not shown here, but the constructor for AustralianBsb would have additional validation 
// e.g. checking the state number is valid
public record AustralianBsb(byte FinancialInstitution, byte State, short BranchLocation)
{
    private static readonly Regex BsbRegex = 
        new Regex("^(?<FinancialInstitution>[0-9]{2})(?<state>[0-9])[-](?<branchLocation>[0-9]{3})$");

    public static bool TryParse(string value, out AustralianBsb? parsed)
    {
        var match = BsbRegex.Match(value);
        if (match.Success)
        {
            // These three statements should be safe as we've matched the regex
            var financialInstitution = byte.Parse(match.Groups["FinancialInstitution"].Value);
            var state = byte.Parse(match.Groups["state"].Value);
            var branchLocation = short.Parse(match.Groups["branchLocation"].Value);

            try
            {
                // The constructor might perform further checking - at time of writing there are only 6 states
                // in Australia, so presumably there are some invalid state values (0, 1, 8, 9)?
                parsed = new AustralianBsb(financialInstitution, state, branchLocation);
                return true;
            }
            catch (ArgumentException)
            {
                parsed = null;
                return false;
            }
        }

        parsed = null;
        return false;
    }
}

public class CallingCode
{
    public void UsingBsb()
    {
        var success = AustralianBsb.TryParse("033-547", out var bsb);
        Console.Out.WriteLine($"The BSB has state: {bsb!.State}");
    }
}
```

The main benefit of this is that no-matter where we are in the codebase, if we have a `AustralianBsb` instance (rather than a `string`) then we **know** that it is valid and conforms to the domain rules. We should therefore try to parse data as soon as it enters our application (at the boundary) - adhering to the [fail-fast principal](https://www.martinfowler.com/ieeeSoftware/failFast.pdf).

Alexis King has written an excellent [blog-post](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) on this subject, which I highly recommend reading (however her post uses Haskell in the examples, which may dissuade some readers).


#### The problem with (unchecked) exceptions

Most modern mainstream programming languages (C#, Python, Java, C++, JavaScript, PHP) use [exceptions](https://en.wikipedia.org/wiki/Exception_handling) to deal with anomalous or exceptional conditions. 

The basic idea is that if something unexpected happens (outside of the "normal" flow) then our code can throw an exception.

When an exception is thrown, the code that would normally follow is not executed - instead, the call stack is unwound to the nearest frame (call) that explicitly agrees to handle exceptions of that type. If the exception is handled (by code lower down in the call stack) then the program can continue (from the point at which the exception was handled) - if not, the program crashes and terminates.

An important consideration around exceptions is whether the program can reasonably recover from the exceptional circumstance of not:
* If the computer has run out of memory or we've hit a bug in the operating system then we probably can't recover from that
* If we made a request to a 3rd party API but the network request timed out, that *is* probably a situation we can recover from
* If one of our invariants (e.g. the current method should never be called with an empty array) has been violated then we _might_ be able to recover from that, but applying the [fail-fast principal](https://www.martinfowler.com/ieeeSoftware/failFast.pdf) would probably be more advisable

Depending on the programming language, exceptions can be either "checked" or "unchecked":
* With _checked_ exceptions, the fact that a method can throw a particular exception is part of the signature of the method. To call a method that might throw an exception, you have to promise to handle those exceptions to be allowed to call the method (or you have to pass those exceptions on in *your* signature)
* With _unchecked_ exceptions, the compiler doesn't know what exceptions (if any) a method might throw. At best, the method documentation will list the exceptions. But there is no guarantee that the list of exceptions in the documentation is accurate or up-to-date

Checked exceptions sound like a good idea in theory, however in practice (at least in Java) they entail too much ceremony so developers end up bypassing (skipping) the checking.

The problem with unchecked exceptions is that the signature of the method is *dishonest* - consider the following function:

```csharp
// Determines the largest number in the sequence
public int FindMaximum(IEnumerable<int> numbers) {
  if (numbers.Count == 0) {
    throw new ArgumentException();
  }

  return numbers.Max();
}
```

When the function is in a compiled library, the programmer only sees the signature - not the implementation:

```csharp
public int FindMaximum(IEnumerable<int> numbers);
```

As the *caller* of the library function, I know that if I provide it with a list of numbers, it will return me a single number (the maximum). But I have no way of knowing that it will explode in my face if the list of numbers I provide is empty. More importantly, my code isn't forced to deal with that possibility.

What we want is a _richer_ function signature in terms of the result:
* The result might be a number
* **Or**, the result might be that we *can't* calculate the maximum (because it isn't possible)

As a caller of this function, I want to be forced to handle both of these possibilities.

#### The problem with null

Similarly, most object-oriented programming languages have the concept of `null` - a special value which represents the _absence_ of a value / object. 

Typically, `null` doesn't have any properties and you can't call any methods on it. If a variable happens to contain `null` & your code performs property access or invokes a method call on the `null` value, an exception will be thrown (e.g. `NullReferenceException`).

The problem is that *the onus is on the programmer* to remember that a value "might" be null & to guard against it[^3].

[^3]: It's worth noting that the situation has become better recently in C# 8.0, with the introduction of [nullable reference types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references).

#### Boolean fields are often a code smell

Imagine a fictional `Order` class in an object-oriented programming language:

```csharp
public class Order {
    private bool _isShipped;
    private bool _isDelivered;
    //.. more fields 
    
    public void AddLineItem(LineItem item) {
        if (_isShipped) {
            throw new InvalidOperationException("Can't add items to an already shipped order!");
        }
        // ... implementation follows
    }
    
    public DateTimeOffset GetEstimatedDeliveryDate() {
        if (_isDelivered) {
            // ...special case, we can return the actual delivery date as the "estimated" date 
        } else if (!_isShipped) {
            throw new InvalidOperationException("Can't estimate the delivery date for an order that has not yet shipped!");
        } else {
            // ... implementation follows
        }
    }
    
    public decimal GetOrderTotal() {/*...*/}
}
```

Assume there is an `IOrderService` which allows us to fetch all `Order` instances in a list. A problem arises when we try and iterate over the items (to show the estimated delivery date for each) - we (**as the caller**) need to either
* "Look before we leap" by checking the status of the order
* Be prepared to catch the exception that could be thrown (if the order isn't yet shipped) 

The problem isn't so much that we need to take a preventive measure, it's that
* We have to **know** that we need to take one
  * This knowledge has to come from out-of-band (by reading documentation) 
* Even if we know that we need to take a preventive measure, the compiler doesn't help us if we forget 

Unfortunately, in practice, often the way that we learn is by our code blowing up in production 💣

The modelling could be improved by using more specific types - perhaps we'd make `Order` abstract and introduce `NewOrder`, `ShippedOrder` and `DeliveredOrder` - these types would only support the operations that make sense for them:
* No `AddLineItem` on a `DeliveredOrder`
* No `GetEstimatedDeliveryDate` on a `NewOrder`

This is definitively an improvement, but it's not perfect - the calling code still needs to perform a runtime type-check and cast the abstract type to the appropriate subtype. If a new subtype is introduced (e.g. `CancelledOrder`) the calling code will need to be updated.  

#### Sum types (discriminated unions) to the rescue 🚑

It turns out that the three problems mentioned above i.e.
* "calculation could fail/is impossible" (max of empty list)
* "value might be missing" (null) &
* "valid operations depend on state" (estimate delivery date) 

basically boil down to the same thing: **we need the type-system to capture that a given value is of _exactly_ one of several types** (mutually exclusive) and for the compiler to force us to deal with those possibilities.  

Languages like Haskell have native support for expressing that a type is "one of" several other types - this is known as a "sum type"[^4]. Other languages like F# have something similar (called discriminated unions).

A discriminated union is a type T that is composed of two or more types (`P | Q | R...`) with a field/tag that indicates (to the compiler) which type a value **actually** is (i.e. _discriminates_ whether the value is actually a P, a Q or an R).

As a concrete example (stolen from [here](http://learnyouahaskell.com/making-our-own-types-and-typeclasses)), assume our program needs to model geometric shapes. 

We define a shape type as being either a `Circle` or a `Rectangle`
* A circle has a (x, y) coordinate (`center`) and a `radius`
* A rectangle is defined by two (x, y) coordinates (`topLeft`) & `bottomRight`)

With discriminated unions we not-only need to specify the core data for each shape (`center` + `radius` for a circle / `topLeft` + `bottomRight` for a rectangle), we need to capture the shape type in a field common to all shapes - e.g. `shapeType`. The value (e.g "CIRCLE" / "RECTANGLE") needs to be unique for each constituent type.

We can then write a function that takes a shape and calculates the surface area:
* The compiler forces us to handle every possibility in the union
  * If a new type `Triangle` is added to the union, our code won't compile until we handle the triangle case
* The compiler ensures we can only use the data that pertains to the type in question - e.g. if we're handling the square case, we're unable to access the "radius"

Unfortunately, C# doesn't have discriminated unions yet (although there is a [proposal](https://github.com/dotnet/csharplang/blob/main/proposals/discriminated-unions.md) to add them). For now, there's a great library called [OneOf](https://github.com/mcintyre321/OneOf) that uses source generators to add F# style unions to C# without too much boilerplate.

Here's an example showing OneOf in action:

```csharp
public record Point(int X, int Y);
public record Circle(Point Centre, double Radius);
public record Rectangle(Point TopLeft, Point BottomRight);

// A shape is either a Circle or a Rectangle
public class Shape : OneOfBase<Circle, Rectangle>
{
}

public class ConsumingCode
{
    public void WithSomeShape(Shape someShape)
    {
        // The compiler will force us to deal with all the "shape" possibilities here
        double area = someShape.Match(
            circle => Math.PI * circle.Radius * circle.Radius,
            rect => Math.Abs(rect.BottomRight.X - rect.TopLeft.X) * Math.Abs(rect.BottomRight.Y - rect.TopLeft.Y)
        );
    }
}
```

Interesting note: Mark Seemann points out that the good old visitor pattern [can replicate sum types](https://blog.ploeh.dk/2018/06/25/visitor-as-a-sum-type/) (albeit with a lot more ceremony) if you're having trouble selling functional style programming to your colleagues.

For a deep dive into sum types (and product types), see this [post on algebraic-data-types](https://jrsinclair.com/articles/2019/algebraic-data-types-what-i-wish-someone-had-explained-about-functional-programming/)

A couple of particularly useful (idiomatic) sum types are "Option" and "Either":

##### Option

`Option<T>` (also known as "Maybe") is a sum type that represents two possible (mutually exclusive) cases:
* It can contain a value of type `T` ("some T") **OR** 
* It can be empty ("nothing")

It can be thought of as a safe alternative to returning `null`. As mentioned earlier, if we attempt to call a method / read a property on `null` our code will blow up at runtime - `Option` avoids this problem.

The key operations we can call on a `Option<T>`:
* `Map`: Takes a function to transform a `T` into a `TNew`
  * If the option contains a T, transform the T value (possibly changing the type into `TNew`) and return a `Option<TNew>`
  * Otherwise (if the option contains Nothing), return an _empty_ option (of type `TNew`)
* `Bind`: Takes a function to transform a `T` into a `Option<TNew>`
  * If the current option contains a T, invoke the supplied function (which returns `Option<TNew>`) and remove one layer of "wrapping" - final return type is `Option<TNew`>
  * Otherwise (if the option contains Nothing), return an _empty_ option (of type `TNew`) 

Both `Map` and `Bind` are used to transform the value inside the option **if it contains something** - if the option is empty then both `Map` and `Bind` simply return an empty option. The only difference between the two operations is that `Bind` removes one level of wrapping ("flattening" the result) - allowing you to use transformations that return `Option<TNew>`

Some example code showing `Option` in action:

```csharp
// All addresses need to have a city and country. State is optional (depends on country)
public record Address(string City, string Country, Option<string> State);
public record Customer(Address Address);

public class ConsumingCode
{
    public void Example1()
    {
        var cityCode = CustomerRepository.FindCustomer(Guid.NewGuid())
            // If we successfully found the customer, pull out the address (& the city from the address)
            // This changes the type - previously it was Option<Customer> (returned by FindCustomer) but the type is now Option<string>
            .Map(c => c.Address.City) // func1: extract address city
            // Transform the value inside our Option (take the substring). Note that the type doesn't change with this
            // operation
            .Map(city => city.Substring(0, 2)); // func2: compute substring

        // NB: If the CustomerRepository.FindCustomer returned an option containing None, neither of the functions above
        // (i.e. func1 / func2) would be executed (if the Option contains None, .Map does nothing)

        // To "get at" the value inside cityCode, we need to handle both possibilities
        var message = cityCode.Match(
            // Handle the case where the cityCode contains a value
            value => $"The city code is {value}",
            // Handle the case where cityCode is empty / contains nothing
            () => "No city code available"
        );

        Console.Out.WriteLine(message);
    }

    public void Example2()
    {
        var state = CustomerRepository.FindCustomer(Guid.NewGuid())
            // If we successfully found the customer, pull out the address
            // This changes the type from Option<Customer> to Option<Address>
            .Map(c => c.Address)
            // The "Bind" operation allows us to call a function that *itself* returns an Option without ending up with
            // double-nesting - after the Bind operation, the type is Option<string>. If we'd used .Map instead, the type
            // would be <Option<Option<string>>. Bind is sometimes known as "FlatMap" or "SelectMany"
            .Bind(address => address.State); // func3: extract the (optional) state from the address

        Option<string> stateCode = state
            .Map(stateName => stateName[..1] + stateName[^2..1]);
        
        // To "get at" the value inside stateCode, we need to handle both possibilities
        var message = stateCode.Match(
            // Handle the case where the stateCode contains a value
            value => $"The state code is {value}",
            // Handle the case where stateCode is empty / contains nothing
            () => "No state code available"
        );        
    }
}

public static class CustomerRepository
{
    // Method signature indicates that we might not succeed in finding the customer
    public static Option<Customer> FindCustomer(Guid id)
    {
    }
}
```

##### Either

`Either<TLeft, TRight>` is a sum type that represents two possible (mutually exclusive) cases:
* It might contain a "left" value (of type `TLeft`) **OR**
* It might contain a "right" value (of type `TRight`)

An `Either` in the "left" state represents a failed computation (any subsequent steps will be _skipped_, as with `Option`). An `Either` in the "right" state represents a computation that has thus far has succeeded (and subsequent steps will be executed).

In many ways `Either` is very similar to `Option`, the primary difference being it allows capturing an indication of **what went wrong**. It's an excellent alternative to using exceptions for flow control. 

As with `Option`, the key operations we can call on a `Either<TLeft, TRight>` are:
* `Map`: Takes a function to transform a `TRight` into a `TRightNew`
  * If the `Either` is in the "right" state (i.e. it contains a `TRight` value), `Map` calls the supplied function to transform the `TRight` value (into a `TRightNew`) and returns a `Either<TLeft, TRightNew>` (in the "right" i.e. success state)
  * Otherwise (if the `Either` was in the "left" state - representing failed computation), `Map` skips calling the supplied function and simply returns a `Either<TLeft, TRightNew>` in the "left" state
* `Bind`: Takes a function to transform a `TRight` into an `Either<TLeft, TRightNew>`
  * If the `Either` is in the "right" state (i.e. it contains a `TRight` value), `Bind` calls the supplied function which returns an `Either<TLeft, TRightNew>` & removes one layer of "wrapping" - the final return type is `Either<TLeft, TRightNew`> 
  * Otherwise (if the `Either` was in the "left" state - representing failed computation), `Bind` skips calling the supplied function and simply returns a `Either<TLeft, TRightNew>` in the "left" state

Both `Map` and `Bind` are used to transform the value inside the `Either` **if the computation has succeeded so far**. If (on the other hand) the computation has failed (the `Either` is in the "left" state) then both `Map` and `Bind` simply return an `Either` in the "left" state. The only difference between the two operations is that `Bind` removes one level of wrapping ("flattening" the result) - allowing you to use transformations that themselves return an `Either`.

Note that while `Bind` allows the "right" type to evolve (i.e. `TRight` -> `TRightNew`) it does not allow the "left" type to evolve. Some implementations offer operations (e.g. `MapLeft`) to evolve the "left" type, but it's not really a core part of `Either`. 

Some example code showing `Either` in action:

```csharp
public enum ErrorKind
{
    NotBlogFound,
    CommentTextInvalid
}

public record BlogPost(Guid BlogPostId, string Title, IReadOnlyCollection<CommentDto> Comments);
public record CommentDto(string CommentText);

public class BlogController : Controller
{
    [HttpGet]
    [Route("v1/[controller]/{blogId:guid}/Title")]
    public IActionResult FetchBlogTitle(Guid blogId, CommentDto comment)
    {
        // LoadBlog returns Either<ErrorKind, Blog>
        return BlogService.LoadBlog(blogId)
            // If we've successfully loaded the blog, pull out the title
            .Map(post => post.Title)
            // Match() forces us to handle both cases of the Either, unifying down to a single type (IActionResult in this case)
            .Match<IActionResult>(
                title => Ok(title), // return type from Ok() is IActionResult
                errorKind => BadRequest($"There was a problem: {Enum.GetName(errorKind)}") // return type from BadRequest() is IActionResult too
            );
    }

    [HttpPost]
    [Route("v1/[controller]/{blogId:guid}/Comment")]
    public IActionResult AddComment(Guid blogId, CommentDto comment)
    {
        // LoadBlog returns Either<ErrorKind, Blog>
        return BlogService.LoadBlog(blogId)
            // The "Bind" operation allows us to call a function that *itself* returns an Either without ending up with
            // double-nesting - after the Bind operation, the type is Either<ErrorKind, Guid>. If we'd used .Map instead, the type
            // would be <Either<ErrorKind, Either<ErrorKind, Guid>>. Bind is sometimes known as "FlatMap" or "SelectMany"
            .Bind(blog => BlogService.AddCommentToBlog(blog, comment))
            .Match<IActionResult>(
                createdCommentId => Ok(createdCommentId), // return type from Ok() is IActionResult
                errorKind => BadRequest($"There was a problem: {Enum.GetName(errorKind)}") // return type from BadRequest() is IActionResult too
            );
    }
}

public static class BlogService
{
    // Method signature indicates that we might not succeed in loading the blog
    public static Either<ErrorKind, BlogPost> LoadBlog(Guid blogId)
    {
    }

    // Method signature indicates that adding the comment could fail
    public static Either<ErrorKind, Guid> AddCommentToBlog(BlogPost post, CommentDto comment)
    {
    }
}
```

For more information on how to use the Either type, see the excellent "railway oriented programming" [slides](https://fsharpforfunandprofit.com/rop/#slides). 

If you're looking to use `Option` & `Either` in your C# code, consider using the library [language-ext](https://github.com/louthy/language-ext) which offers fully fleshed-out implementations.

[^4]: Sum types are named due to how the "value space" grows as we add possibilities. If we have a sum type that is _either_ a boolean _or_ a byte i.e. `Boolean | Byte`, there are 2 (true/false) + 256 (0, 1..255) = 258 possible values. By contrast, a "product type" that combines a boolean with a byte has 2 (true/false) * 256 = 512 possible values.

### Higher Ordered Functions

In the object-oriented world, we frequently encounter methods that
* Accept objects (not just primitive values) as parameters
* Return an object (rather than a primitive value)

There is a symmetry in the functional-programming world - we have functions that
* Accept other functions (not just primitive values) as parameters
* Return a function (rather than a primitive value)

Such functions are known as _Higher order Functions_ (HoFs). HoFs are one of the primary means for code reuse in functional programming.

In the previous section we saw that `Either` and `Option` were able to offer safety through inversion of control - instead of reaching inside to get the value/result, you (the caller) provide code (a function) to _consume_ the value inside (if present). This frees you from having to remember to check which state the `Option` / `Either` is in. Because `Map` and `Bind` take a function as an argument, they're considered higher-order functions. 

And to finish off, here's an example of a higher order function where a function is created from data:

```csharp
public record ValidationRule(string FruitName, string InvalidReason);

public class ValidationExample
{
    // Imagine these were loaded in e.g. from a database. Also imagine the rule definitions support some additional
    // complexity e.g. they could include an operator name (GREATER_THAN, LESS_THAN, CONTAINS)
    private static readonly ValidationRule[] ValidationRules = new[]
    {
        new ValidationRule("pineapple", "Too Prickly"),
        new ValidationRule("potato", "Not a fruit"),
    };

    public void ConsumingCode()
    {
        var fruitToCheck = "pineapple";
        
        // Create a validator from one of the validation rules. In reality, we might map (iterate) over the collection
        // of validation rules to create a list of validators.
        // We could then apply them sequentially to produce a list of failure reasons        
        Func<string, string?> validator = CreateFruitNameValidator(ValidationRules[0]);
        
        // Call our validator function
        var invalidReason = validator(fruitToCheck);

        if (invalidReason != null)
        {
            Console.Out.WriteLine($"Fruit {fruitToCheck} is invalid because {invalidReason}");
        }
        else
        {
            Console.Out.WriteLine($"Fruit {fruitToCheck} is valid!");
        }
    }

    // CreateValidator is a higher-ordered function because it *returns* a function
    private Func<string, string?> CreateFruitNameValidator(ValidationRule validationRule)
    {
        return value => value.ToLower() == validationRule.FruitName ? validationRule.InvalidReason : null;
    }
}
```

## Summary

This is just scratching the surface of functional programming, but hopefully I've sparked ✨ your interest! See the resources listed below for further reading. And please add comments if you have feedback or anything to share 😊

## Resources / Further reading

* [language-ext](https://github.com/louthy/language-ext) (C# library adding various functional features)
* [Functional Programming in C#](https://www.manning.com/books/functional-programming-in-c-sharp)
* [Functional Programming Jargon (Github)](https://github.com/hemanth/functional-programming-jargon) (Glossary of FP terms)
* [The Not-So-Scary Guide to Functional Programming](https://www.yld.io/blog/the-not-so-scary-guide-to-functional-programming/)
* [Monads explained in C# (again)](https://mikhail.io/2018/07/monads-explained-in-csharp-again/) Brief explanation of the Monad (theoretical view of Either and Option)
* [How functional programming can improve testing, reuse, & maintenance in your current codebase](https://www.youtube.com/watch?v=NZ3-fsPIiYM) (Goes further into distinguishing actions from calculations, from the book "Grokking Simplicity")