---
id: '7d003151-395f-4f45-848a-8d587aa46334'
title: 'My optimum Spring Boot development environment'
description: 'Some simple recommendations for a Spring Boot development environment that can increase your productivity and reduce your time waiting for builds.'
date: '2018-10-21 22:45:39'
tags:
  - Spring
  - Java
  - Docker
---

During my last few years of working with Java, and more specifically with Spring Boot, I have worked
with several types of development environment. Whilst all of these were functional and worked, some
did not really allow developers to quickly iterate in the way I would have liked. I originally
started out writing PHP and Javascript, so it was initially quite a shock to encounter long build
times in my Java projects!

In this post I will review some of the different types of development environments and explain some
of their pros and cons. I will finally offer my opinion on the best solution for being a productive
(and happy) Java developer.

## The traditional build cycle

We'll start off with the simplest development environment. This is where you write some code then
manually trigger a build, be it through the IDE or the build tool CLI. The application will need to
be completely redeployed with the newly built artifacts.

Unfortunately, whilst this is a fairly common approach (not just in Java), it leaves developers at
the mercy of the build. As any project grows, the complexity increases and build times can gradually
creep up to unacceptable levels.

Slow build times make working on any project a chore. One project I have worked on had build times
that could range between 5-20 minutes! Industrious developers may find ways to get around such
bottlenecks, but they can't be avoided forever. As long as the build is not optimized, developer
productivity across the entire team will suffer.

In my opinion slow build times should really be viewed as major bugs. Unfortunately, there is the
temptation to classify them as technical debt, but this does not really convey how detrimental a slow
build can be. From a pure metrics perspective, there are potentially days worth of developer hours
being wasted every week across the _entire_ team.

On the individual level, this type of sluggish development can create negative sentiment towards the
project and burn developers out. Bad project managers may feel that things aren't being delivered
fast enough (not unduly), so they heap more pressure on the developers, which in turn creates more
resentment towards the project - a vicious cycle.

Ultimately, using the traditional build cycle with slow build times is an anathema to rapid
development. It surprises me how this problem is usually grossly underestimated by developers and
project managers alike. It is also impressive that teams put up with this type of development
environment for months/years on end and still manage to roll out features. Unfortunately I also
can't help but think about how much faster things could be delivered with some better tooling.

## Hot restarts with Spring Boot developer tools

One of the more interesting features of Spring Boot is its developer tools which provide the ability
to hot restart the application with only new code changes and without having to reload the entire
classpath. Only classes that are expected to change are reloaded, whilst classes that should not
change, such as third-party libraries, are not. This consequently makes hot restarts much quicker
than cold starts.

Turning this feature on is quite simple and just requires the following:

1. Add the `spring-boot-devtools` dependency into the build (see [documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-devtools.html#using-boot-devtools)).
2. Run the application in a non-production mode using one of the build tool tasks e.g.
   `gradle bootRun`.
3. Trigger hot restarts by rebuilding the project. For a typical workflow, this will usually be
   triggered through your IDE.

You can take this a little bit further by using the [LiveReload](http://livereload.com/) browser
plugin to refresh the browser whenever a hot restart takes place!

I personally think this is already quite a nice setup as it enables a developer to make changes and
nearly immediately see the results reflected in the browser. This is very close to the development
experience offered by PHP and Javascript and makes the traditional build cycle seem like a distant
nightmare.

Surprisingly I don't see many developers taking advantage of this feature when they have the
opportunity to do so. Perhaps people aren't aware of it, or inertia means that people are happy to
continue developing how they've always developed. There can potentially be other environmental
constraints preventing developers from doing so, such as Docker.

### What about Docker?

In today's world of microservices and Docker, it is common to see Java applications being deployed
via Docker containers. Unfortunately, this also adds a new set of requirements for our development
environment and how we should work.

A common approach is to package up our Spring Boot application into a far JAR, copy it into the
Docker container and then run it. When running applications in Docker containers like this, they can
be considered to be running on a completely different machine to the host machine. This affords a
level of encapsulation that is one of Docker's key features, but unfortunately it can cause issues
when we also want to use Spring Boot developer tools.

## Hot restarts with Docker containers

Spring Boot developer tools relies on the ability to detect local file changes to trigger hot
restarts. Consequently, a Spring Boot application that has been packaged up and is running in a
Docker container will not be able to detect any new code changes made on the host machine after it
has been started.

Without the ability to hot restart, this type of development environment is actually just the
traditional build cycle in sheep's clothing! Can we do anything about it?

### Connecting remotely

One solution to this limitation is to connect to the application [remotely](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-devtools.html#_running_the_remote_client_application)
via the IDE GUI. Once connected, hot restarts can be triggered in a similar manner to when they are
performed locally.

Unfortunately, this remote mechanism is not perfect. It is usually much slower than the local
equivalent and my past experience with it has generally been quite flaky. The application would
often crash after a certain number of restarts, making it quite difficult to work consistently with
this workflow.

### Using volume mappings

An alternative solution that I investigated a year or two ago was to leverage Docker's volume
mappings instead. The idea was to simply map the project directory into the Docker container and
then run the application. This would allow the application to detect file changes made on the host
machine, as if it were running locally (and not in a Docker container).

This has the potential advantage of being faster and less flaky than the remote alternative. Check
out the proof-of-concept [here](https://github.com/ntsim/hot-spring-docker) for a more complete
example. Notably, I have recently revisited it and addressed some of the issues with the original
naïve implementation (specifically fixing issues with conflicting file permissions and Gradle
caching). It's a good starting point if you're interested in getting the most out of hot restarts in
a Docker environment.

## The 'best' environment

From my personal experience, the best development environment is one where we can use hot restarts
whilst running our Spring Boot applications locally. Stuffing applications into Docker containers
(at least during development) creates needless complication and can actually make our lives harder.
Contrary to what some people may think, not having everything inside of a Docker environment is
okay!

Java applications in fat JAR format are essentially containers themselves and can run with minimal
infrastructure requirements on multiple platforms. You just need to have a Java runtime installed on
your machine! Docker doesn't really add much value here.

Where Docker does add value is in packaging up other large infrastructure components that we have to
integrate with e.g. databases, search engines, microservices written by other teams, etc.
Integrating between our locally running applications and applications running in Docker containers
is not particularly difficult and usually just requires some networking configuration tweaks.

The icing on the cake for this kind of development environment is the ability to run Spring Boot
applications via IntelliJ's ['Run Dashboard'](https://blog.jetbrains.com/idea/2017/05/intellij-idea-2017-2-eap-run-dashboard-for-spring-boot/).
This dashboard makes it super easy to start, stop and debug multiple applications with only a few
clicks in the GUI. This is very useful when working with a bunch of microservices.

### On using Docker for production

Whilst I don't advocate placing applications in Docker containers during development, the same can't
be said when deploying to production. Deploying applications within Docker containers makes much
more sense in production, especially if you're using an orchestration tool such as Kubernetes.

Remember that production does not have the same set of requirements as development. The development
environment should be geared towards improving the developer experience and allowing for quick
iteration, whilst production should be aimed at providing a stable and consistent environment to
deploy to.

### What if I'm not using Spring Boot?

It should be noted that hot restarts are specific to Spring Boot applications. For non-Spring Boot
applications you will need to rely on something like [JRebel](https://zeroturnaround.com/software/jrebel/).
I have heard some good things about JRebel, but unfortunately it has a license cost (which seems a
bit on the high side). An open source alternative is [Hotswap Agent](https://github.com/HotswapProjects/HotswapAgent).
Both of these projects seem to perform low-level bytecode wizardry to hot reload new code changes
into the running application.
