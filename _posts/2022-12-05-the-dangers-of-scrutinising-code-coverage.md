---
title: 'The dangers of scrutinising code-coverage'
date: 2022-12-05T14:13:00+13:00
author: eddiewould
layout: post
permalink: /2022/12/05/the-dangers-of-scrutinising-code-coverage/
spay_email:
  - ""
categories:
  - Uncategorized
---

## Introduction

Recently, a proposal was put forward at my organisation that 
* All teams **must** report on code (line/branch) coverage in SPAs (web applications) we build
* Teams (ideally) **should** set a coverage threshold level and fail builds when it drops below the threshold.

Here are my thoughts on why this (surprisingly common) idea is a misguided one. 

**TL;DR**: Successful teams focus on high value tests 🏆. Asking teams to report on code-coverage is likely to result in low (even negative!) value tests being written 📉.      

## Common misconception:
> "The more automated tests in the codebase, the better".

Actually, every test has a (not insignificant) maintenance cost associated with it. **If a test isn't providing value it should be removed**. 

Generally speaking, there are three ways a test can provide value 
1. It enables safe refactoring
2. It prevents regressions 
3. It documents correct/example usage

## Can tests be harmful?
While it won't set your dog on fire, a test with negligible or zero value is harmful in the sense that you still pay the maintenance (and initial development) cost associated with that test. 
Such a test has a **net negative value**. Net negative tests slow your team down.

Tests that check implementation details are _particularly_ susceptible to this - every time the implementation changes, a corresponding (mirroring!) change must be made to the tests.

A potentially less obvious (but more important consideration): Not only do such tests not assist refactoring, they actually **create barriers**. 
A well-intentioned developer might start down the path of some [boy-scout refactoring](https://deviq.com/principles/boy-scout-rule), only to see a hundred false-positives in terms of failing tests - at which point they give up without performing the refactoring. This, is how codebases slowly rot 🤮.

This blog post covers the topic well: [Kent C Dodds: Testing implementation details](https://kentcdodds.com/blog/testing-implementation-details)

## What do valuable tests exercise?
**Use cases of the code in question**. What these use-cases look like depends on how the code is consumed and who the "user" is. Sometimes, there will be multiple "users"

* For bespoke functionality in a SPA, the "user" is literally the human interacting with the web application through the UI. These users don't directly use your Redux sagas / reducers / selectors. They use the web application
* For a library/API/reusable component etc, a possible "user" is the developer who is integrating the library/API/component in question

This blog post sums it up really well IMHO: [Kent C Dodds: Avoid the test user](https://kentcdodds.com/blog/avoid-the-test-user) 

## What does code-coverage do?
To simplify greatly, code-coverage instruments your production code when your tests run and examines which lines/branches (i.e. IF/ELSE) are hit by the tests. 

However, the fact that an automated test has hit a line/branch does not give any guarantee at all that
* The logic the line/branch implements is correct 
* The line/branch is necessary

In fact, it doesn't even ensure the line/branch executed without throwing an exception!

I will concede that (_generally speaking_) if the important use-cases (see "What do valuable tests exercise") are tested AND the SUT code does not contain redundant logic, the line/branch coverage should be fairly high. 
Also, examining the coverage can **certainly** be a useful tool 🔨 in the belt of the developer "These lines aren't being hit at all - Oops!, I've forgotten to write tests covering use-case x"

This blog post goes into some more detail: [Kent C Dodds: Common testing mistakes: 100% code coverage](https://kentcdodds.com/blog/common-testing-mistakes#mistake-number-2-100-codecoverage) 

> There's no one-size-fits-all solution for a good code coverage number to shoot for. Every application's needs are different. I concern myself less with the code coverage number and more with how confident I am that the important parts of my application are covered. I use the code coverage report to help me after I've already identified which parts of my application code are critical. It helps me to know if I'm missing some edge cases the code is covering but my tests are not.

## What are the problems with asking teams to report on code-coverage?

### Unjustified technical cost
As mentioned above, code coverage can be a useful tool for a developer to look at when deciding whether they have written the right tests. 
However, for some teams there could be significant technical challenges in automating the generation of the coverage report.

If the team in question doesn't care about the coverage number, we've effectively foisted a bunch of toil on that team for no gain

### A single number misleading
A single coverage % number doesn't take into consideration the app in question (which parts are important to cover). If we're at 50% coverage is that 100% of the important business critical-code and 0% of the boring infrastructure, or the other way around?

### False sense of security
Code coverage numbers can easily lull teams into a false sense of security - "We're at 100% branch coverage, so we're good"

If your assertions are non-existent or rubbish, you might as well have 0% coverage

### Goodhart's law 🎯
Goodhart's law says that "When a measure becomes a target, it ceases to be a good measure". 

Simply by asking teams to report on the coverage %, undue focus will be put on increasing the coverage % - to the detriment of other aspects.

### Negative value tests
Once you accrue a lot of negative-value tests, **teams will start to de-value test failures** (this is known as the broken-window syndrome 💔).

It spells bad news - we want to be in a place where a failing test means something has been broken badly and we've dodged a bullet 🔫. Don't let your tests cry wolf! 

### Encourages testing implementation details

Often, it's difficult to maintain a high coverage percentage without resorting to testing **implementation details** (and/or tests that simply don't reflect real-world usage). 

In my opinion this is the most dangerous 💀 (but also invisible) second-order effect from asking teams to report on code coverage. 
Goodhart's law tells us that teams **will** write these negative-value tests (due to the implicit target). As noted elsewhere, these negative-value tests **slow teams down** and **create barriers to refactoring** (bad).

## Conclusion

Code coverage is a useful tool, but we should take care when mandating reporting around it. 

> It doesn't make sense to hire smart people and then tell them what to do; we hire smart people so they can tell us what to do - Steve Jobs