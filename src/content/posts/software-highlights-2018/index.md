---
id: '7de94b13-0076-42f8-ae08-a012de182ff8'
title: 'Software highlights of 2018'
description: "A collection of what I thought were 2018's most interesting developments in the software industry."
date: '2018-12-30 22:16:42'
tags:
  - Java
  - JavaScript
  - React
  - Kotlin
  - Go
---

As the year comes to a close, I think it's worth reflecting on some of the interesting developments
over the last year that are worth re-visiting to discuss their impacts. Here are my top five ranked
in order:

## 1. Java 11 released

This makes the top spot as it's the natural upgrade path for many teams that are currently working
with Java 8. This is considered to be a major release as it will receive LTS support until at least
September 2022 on the AdoptOpenJDK [roadmap](https://adoptopenjdk.net/support.html#anchor4) (non-LTS
releases get 6 months of support).

It should be noted that Oracle have also changed their official support cycle so that they will
only support each new JDK for six months (this includes Java 11). The intention seems to be for
initiatives such as [AdoptOpenJDK](https://adoptopenjdk.net/) to become the main source of support
instead.

I personally see this as quite a positive step as we should hopefully see the community move away
from the proprietary Oracle JDK and onto the open-source OpenJDK. At this point, OpenJDK is
essentially developed in parity with Oracle JDK so there should be no technical difference (in most
situations).

Undoubtedly, I would still expect to see an increase in enterprises spending money on Oracle support
fees. This should be unnecessary most of the time, but enterprise teams are incredibly risk-averse
and will feel there is some level of 'safety' afforded from being on the Oracle JDK (shrug).

For more information, check out the comprehensive [Java Is Still Free](https://medium.com/@javachampions/java-is-still-free-c02aef8c9e04)
post that covers this topic in more depth.

### New language features since Java 8

- The new, _controversial_ module system (the reason that Java 9 was delayed significantly).

  This was controversial as it turned out that getting various JEP stakeholders to agree on the
  final implementation was... [difficult](https://developer.jboss.org/blogs/scott.stark/2017/04/14/critical-deficiencies-in-jigsawjsr-376-java-platform-module-system-ec-member-concerns).
  Despite the controversy, I think it's positive that we now have a module system that is overall
  simpler than the abstraction behemoth that is OSGi.

  Something worth considering is that it can be used with the new `jlink` tool to create minimal
  runtimes that are [much smaller](https://steveperkins.com/using-java-9-modularization-to-ship-zero-dependency-native-apps/)
  than bundling the complete JDK. This is awesome as we can potentially save significant bandwidth
  and storage space when deploying applications to production environments.

- Local variable type inference i.e. the `var` keyword.

  This was also fairly controversial for a lot of people, for different reasons. I personally view
  this as a great step towards modernising Java and moving away from the relentless verbosity that
  it is known for. I only wish that they had gone further and added `val` (to declare an inferred
  variable as `final`) as well... Can't have everything I suppose!

- Improvements to the `Collections` , `Optional` and `CompletableFuture` APIs.

- Allowing private methods on interfaces.

## 2. React Hooks

The new Hooks proposal from the React team caused some heated discussion (resulting in a mammoth
**1300+** comment [RFC](https://github.com/reactjs/rfcs/pull/68)) back in October and edges its way
into second place.

Hooks are first and foremost a way to enable code re-use in a way that was not previously possible
in React. This was motivated by the current status quo:

- Patterns such as render props or higher order components (HOCs) force restructuring of the
  component tree (potentially creating multiple layers of extra components).
- Using class components to re-use code is generally not considered good practice as inheritance
  does not really provide useful benefits over [props and composition](https://reactjs.org/docs/composition-vs-inheritance.html).

These limitations currently mean that re-using code related to state and lifecycles is not simple.
Hooks should allow developers to refactor state and lifecycles (called 'effects' in Hooks) out of
components and into _custom hooks_ which can be imported and applied on-demand. Certain types of
render props/HOCs may become _unnecessary_, resulting in a cleaner component tree.

On the other hand, Hooks have also been somewhat controversial. The main issues generally revolve
around their almost 'magical' implementation, somewhat awkward feeling API and introduction of
additional complexity to React.

Whilst I agree with some of this sentiment, I am mostly looking forward to seeing where Hooks takes
React. From the start, it should enable us to write terser code, but it could possibly unlock new
abstractions and libraries that we can further use to our advantage.

Currently, Hooks are only available as a preview (starting from 16.7.0), but the original RFC has
now been accepted and we should see a lot of new developments soon. Expect the official release of
Hooks sometime later this year.

## 3. Go modules

Back around March, the initial Go [modules proposal](https://go.googlesource.com/proposal/+/master/design/24301-versioned-go.md)
was unveiled and it was certainly an interesting one. Dependency management is still an ongoing
problem throughout the industry so it was cool to see the Go community try to push the envelope with
a pretty innovative solution.

For a bit of backstory, Go has had a bad start with dependency management.

From inception, its native dependency tools would place all dependencies in the same global
directory, defined by a single `GOPATH` environment variable. Unfortunately, whilst this approach
seemed to fit Google's monorepo ecosystem quite well, it was actually terrible for anyone else in
the community. You could very easily expect to get dependency clashes between multiple projects and
consequently this spawned a variety of user-land solutions over the years.

Some solutions included [Dep](https://github.com/golang/dep) and [Glide](https://github.com/Masterminds/glide),
both of which mimicked package managers in the rest of the industry (Maven, NPM, Composer, etc).
At one point Dep could have been deemed sufficient for the community's needs, however, the Go team
decided to take it further with the modules proposal.

### Key proposal features

- Only respecting semver for module releases.

- Only allowing _minimum version_ requirements in `go.mod` files (dependency requirements file).

  This massively simplifies the dependency resolution algorithm and avoids 'dependency hell'
  situations where the algorithm cannot resolve an optimum configuration. There will _always_ be an
  optimum configuration.

  Dependencies themselves will most likely have their own `go.mod` files that also state their own
  minimum versions, however, the top-level `go.mod` (our project/module) will have overriding
  control on the required versions and exclusions.

- Including versions in import declarations i.e. semantic import versions.

  This forces developers to specify what **major** version of a dependency they are using in the
  code itself. For example:

  ```go
  import (
  		moduleV1 "github.com/ntsim/package/module"		// Imports v1
  		moduleV2 "github.com/ntsim/package/module/v2" // Imports v2
  )
  ```

  The argument is that a major release should be considered (rightly) as an entirely new API.
  Consequently, the developer should explicitly declare what major version they are using in the
  code (and not the package manager).

  Initially I was a little put-off by the longer import name, however on balance, I think the
  benefits seem to out-weight the extra verbosity. This alone could potentially be a major win for
  situations where we need to maintain backwards compatibility in our code.

  Unfortunately, it does place the onus on the developer to upgrade version numbers by hand (or
  grep) across the entire codebase, but hopefully this shouldn't be _too_ painful.

Personally I think there are some great ideas here and it will be very interesting to see how things
play out. I would recommend that you check out the full proposal for yourself as it makes for an
interesting read (as far as dependency management goes). Ongoing details about the actual
implementation can also be found [here](https://github.com/golang/go/wiki/Modules).

Hopefully we will see Go modules make it to general availability (expected in Go 1.13) in 2019!

## 4. Kotlin 1.3 and coroutines

The release of Kotlin 1.3 takes fourth place as it marks the long _awaited_ (pun intended) stable
release of [coroutines](https://kotlinlang.org/docs/reference/coroutines-overview.html). Coroutines
introduce a new model for asynchronous, non-blocking programming which should be simpler to work
with than built-in primitives like threads or futures, and user-land reactive libraries such as Rx
or Reactor.

Reactive libraries are still valuable for working with large complex streams of data (in a reactive
manner), however, I think coroutines will replace their usage in a lot of situations.

As well as being simpler to work with, coroutines are considered to be lightweight threads that are
essentially scheduled by the user (and not the OS). This means that they should theoretically be
advantageous in terms of performance (when used correctly).

I'm personally not too familiar with coroutines currently, but I'm intending to get started with
them in 2019!

[Kotlin 1.3](https://kotlinlang.org/docs/reference/whatsnew13.html) itself adds a few token new
features, but nothing too major besides coroutines. The most notable feature is the introduction of
Contracts, allowing for enhanced compiler analysis. This should be particularly useful for providing
even better type inference in places such as smart-casts.

## 5. event-stream incident

In last place, I have made some room for the `event-stream` [incident](https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident)
from the end of November. Whilst it was definitely not a positive event, it is part of a wider issue
that is interesting to discuss.

This incident involved a relatively popular project called `event-stream` being exploited by a rogue
developer to act as a trojan for malicious code. The code in question was eventually found to target
applications that would store Bitcoin wallets and attempt to extract their details discretely.

By itself, this might not have been too bad but unfortunately `event-stream` was transitively
required through a bunch of larger projects making it inevitable that it would eventually be
consumed by its intended target. This target turned out to be [Copay](https://copay.io/), which
confirmed that the code was released with **several** of its versions.

The exploit was first documented in this GitHub [issue](https://github.com/dominictarr/event-stream/issues/116),
but it flew under the radar for a few months beforehand. It eventually conspired that the project
maintainer had promoted the rogue developer to a maintainer on bad judgement.

I think this whole incident highlights the metaphorical tower of Jenga that the JavaScript community
has built for itself over the years. Whilst NPM is an incredible ecosystem with a staggering buffet
of modules, it is increasingly becoming apparent that this has potentially come at the cost of
security and stability - remember `left-pad` back in 2016?

I doubt this will be the last time we see such an attack, and it may not even be the first! Our
dependencies may be hiding even more malicious code beneath the surface, and it is essentially
impossible to audit every single module required by larger projects.

So what can we do about it?

Personally, I would like to see the following things:

- Project maintainers performing proper due-diligence before handing over keys to their project.
  `event-stream` should be used as a textbook example of what happens when you don't.
- Significant improvements to the standard library (in Node and browsers) to avoid needing many of
  the popular lower-level modules. It will be good to see where this TC39 [proposal](https://github.com/tc39/proposal-javascript-standard-library)
  goes.
- Projects consolidating trivial dependencies directly into their own codebases to avoid having
  excessive dependencies on simple modules.
