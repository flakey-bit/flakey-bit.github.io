---
title: 'Sustainable software delivery: The core-capabilities & plugins model'
date: 2024-02-23T12:00:00+13:00
author: eddiewould
layout: post
permalink: /2024/02/23/sustainable-delivery-core-capabilities-and-plugins-model/
spay_email:
  - ""
categories:
  - Uncategorized
---

An approach to managing bespoke behaviour (to support sustainable delivery).   

### Introduction

Have you ever maintained a product (codebase / component) that
- Frequently has other teams "needing to make changes" to it (often, last-minute!)  
- Seems to be constantly growing 🌱 in complexity & getting harder to understand/maintain?

and wondered how to wrangle it into a better shape? Then this post might be for you!

### What is product ownership?

The idea of product ownership is nuanced, but three key aspects are
- Deciding _which_ problems/opportunities to go after (solve) & _how_ they should be approached
- Setting the (technical) direction for the component 
- Looking out for the well-being of codebase over time (like a parent caring for a vulnerable child - the world is a scary place!) 

You can follow the parent/child analogy a bit further - it's a **long-term** relationship (spanning years) & there is a sense of investment/pride.    

Ownership doesn't (necessarily) mean disallowing external code contributions, but it **does** mean there is a clear person (or team) looking out for it ❤️ 

### Bespoke behaviour driven by "pet" 🐶 data (conditionals) doesn't scale

Imagine a hypothetical application that deals with "company" entities; companies (stored in a database) are added/removed/updated on a semi-regular basis.

In the (trivial) code below

```javascript
export const render = (company) => {
    if (company.code === 'MSFT') {
        renderLogo('./Microsoft.png'); // or worse, renderMicrosoftLogo();
        renderLink('https://www.microsoft.com/en-us/', 'Microsoft');
    } else if (company.code === 'NVDA') {
        // No logo for NVidia, just a link
        renderLink('https://www.nvidia.com/en-us/', 'NVidia');
    } else if (company.code === 'ACME') {
        renderLogo('./ACME-logo.png');
        // ACME pays us for the referrals we send them. Needs a JavaScript snippet 
        renderLink(
            'https://www.example.com/', 
            'ACME', 
            { monetizeReferral: true, referrerId: 'd832d050-3cb0-43fe-8ad1-eb72eb1cf5b5' }
        );        
    }
}
```

the behaviour is tied directly to specific data cases (`MSFT`, `NVDA`, `ACME`). 

This kind of code **scales poorly** (from a maintenance perspective):
* Adding or removing a company typically requires a code change
* It's easy to end up with dead (unreachable) code

In a nutshell, the problem is that we're treating each company as a "pet" - we should be treating them collectively as "cattle" - there should be no special-snowflakes ❄️

The problems are amplified when the knowledge of "pets" is _scattered throughout_ the codebase (not limited to a single location) and _especially_ when there is combinatorial explosion due to interactions between different kinds of entities (that are treated as pets). 

At the risk of giving away the plot, at a _high-level_ the solution is often
* Moving code to configuration
* Inversion of control
* Extension points for "pluggable" behaviour
* Indirection

### Different kinds of "users"

Almost all software products have "end-users" (the people who directly interact with your product) - the ones we know and love. 

But in addition to end-users, it's possible that you have _other_ kinds of users to consider:
* If your product is "integrated into" (part of) another team's system, that external system can be considered a "user" of your product   
* If your product displays/aggregates content from other teams, the content-providing teams should be considered users too

☝️ These additional users are often referred to as "stakeholders" 🥩  

### What usually happens

Unfortunately, what often happens in these content-provider scenarios is some kind of "shared-ownership" where teams "do whatever is easiest" to meet _their_ requirements, without regard for the long-term health of the product.

* ⚠️ "I'll just copy + paste the `FooBar` implementation and tweak it slightly"
* ⚠️ "Oh! There's a bug in the `FooBar` implementation (that I just copy+pasted). I don't want the hassle of fixing it there though"
* ⚠️ "It would be much easier to write this code if I was provided a `<Baz>`. Oh well" (proceeds to write a buggy `<Baz>` from scratch)

Shared-ownership might as well be called "no ownership". 

As time goes on, the software inevitably enters a [change-aversion doom-loop](https://software.rajivprab.com/2019/11/25/the-birth-of-legacy-software-how-change-aversion-feeds-on-itself/): it gets harder to clean the software up / refactor things and you end up with a legacy product.  

### Enter: The core-capabilities & plugins model

The key problem here is managing complexity. One way of tackling it is by applying the "core-capabilities & plugins model".

We take a modular, product-oriented view to a component (imagine you are offering a SaaS product). That is to say
* The product has core capabilities (which are _always_ enabled)
* The product has _optional_ capabilities (can be turned on or off through configuration)
* Finally, the product offers extensibility points where functionality can be _plugged in_ 🔌

Each extension point basically acts as an "escape hatch" & allows **inversion of control**. 

By writing a plugin, the team that requires the non-standard behaviour (complexity) becomes responsible for owning/maintaining that complexity.

The complexity is isolated from the rest of the application (minimal API surface). 

So long as you don't change the extension point API surface, you're free to evolve the rest of the application without worrying about that complex special snowflake.

##### Triaging

When a change request comes through, we can put it into one of three buckets

* Is our product missing some core capability? 
  * We (the owners) should clearly build that
* Is our product missing a configurable (opt-in) feature?
  * If that's something that would be somewhat generally applicable, we should probably build that
* If not, are we missing an extensibility point that allows injection of bespoke behaviour?
  * We should design the API surface for that extension point and probably provide a reference implementation for a plugin (example)

As the owners of the product, it's **crucial** that this initial triaging goes through us - we've got the vision for the product and we know what _other_ teams are up to (features that have been requested).

This is all largely a case of shifting left ⬅️: we want teams to come to us with a problem/opportunity that they think might involve our product, rather than coming to us with a PR. 

Once opt-in features and extension points exist, teams that "need to make changes in our product" should be able to do so through configuration (rather than directly commiting code in our product).

By triaging (as outlined above) we avoid ending up with dozens of subtle variations of (essentially) the same feature, instead standardizing on a general one.

##### Quick example

For the example given in the section above (code dealing with company entities) we might do the following:

###### 1) Invert the control by moving code to configuration

We now store the rendering configuration for each company:

```javascript
// This would be loaded from the database or JSON config etc
const companies = [
  {
      code: 'MSFT',
      logo: './Microsoft.png',
      linkUrl: 'https://www.microsoft.com/en-us/',
      linkText: 'Microsoft'
  },
  {
    code: 'NVDA',
    logo: null,
    linkUrl: 'https://www.nvidia.com/en-us/',
    linkText: 'NVidia'
  }
];
```

The rendering code is updated to map-over the companies array and calls the functions `renderLogo()` / `renderLink()` as appropriate.

###### 2) Add an extension point for custom link rendering
In the first version of the code you'll see that for ACME, we render a _customized_ JavaScript link (for monetization). 

For now, we've decided against building first-class support for monetized referral links (note, we could change our minds on that in the future!) on the basis that it's not that common and it's quite involved. 

However, we want to make the ACME use-case (and other, similar use-cases) _possible_ - so we build an extension point.

For ACME, the configuration won't specify the link URL/text (like it does for MSFT/NVDA) but rather, the ACME configuration will specify the plugin that is used to render the link.

```javascript
const ACME = {
    code: 'ACME',
    logo: './ACME-logo.png',
    renderLinkPlugin: 'AcmeLinkRenderer'
};
```

When mapping over the companies array we'll invoke the `renderLinkPlugin` (if present) instead of calling `renderLink()`. 

The `renderLinkPlugin` will need to conform to a well-defined interface which documents the context/arguments it receives and the result (if any) it should return.

That interface is the extension-point API surface which must be **stable** & **evolvable** (**so take care when designing it**).

### Challenges
Taking this approach to delivery requires (unsurprisingly) a strong product-owner who is prepared to have difficult conversations.

##### Compromises
Frequently, special-snowflake ❄️ requests can be reduced to core (generally applicable/opt-in) features by making some some (often minor) **compromises**. 

In the long term, **this is actually the ideal outcome** for all parties; the team requesting the subtle variation might be unhappy they didn't get exactly what they asked for, but eventually they'll be grateful because the the feature gets maintained "for free" (since it's a common feature). **Software doesn't live in a vacuum**, it needs to constantly evolve to the environment around it.

##### Saying "No"
_Occasionally_, it will be necessary to say "No" - "What you're asking for is something our product will **never** do". 

☝️ ️️Providing guidance (about what they could do _instead_) in an empathetic way is key here. 

##### Tight Deadlines
Sometimes things are overlooked and only surface at the last minute. 

Inevitably, you'll have an urgent request come through that can't be "self-served" through the existing configurable features + extension points. 

In that case you might need to _deliberately_ take on some tech-debt to unblock the requesting team. But you should **absolutely** prioritise addressing that tech-debt (adding features and/or extension points) **as soon as possible**.     

### How to get there (legacy project)
So this all sounds great, but let's say you're responsible for a legacy project that wasn't built like this. How should you proceed?

##### Short term

When you find yourself in a hole, the first thing to do is stop digging - **don't add to the mess** (see also [ratcheting](https://qntm.org/ratchet)). 

The _first_ thing you should do is sketch-out (at a high-level):
* What are the (core) features your product should have
* What might be the optional/configurable features (at a high level) 

This is about forming a _vision_ for your product. If you don't know what your product currently does, then some discovery/exploration is in order.

You can then look through the existing code in your product and try to "rationalize" it into the features above (see if there are any close-ish "fits"). 

You **don't** need to go and rewrite all that code now (gosh no!) but rather, just build an internal map 🗺️ and try to see where things deviate.

That exercise will give you an idea of what extension points 🔌 might be needed (now or in the future).

##### Longer term
Over time (once you've built that map), you're going to want to consolidate the existing code to either core capabilities, optional features or move it to extension points. 

**This part is really hard** ⚠️ because you're trying to put the proverbial cat 🐱 back into the bag 👜 - you're essentially trying to convince your consumers / stakeholders to either
* Accept a more generic version of the feature (which will typically be perceived as "worse") - a hard sell
  * The only "carrot" 🥕 here is that they get free maintenance
* Own the complexity themselves (as an extension point plugin)

...neither of which are particularly palatable (an ounce of prevention is worth a pound of cure here!)

A possible compromise is to move the complexity to an extension point plugin, **but agree to maintain the plugin** (grandfathered arrangement).
* Eventually, the plugin might no-longer be needed (natural attrition)
* If significant changes to the plugin behaviour are requested, that's a good point to restart the conversation on plugin ownership

### Summary
* Managing complexity in software is tricky & an ongoing battle. Particularly, complexity that inhibits the evolvability of the software
* Often, people think they need something special when (in fact) something more general would do just fine
* The maintenance cost of special-snowflakes isn't always obvious & it can compound/multiply
* The core-capabilities & plugins model (if you have a strong product-owner) can help you manage this complexity