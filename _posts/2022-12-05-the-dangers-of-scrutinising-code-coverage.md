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

The dangers of scrutinising code-coverage

## Introduction

Recently, a proposal was put forward at my organisation that 
* All teams **must** report on code (line/branch) coverage in SPAs (web applications) we build
* Teams (ideally) **should** set a coverage threshold level and fail builds when it drops below the threshold.

Here are my thoughts on why this (surprisingly common) idea is a misguided one.

## Common misconception:
"The more automated tests in the codebase, the better".

Actually, every test has a (not insignificant) maintenance cost associated with it. If a test isn't providing value (whether that value is 
* enabling safe refactoring
* preventing regressions or 
* documenting correct usage

) it should be removed.

## Can tests be harmful?
While it won't set your dog on fire, a test with negligible or zero value is harmful in the sense that you still pay the maintenance (and initial build) cost associated with that test. 
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

* For some teams there may be technical challenges associated with generating the a (consolidated) report (e.g. different test frameworks used for different parts of the application)
  * From the outside in it's impossible to see the nuances that might be at play here
* A single coverage % number doesn't take into consideration the app in question (which parts are important to cover). 
  * A single number doesn't tell you whether it's the important parts that have been covered or the trivial / unimportant parts
* It can very easily lead teams into a false sense of security - "We're at 100% branch coverage, so we're good"
  * If the assertions are non-existent or rubbish, you might as well have 0% coverage
* See Goodhart's law: "When a measure becomes a target, it ceases to be a good measure" 🎯
  * Simply by asking teams to report on the coverage %, undue focus will be put on increasing the coverage % - to the detriment of other aspects.
* Once you accrue a lot of negative-value tests, teams will start to de-value test failures (broken window syndrome 💔)
* Often, it's difficult to maintain a high coverage percentage without resorting to testing **implementation details** (and/or tests that simply don't reflect real-world usage). 
  * This is the most concerning aspect of the proposal - Goodhart's law tells us that teams **will** write these negative-value tests (due to the implicit target) 
  * As noted above, such tests **slow teams down** and **create barriers to refactoring** (bad)