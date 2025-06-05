---
id: 'b8fdb269-978c-4660-97a7-37fff19a59b7'
title: 'New interesting features in Spring 5 (using Kotlin)'
description: "Spring Framework 5 makes working with Kotlin even better than ever. Let's take a look at the most interesting new features!"
date: '2018-04-29 13:26:11'
tags:
  - Spring
  - Kotlin
  - Reactor
---

Spring Boot 2 was released at the end of last month and packs some new shiny toys for us to play around with.
Importantly, it brings the massive Spring Framework 5 update (released back in September), so I decided to take
Spring Boot 2 out for a quick spin to check out what's new.

I will mostly be looking at this from a Kotlin perspective as the Spring team have thrown in some additional
language-specific features as well. It's great to see such a level of support for the language, and I hope this
incentivises more Java developers to start considering Kotlin!

## Null safe API

One of the main selling points of Kotlin is the way nullability is handled. It uses both nullable and non-nullable types
to enforce compile time null safety (making it harder for bugs to creep in). When working with Kotlin code we can make
the most out of this feature, but unfortunately it isn't so simple when consuming external Java libraries.

The main issue is that types returned from Java code still have to be treated as potentially nullable. For the sake of
interop, the Kotlin compiler converts Java types into **platform types**. These types have relaxed null-checks so that
we don't have to deal with nullable types everywhere. Unfortunately, as a consequence of this we also lose the compile
time null safety we would normally get with pure Kotlin types. The following can be observed with our Kotlin type
definitions (where `item` is a `String` platform type):

```kotlin
val nullable: String? = item // Allowed at compile time and runtime
val notNull: String = item // Allowed at compile time, may exception at runtime
```

This is quite frustrating as we have to potentially suffer a degraded language experience when working with Java code!
To alleviate this issue, Java developers can add nullability annotations to their code to assist in interop with Kotlin
consumers. Nullability annotations can be understood by the Kotlin compiler to assist in providing more accurate type
information than the default platform types.

The Spring team has thankfully been very diligent and went to the trouble of adding these nullability annotations across
the entire framework API. This means that Spring code is treated with the same compile time null safety guarantees that
we would normally get in Kotlin. Sweet!

### What do nullability annotations look like?

The Kotlin compiler is able to understand a variety of nullability annotations, such as [JSR-305](https://jcp.org/en/jsr/detail?id=305)
annotations. Unfortunately JSR-305 actually has a 'dormant' status (and has been for many years) due to the various
groups involved being unable to form a consensus. This is a shame as the proposal is clearly useful and many projects
actually consume the existing proposed annotations. There are several other nullability annotation packages that can
also be used from libraries such as FindBugs and Checker Framework.

For the Spring team, they opted for JSR-305 annotations and created new `@NonNullApi` and `@NonNullFields` annotations
that look like this:

```java
@Target(ElementType.PACKAGE)
@Nonnull
@TypeQualifierDefault({ElementType.METHOD, ElementType.PARAMETER})
public @interface NonNullApi {}

@Target(ElementType.PACKAGE)
@Nonnull
@TypeQualifierDefault(ElementType.FIELD)
public @interface NonNullFields {}
```

By applying them at the package level, they applied `@Nonnull` (by default) to most methods, parameters and fields
across the entire Spring API surface. For instances where they needed nullable types or more granular control, they have
used additional `@NonNull` and `@Nullable` annotations. Pretty clever!

For reference check out the commit where they finalised these changes [here](https://github.com/spring-projects/spring-framework/commit/1bc93e3d0f2bd3ffc6b48662073a427c31088b3d).

## Functional bean registration

Spring Framework 5 importantly sets its minimum version at Java 8, meaning that it can leverage some of the latest
language features in its API. The main one to note is the addition of lambdas, particularly in areas where we would
previously have needed annotations. An interesting place where they have done this is in bean definitions.

In Spring 4 we would have to use the typical `@Bean` definitions in some configuration like the following:

```kotlin
@Configuration
class AppConfig {
  @Bean
  fun foo() = Foo()
}
```

With Spring 5, we have some additional options for how we define our beans. If we can get a hold of a
`GenericApplicationContext`, we can directly define our beans there. One way is to use an `ApplicationContextInitializer`.
In Java this looks like the following:

```java
public class BeanInitializer implements ApplicationContextInitializer<GenericApplicationContext> {

  @Override
  public void initialize(GenericApplicationContext applicationContext) {
    applicationContext.registerBean(
      Foo.class,
      () -> new Foo(),
      beanDef -> {
        // This is a customizer lambda where we
        // can perform some additional configuration
        // of the bean definition.
        beanDef.setAutowireCandidate(false);
      }
    );
  }
}

```

Interestingly, we can now programmatically configure the bean definition via 'customizer' lambdas that are passed in as
varargs. Previously in Spring 4, such configuration would have to be passed in by creating a new `BeanDefinition`
instance and configuring it before passing it in as a parameter. I think this cleans things up nicely, although I'm not
sure about the value of having them as varargs.

Unfortunately, customizer lambdas are more awkward to use in Kotlin due to the way that SAM conversions are handled.
We actually have to explicitly define the type of the SAM interfaces that we are using as the compiler cannot infer the
type directly.

```kotlin
class BeanDefinitions : ApplicationContextInitializer<GenericApplicationContext> {

  override fun initialize(applicationContext: GenericApplicationContext) {
    applicationContext.registerBean(
      Foo::class.java,
      Supplier { Foo() },
      BeanDefinitionCustomizer { beanDef ->
        beanDef.setAutowireCandidate(false)
      }
    )
  }
}
```

Unfortunately, this is even more verbose than the Java equivalent! Thankfully, the Spring team have provided some
additional Kotlin-specific functionality to reduce this verbosity via extension functions.

```kotlin
import org.springframework.context.support.registerBean

class BeanDefinitions : ApplicationContextInitializer<GenericApplicationContext> {

  override fun initialize(applicationContext: GenericApplicationContext) {
    applicationContext.registerBean(BeanDefinitionCustomizer { beanDef ->
      beanDef.autowireCandidate = false
    }) {
      Foo()
    }
  }
}
```

This is not as pretty as it could be, but generally we don't need to use bean definition customizers much anyway.
Normally we would just need one of the following:

```kotlin
applicationContext.registerBean { Foo() }
applicationContext.registerBean<Foo>() // Use reified type to define bean instead
```

### Using the new bean DSL

As well as the aforementioned bean registration methods, Spring 5 introduces a new Kotlin-specific bean DSL that can be
used to define our beans. This is much nicer as it improves the API considerably and removes any verbosity. For the
customizer lambda example, we can simplify it to the following:

```kotlin
val beans = beans {
  bean(isAutowireCandidate = false) { Foo() }
}
```

Under-the-hood, the bean DSL actually uses the new lambda-based bean registration methods seen earlier, however, you
would probably agree that this is much easier to use!

How do we go about adding these beans definitions to the application context?

`BeanDefinitionDsl` actually inherits from `ApplicationContextInitializer` and consequently can be directly applied upon
application start up like any other initializer. Spring Boot provides us with a new `runApplication` method for Kotlin,
which we can use to start up our application with our bean DSL like so:

```kotlin
val beans = beans {
  // Some bean definitions...
}

fun main(args: Array<String>) {
  runApplication<Application>(*args) {
    addInitializers(beans)
  }
}
```

The bean DSL makes it possible to completely avoid any annotation-driven configuration classes. This is probably not a
major improvement for everyone, but if you're looking to create more declarative and transparent configurations (with
less 'magic'), this could be a very welcome change.

## Getting reactive with WebFlux

One of the major features of Spring 5 is the new WebFlux framework that allows developers to create reactive,
non-blocking Spring applications. The intention of this is to maximize performance from as few threads as possible,
and allow them to manage high traffic situations better.

A prerequisite to this is that we have to adopt asynchronous, event-driven application architectures using streams and
functional programming paradigms. One of the key components we have to get comfortable with is [Reactor](https://projectreactor.io/).
This library provides the reactive stream APIs that glue the WebFlux framework together.

Sounds like fun! What does it look like?

WebFlux supports the normal annotation-driven controllers that we are used to from the servlet world. A simple example
looks like this:

```kotlin
@RestController
class FooController {

  // Returns {"value": "a"}
  @GetMapping("/foo")
  fun getFoo(): Mono<Foo> {
    return Foo("a").toMono()
  }

  // Returns [{"value": "A"}, {"value": "B"}]
  @GetMapping("/foos")
  fun getFoos(): Flux<Foo> {
    return Flux.just(Foo("a"), Foo("b"))
      .map { Foo(it.value.toUpperCase()) }
  }
}
```

You will probably be wondering why we are wrapping things with either `Mono` or `Flux`. These are actually primitives
from the Reactor library. `Mono` represents a _single_ asynchronous result and `Flux` represents a stream of _1 to n_
asynchronous results. These are quite easy conceptually to grasp, however they have quite a large API surface to learn.
In simplistic terms, they mostly function like normal streams, and we are able to perform common operations like
`map` and `filter`. I will discuss Reactor in more depth in a later post.

Initially, I got quite excited as I hoped we'd be able to get non-blocking behaviour with minimal effort. Unfortunately,
in most scenarios we also need to interact with a database; so to make the most of the non-blocking web layer, we also
need a non-blocking persistence layer. Unfortunately, there are currently only a handful of Spring Data drivers that
support this - specifically for Cassandra, Redis and MongoDB.

SQL database abstractions do not exist yet as JDBC is implemented in a blocking fashion (part of this is due to the way
that transaction management is handled under-the-hood). Unfortunately, this means that if you are using a SQL database,
you won't really see much benefit from using WebFlux. Hopefully we'll get some non-blocking JDBC abstraction in the near
future.

If you do happen to be using a supported database, connecting the pipes in the web and persistence layers should be
trivial. Building on our example from earlier, it might look like the following:

```kotlin
@RestController
class FooController(private val fooRepository: FooRepository) {

  @GetMapping("/foo/{id}")
  fun getFoo(@PathVariable id: Mono<Long>): Mono<Foo> {
    return fooRepository.findOne(id)
  }

  @GetMapping("/foos")
  fun getFoos(): Flux<Foo> {
    return fooRepository.findAll()
      .map { Foo(it.value.toUpperCase()) }
  }
}
```

This is not too far from what Spring code would normally look like! It's great that the Spring team have clearly gone to
some effort to make the transition to WebFlux as painless as possible. I look forward to getting to grips with WebFlux
more in the future!

## Declarative routing

Another area that gets the functional programming treatment in Spring 5 is routing, specifically for WebFlux
applications. There is a new routes DSL that can be used instead of the normal annotation-driven controllers. A simple
example that provides the same functionality as the `FooController` from earlier would look like this:

```kotlin
class Routes(private val fooRouteHandler: FooRouteHandler) {

  fun router(): RouterFunction<ServerResponse> = router {
    accept(MediaType.APPLICATION_JSON).nest {
      GET("/foo/{id}", fooRouteHandler::getFoo)
      GET("/foos", fooRouteHandler::getFoos)
    }
  }
}
```

The syntax is pretty simple and makes it quite easy to see what is going on. A nice advantage of having a declarative
DSL is that we can also programmatically define our routes by adding in conditional logic when required. It's also
available as a Java DSL (unlike the Kotlin bean DSL from earlier).

You might also be wondering what `FooRouteHandler` is. It is simply a component with methods that accept a
`ServerRequest` and return a `Mono<ServerResponse>`. There is no need to explicitly call it a 'controller' (you still
can), but 'handler' seems to be a more popular convention.

```kotlin
@Component
class FooRouteHandler {

  fun getFoo(request: ServerRequest): Mono<ServerResponse> {
    return ok().body(Foo("a").toMono())
  }

  fun getFoos(request: ServerRequest): Mono<ServerResponse> {
    return ok().body(
      Flux.just(Foo("a"), Foo("b"))
        .map { Foo(it.something.toUpperCase()) }
    )
  }
}
```

Overall, I think the DSL is quite refreshing (for Spring) as we explicitly declare all the available routes in one
place. If you've ever worked with microframeworks in NodeJS or PHP, most opt to use this kind of approach. I have
personally never been completely sold on using annotations for routes as they seem to be harder to maintain as you can
end up with many routes defined in many different files across your application.

To get started using the DSL we have to register the routes by defining a web handler bean. Using the bean DSL from
earlier (in the functional spirit of things), it would look like this:

```kotlin
val beans = beans {
  // Routes is our routing class from earlier
  bean<Routes>()

  // Register the web handler, providing it with the routes
  bean(WebHttpHandlerBuilder.WEB_HANDLER_BEAN_NAME) {
    RouterFunctions.toWebHandler(ref<Routes>().router())
  }
}
```

Alternatively, we can just define a configuration class with annotations like we would normally. Spring can
automatically pick up any beans that return a `RouterFunction` and register its routes:

```kotlin
@Configuration
class Routes {

  @Bean
  fun router(): RouterFunction<ServerResponse> = router {
    accept(MediaType.APPLICATION_JSON).nest {
      GET("/foo/{id}", fooRouteHandler::getFoo)
      GET("/foos", fooRouteHandler::getFoos)
    }
  }
}
```

### Teething problems

Working with the new routes DSL was not without its pain points.

When I first started trying it out, the routes would not register with Spring, causing a good few hours of frustration!
I think the most frustrating problems are always related to configuration... Upon reading the documentation further,
it turned out that the routes DSL only works for WebFlux applications. Additionally, when using Spring Boot, the
application would be configured to use servlets rather than WebFlux if `spring-boot-starter-web` was on the classpath
(despite `spring-boot-starter-webflux` also being on the classpath).

To get around this, we can have to explicitly tell Spring to configure itself to be a WebFlux application upon startup:

```kotlin
fun main(args: Array<String>) {
  runApplication<Application>(*args) {
    webApplicationType = WebApplicationType.REACTIVE
  }
}
```

## Conclusion

Spring Framework 5 brings some awesome new features to the table that take advantage of the power of Java 8+ and Kotlin.
The most notable features include:

- Nullability annotations added across the Spring API to provide true compile time null safety for Kotlin users.

- Bean registration using lambdas provide the option to move away from annotation-driven configuration classes. Kotlin
  users also have the option to use the new bean DSL, making this even easier.
- The new WebFlux framework provides non-blocking behaviour to Spring applications. This allows us to maximize thread
  performance and embrace reactive programming using Reactor.
- A routes DSL using lambdas to declare routes in a functional style. This allows us to make our routing configurations
  more transparent. We can have all our routes in the same place!

Most of these changes are focused on promoting more of a functional programming (FP) style in Spring. This is great as
FP concepts are becoming more and more popular in mainstream languages, at both the language and user level. I think
this mindshift is very welcome as there are some great concepts that can add a lot of value to any developer's toolkit.
I certainly welcome the chance to use more FP in my day-to-day coding!

WebFlux is also a great new development as it makes reactive programming way more accessible. I will be interested to
see how it pans out in the future as it becomes more mainstream. It certainly seems like an attractive option for
creating fast, scalable microservices using NoSQL databases.

The improvements to the Kotlin developer experience are also fantastic. I personally really enjoy using Kotlin, so to
also see this level of confidence in it from the Spring team is inspiring. Perhaps this will be able to win over more
Java developers!

Overall, I think this is an exciting update to Spring and I personally can't wait to use it more in real projects over
the coming years.
