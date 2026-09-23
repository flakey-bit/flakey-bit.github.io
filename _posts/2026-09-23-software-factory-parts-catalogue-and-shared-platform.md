---
title: 'Software factory? Start with the parts catalogue and the shared platform'
date: 2026-09-23T12:00:00+13:00
author: eddiewould
layout: post
permalink: /2026/09/23/software-factory-parts-catalogue-and-shared-platform/
spay_email:
  - ""
categories:
  - Uncategorized
---

Even though LLMs make code generation cheap, we should still find ways to deliver features with less code.

Fast, sustainable software delivery has always come from building and *using* great abstractions. Solid abstractions create *leverage* - they enable us to deliver more functionality while owning less code - code is a **liability**: the less you have to maintain, the better.

LLMs allow us to quickly generate *vast* quantities of code. *Left unchecked*, this could lead to a hundred slightly different implementations of the same thing across the company. Each variation brings its own bugs & maintenance burden. Of course, this happened before LLMs, but **LLMs amplify whatever you're already doing - good or bad**.

Currently, the idea of a "software factory" is growing in popularity online. However, if the success of the factory is measured by the number of lines of code produced, **we've made a wrong turn**. Car manufacturers figured this out decades ago: they adopted a shared chassis (platform) that was used across a family of vehicles, layering model-specific variations on top of the common platform. This approach drastically reduced R&D costs - iron out the defect once and it's gone everywhere. Design it once, reap the benefits for years to come.

Perhaps before focusing on the software factory itself, software organisations should focus on some foundational work, the two things this post argues for:

1. A "parts catalogue" (libraries)
    - Building blocks at several levels of abstraction. You need both big blocks and the little blocks the big blocks are built from. The parts need to be discoverable!
    - Domain-specific-languages (DSLs)
    - Rules-engines

    This is not a new idea. It's not quite "build systems that build systems", but rather "build tools that allow describing a new system at a much higher level of abstraction" (see also: policy vs mechanism separation).
2. "Shared platforms" (application harnesses)
    - Instead of feature teams maintaining their entire application, we apply inversion of control: the team writes and owns a *plugin*, and the platform hosts everything else. The plugin contains just the aspects pertinent to the feature.

### A liability - really?

Yes - software is a liability. Or more accurately, software is **both** an asset *and* a liability simultaneously. Let me explain:

This post is geared primarily at SaaS (software-as-a-service) companies. The customers of a SaaS company don't rent the lines of code the company has written. Rather, they pay the company for **solving their problems and/or making their lives easier**.

Gross simplification, but customers pay for a set of features that they find useful, delivered to an acceptable (high) standard of quality, availability, trust etc. In other words, what the software **does** for the end user is the **asset**. *Optionality* also comes into play here - what the software *could do* (in the future) contributes positively on the "balance sheet".

Now onto the liability side. There are actually several ways that software behaves as a liability:

At the micro level, **every single line of code**

- Potentially contains bugs or vulnerabilities
- Needs to be understood (by humans and/or LLMs)
- Needs to be maintained/updated for cross-cutting concerns
    - In particular, verifying that updates have been applied correctly
- Needs to be kept consistent with patterns in the rest of the codebase as it evolves
- Is a potential source of coupling (makes it harder to change things elsewhere)

There's also significant liabilities associated with every component / service (deployable with a CICD pipeline). This will vary company-to-company, but 

- Planning/architecture documents
- Maintenance of the CI/CD pipeline
- Support channels / escalation paths
- Monitoring, logging, alerting & telemetry
- SLOs
    - In particular, keeping on top of how they’re performing
- Dashboards
- Runbooks
- Team ownership overheads
- Threat modelling
- Pen testing
- Secrets management
- Data classification
- Versioning / release strategy
- Quality gates
- Patching
- In the case of a REST/gRPC service, there's everything associated with making a network request (retries, error handling, rate limiting etc)

And that list isn't even exhaustive 🥵

As you can see, there are lots of (hidden?) costs! Hopefully, you'll agree that **even if we can speed up the generation of code and initialization of services/deployables, we should be judicious about doing so**. Just because you can, does not mean you should.

This problem isn't new with LLMs - we've had code generation/scaffolding tools for decades now. These tools would spit out large swathes of boilerplate code almost instantly, as well as automating several of the items from the component/service list above. *However*, for the most part, the generation is a one-off - **after initial generation, the responsibility lies with the owning team to maintain everything that was created**.

To be clear, I'm not saying teams shouldn't spin up new services. I'm also not suggesting to avoid scaffolding/code generation tools. Rather, I'm saying:

- We should be judicious & *measured* about **why** we're creating a new deployable (e.g. need to isolate sensitive/personal data; new service has different load/scaling requirements; need to isolate blast radius - etc) rather than doing it **by** **default**
- Ideally, we should do it in a way that minimizes the maintenance burden for the owning team

I'm also not saying we shouldn't write new code (or use LLMs to do so). Rather, I'm saying that we should be finding ways to write code that operates on a more abstract level - allowing the same code to solve many different variations of a problem. A couple of examples to illustrate

- We need to implement a business workflow (say, invoice approvals or something like that). After thinking for a bit, we realize that the workflow can be modelled as a finite state machine (FSM). Instead of writing a bespoke implementation of the approval workflow, we pull a (vetted, battle tested) FSM library off-the-shelf and supply our invoice workflow as *configuration*
- We realize that our error handling code is often a bit of a mess, frequently contains bugs and is intertwined with the happy-path. We adopt a pattern like the `Either` monad and apply it consistently.

The final way that software acts as a liability relates to optionality. If "what the software could do in the future" (optionality) is an asset, then anything that detracts from that optionality is a liability.

Kent Beck talks about this in [Features vs. Futures](https://www.youtube.com/watch?v=K7Z8Wa83Wgo) - the key idea is that every decision that has been locked-in (to support a feature) makes it harder to deliver features in the future (systems tend towards entropy which means more coupling, less cohesion - literally, chaos). Compare starting with a blank slate to adding to a legacy monolith.

Obviously, we don't want to stop adding features. But we need to do something to restore that optionality. This is the classical tech-debt workflow:

1. Design system for initial set of known capabilities
2. Implement system
3. Add new capability
4. Improve the design to cater to the new capability **so that the design looks as if you originally designed the system with that capability in mind**

One lens to view all of the above through is [accidental complexity](https://louismccormack.com/accidental-vs-essential-complexity) (complexity introduced by engineering decisions) vs essential complexity (complexity inherent in the problem space). A large part of our job as engineers is minimizing the accidental complexity associated with what we build.

### What I'm concerned about

A recent trend in the software industry towards "software factories" *seems* to be predicated on the idea that **if** we add **enough** guardrails, we can almost entirely remove the need for a human in the loop (HITL).

Basically, just give the agents free rein to solve the problem so long as they follow the specification we provided & do so within the confines of the rules (guardrails) we've set out.

Ended up with a result you didn't like? *Clearly*, what you need is another guardrail.

![One more lane (guardrail)](/images/2026-09-23/one-more-lane.png)

Don't get me wrong. I **love** ❤️ a good guardrail (a couple of my favourites are functional domain modelling to [make illegal states unrepresentable](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/) & expressing rules through [convention tests](https://github.com/andrewabest/Conventional)).

The issue is, while guardrails do great at the micro-level, they're not as helpful at the macro level.

In *particular*, I'm concerned about the lack of a forcing function to **minimize the amount of code that has to be maintained** holistically, for the company, across all their repositories.

If you give an agent/LLM a specification that describes an invoice approval workflow system (in terms of what the system must do), I suspect it will go ahead and build one for you with a bespoke implementation. It is unlikely to meaningfully consider

- How does this relate to other workflow systems that we may already have? Or may need in the future?
- Does this make appropriate use of APIs that already exist internally?
- Instead of implementing the workflow system as described in the specification, could we represent the invoice approval workflow as a finite state machine and use an off-the-shelf library/product to implement it?

All of the above considerations are examples of applying abstraction to generate *leverage* (doing more with less).

I think there's a risk that (with agentic workflows) we'll miss out on opportunities to look for and apply abstractions. Instead, we'll just accept an increasingly large ball of mud.

This creates a vicious cycle - LLMs generate **vast** seas of code, it becomes too hard for a human to meaningfully review, so we start looking for ways to remove the need for the human to review the changes. I think that's essentially where the industry has ended up (many teams are now trying to solve the review issue with guardrails).

But what if we get really good at writing specs, and let the LLM turn a [sufficiently detailed spec](https://haskellforall.com/2026/03/a-sufficiently-detailed-spec-is-code) into code for us?

![A sufficently detailed spec is just code](/images/2026-09-23/spec-is-code.png)

(this comic pre-dates LLMs). Well congratulations, now you've got **two** liabilities:

1. The spec file
2. The code generated (non-deterministically!) from the spec by the LLM *this particular run* - this is what's actually compiled and executed.

My take is that we (as an industry) are approaching this **entirely the wrong way**. Instead of accepting defeat on the review front (because there is too much to review!), we should be doing the **actual engineering** to ensure the designs/code we *do* produce is high leverage.

That means asking ourselves (or our agents)

- Does something like this already exist
- Is this making use of appropriate abstractions
- Should this be abstracted out
- Have we separated the policy from the mechanism

The underlying principle being that, overall, our pull requests should contain far less low-level code changes and far more configuration changes - describe the mechanism once and then make use of it MANY different times through policy. The commits should also be much smaller - the policy should describe the behaviour succinctly and declaratively.

Following the example earlier - if you

- Know what a Finite State Machine (FSM) is
- Can see how the problem at hand (invoice approval workflow) translates to a FSM
- Trust the FSM library

then the review becomes very focused - you're reviewing the states, inputs and transitions (which is all domain knowledge). Compare that to reviewing a bespoke implementation (which is likely to be many hundreds of lines of code).

**Success looks like**:

- The majority of our features delivered by writing succinct configuration/policy
- The sizes of our pull requests (to deliver a feature) goes down dramatically
- We have much smaller repositories in general

### My recommendations

So if I take a step back to define *what I think success looks like*, there are multiple parts to it

#### Externally observable

- We need to be shipping features much faster than we are now (improved velocity)
- But shipping fast is not good enough, our customers have to love the features we ship (building the right things) and we should have a low defect rate (building them right). Trust is hard to gain but so easy to lose.
- We must deliver *sustainably*. It's no good for us to ship the first five features of a product in a month and then take a year to deliver the next two features of that product

#### Internally observable only

- We should observe a **marked** reduction in lines-of-code-per-feature measured across the company. That's because we're operating at a higher level of abstraction
- We should see the majority of new features delivered primarily through succinct configuration changes
- We should see much lower code churn than we're currently seeing.

So given that's what (I think) success looks like, what do we need to do to get there?

I don't claim to have all the answers to this. Or at least, not **good** answers. It seems likely that there is [no silver bullet](https://worrydream.com/refs/Brooks_1986_-_No_Silver_Bullet.pdf). As stated already, I think that blindly accepting large quantities of LLM generated code into our repos (so long as it meets the guardrails set out at the repo level) is **not** the answer. It will not be sustainable over the long term. There's an analogy between the difference between local and global maximums of a function here - everything done inside that repository might be (arguably) defensible in isolation, but falls apart when considered in a broader (company) context.

With that said, in the spirit of "don't present a problem, present a solution", some ideas on what we can try:

#### Focus on up-levelling our engineering capability

It's all well & good to say that we should be developing at a higher level of abstraction - but (currently at least) it takes a high degree of expertise & experience to do so.

If you describe a set of product requirements to an LLM, it is currently unlikely to consider whether the problem fits a software engineering "well-known problem" or "design pattern" archetype. The wisdom to even ask these questions comes from experience.

Interviewing processes should test candidates' ability to recognize and apply these patterns - in other words, think in terms of abstraction.

In addition, we should be holding each other, as engineers, accountable to working in this way.

#### Embrace "modular monoliths" where it makes sense

As already mentioned, there are significant costs associated with creating (but mostly maintaining) each new deployable.

Obviously, we should create new services where there's a genuine need (independent scaling, blast radius or data sensitivity) but that should not be our only tool in our toolbelt.

If the only reason we're reaching for a microservice is to enforce a code organisation / ownership boundary (something akin to [Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law)) then we've not only accepted a whole bunch of maintenance burden for a new service, we've incurred the reliability and performance problems that come from introducing a network request / distributed system.

It is entirely possible to create a clear domain & ownership (module) boundaries within a single codebase. These boundaries can be enforced (e.g. using convention tests). In many cases, agents will work better (and more autonomously) when they can see the bigger picture.

#### Narrow implementation surface area

At least prior to the advent of LLMs, it was generally accepted that one of the best ways to work with junior engineers was to give them a deliberately constrained task. For example, you might give them an interface to implement. It was a great way of ensuring they stayed focused on the task-at-hand.

Of course, for the growth of that junior engineer it's important to give them the opportunity to see how the interface will be used & how it fits into the bigger picture.

Ironically (?), I think the same guardrail that narrowly-defining the implementation window provides for junior engineers is useful when working with LLMs (people keep saying they're like an overly-confident fresh graduate, so it makes sense).

I don't think this *necessarily* precludes agentic development or prescribes more hands-on - hypothetically, at least, the narrow window could be assigned to the subagent.

I also don't think this contradicts what I just talked about in "modular monoliths" - the narrow window is simply the well-defined module (rather than the microservice).

#### Building a discoverable "parts catalogue" (and using it)

Mature organisations are generally pretty good at reusing things at the platform level (e.g. concerns such as authentication, authorization, routing, eventing) and fairly good at dogfooding their own APIs.

But I see an opportunity for reuse at a lower level. Solve the **genuinely hard** problems underpinning what we do. Solve these problems generally, with solid implementations. Publish and commit to maintaining these as versioned packages (NuGet, npm, Maven, whatever fits your stack).

Make them discoverable by both humans and agents. The documentation needs to describe the kinds of problems it solves, how to use it, when to use it and when not to use it. We're going to hit a "Got a hammer? Every problem is a nail" situation - it's somewhat unavoidable. But at least we need to be aware of that tendency.

A bad abstraction is **absolutely** worse than no abstraction at all. To be successful, we must

- Ensure we're only abstracting genuinely hard problems
    - **Don't** create a library to share a boatload of shared configuration
- Modules should be narrow and deep. That means they must offer a simple interface that hides significant complexity behind it
- Take real care in the design of the abstractions
- Think about versioning/evolving the abstraction
- Build & offer abstractions in *layers*
    - The high-level abstractions are built using the low-level abstractions ("dogfooding")
    - Consumers should use the highest-level abstraction that meets their needs / is a good fit - dropping down to progressively lower levels of abstraction is the escape hatch
    - Read [this excellent article](https://surma.dev/things/cost-of-convenience/) that talks about building "good" abstractions

#### Invert control with the "plugin" model

Currently, industry best-practice is to run a scaffolding tool each time you want to create a new service. You get a whole swathe of code that you now must maintain. Theoretically, you should review the code that was spat-out for the initial commit - **does anyone actually do that**?

Kubernetes is great - it means we don't need to maintain the whole operating system any more. But you're still maintaining a significant portion of application code - this applies to both APIs and worker processes.

AWS Lambdas are very much a step in the right direction. We need to find ways to deliver more functionality as Lambdas.

But until that time comes, perhaps we can have pre-built binaries of

- A Web API
- A worker process
- ... a webhook host

i.e. the *harness*

The application team is responsible for building only a single assembly / DLL (the "plugin"). At deployment time, we deploy a build of the harness together with the build of the application code plugin. The plugin is loaded and functionality is resolved at load time.

Note that for practicality, we'd likely use a separate instance of the harness for each plugin But we *could* (theoretically) host multiple applications (plugins) in the one harness if we wanted to.

There's a contract between the host harness and the application code, so upgrading the harness is still an explicit action the application team must take. We should endeavour to minimize breaking changes, but not at the expense of improvements. In most cases, the upgrade process should involve just updating the harness version in one place.

Note that this architecture makes it difficult for application teams to implement concerns that should be handled at the platform level. **That's a feature, not a bug**. What it encourages:

- Conversations - perhaps there's a sanctioned way to achieve the objective that the application team isn't aware of
- Consideration of future similar use-cases that are likely to crop-up (i.e. does this need generalize)
- Introduction of extensibility points - all else failing, the application team should consider offering an extensibility point for the concern

If this sounds very much like what happens with a Lambda, that's quite deliberate. There needs to be hard boundary between code that the application team maintains and the platform team is responsible for maintaining.

In general, your code should be agnostic to the platform that it runs on to the extent possible (keeping in mind performance considerations). In the spirit of "keep the cost of being wrong low" and "make code easy to delete", it should generally not be a big deal to convert your application plugin code to run inside a Lambda instead (or vice versa).

### What should happen regardless

*Regardless* of how code is generated (hand tooled/artisan or LLM generated) and regardless of whether that code is composed from abstractions or not, **the priority is shipping safely and with confidence.**

There are a bunch of practices worth adopting to achieve that:

- Ideally, all deployments should be automatically canaried **without** human involvement. 
    - Systems should self-monitor the performance of the canary (relative to baseline), primarily by observing user behaviour and traffic patterns
    - Multiple, in-flight canaries running for the same deployable simultaneously will be the norm. That might mean it’s some time before a feature fully lands for all users.
    - Feature flags will still have a role, but increasingly for A/B testing or product/marketing reasons rather than controlling blast radius
- Double-down on telemetry & observability (OTel, Prometheus, Grafana etc)  
    - Can’t have automated self-monitoring canaries without it
- By default, breaking changes to API contracts should be prevented
    - A tool like `openapi-diff` can tell you if you’ve introduced a breaking change, provided your APIs generate an OpenAPI schema
    - If a breaking change is needed, then some coordination will be needed across consumers

One bonus of the shared harness approach is that it makes it much easier to implement and iterate on these engineering practices.

### Conclusion

Code being cheap to generate (with LLMs) makes bespoke code look free, but that is far from the case (the cost has moved from writing it to owning it).

We should be focusing on building high leverage software - increasing leverage (improving the asset:liability ratio) is the primary way to improve the profitability of the software we build.

We should free application teams from the responsibility of maintaining what essentially amounts to platform code.

Finally (and perhaps most importantly) - we should **absolutely** still improve our workflows & processes to safely ship changes with confidence.
