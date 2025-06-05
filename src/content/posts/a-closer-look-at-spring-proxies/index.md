---
id: '0a2e1507-eda2-4929-aa10-48f457bafe18'
title: 'A closer look at Spring proxies'
description: "How do Spring's proxies and annotation work? In this post we will be taking a deep dive into some finer details and magic. Come join me!"
date: '2018-03-22 01:25:20'
tags:
  - Spring
  - Java
  - Proxies
---

Recently I had the opportunity to look into the basement of Spring and investigate more into how it uses proxies
(everywhere) to achieve some of its 'magic'. In this post I will be sharing some of my findings and exploring the
motivations for proxies and some of the places they are actually being used.

## Why proxy?

Proxies are commonly used when we want to augment the functionality of a **target class**, without directly modifying
the class itself. Typically we wrap the underlying target class with a **proxy class** and expose exactly the same
public interface. Calls to the target class should then be indistinguishable (on the face of it) from calls to the
proxy class. One way of achieving this is to _subclass_ the target class (more on this later).

Once we have this in place, the proxy is free to decorate any calls with additional behaviours. Examples of this include
adding caching, logging or security functionality. Libraries such as Hibernate use proxies for features such as
lazy-loading collections on one-to-many relationships.

The main benefit of proxies is that they allow us to just focus on writing our domain code, as the proxy implementations
can be handled by the library code instead. This allows us to achieve quite a lot with minimal effort and we can avoid
writing a lot of boilerplate code.

In Aspect Orientated Programming (AOP) proxies are actively encouraged. AOP's goal is to separate out cross-cutting
concerns from business logic code and we can see that this is a great fit with proxies. There are also other mechanisms
to write AOP code, but proxies are a sensible and fairly understandable starting point.

## Should we proxy?

Whilst proxies are certainly beneficial, they provide behaviour _implicitly_ rather than _explicitly_. Like most
implicit things, we need to grow an appreciation of how they work behind-the-scenes to not be surprised by them. For
people unfamiliar with proxies, this can lead to the feeling that there is too much 'magic' going on that they cannot
fully understand.

I personally feel that proxies are mostly fine (and often a necessary evil), but in the instances that they do not
behave correctly, they quickly become an inconvenience. In these cases they are a level of indirection that can make
your life harder than necessary. At various times I've felt things would be easier if we were explicit and didn't rely
on proxies.

Regardless of personal opinion, proxies are ubiquitous in Java and are definitely here to stay.

## How Spring uses proxies

Spring has long been a subscriber to AOP and fully leverages proxies out of the box in many of its features and even
within the framework internals. Let's look at some examples of where proxies are used.

### Proxies in bean definitions

In modern Spring applications, annotation-driven autowiring of beans is very common. However, there will usually be
some level of manual configuration of certain beans where we need more fine-grained control over their instantiation.
In annotation-driven configuration code, we might have something that looks like this:

```java
@Configuration
public class AppConfig {

  @Bean
  public ComponentA componentA() {
    return new ComponentA();
  }

  @Bean
  public ComponentB componentB() {
    return new ComponentB(componentA());
  }
}
```

Nothing too controversial here. We should expect an instance of `ComponentA` to be injected into `ComponentB`. On
initial inspection, we can see that the instance of `ComponentA` is directly created by the method call `componentA()`.
However, you might be thinking these things at this point:

- Why do we need to annotate `componentA()` as a `@Bean` if we just directly call the method?
- If we directly call the method, then isn't the instance of `ComponentA` passed to `ComponentB` un-managed by the
  Spring context?

You might be surprised to then find out that the following **does not** complain:

```java
@Bean
public ComponentB componentB() {
  ComponentA a1 = componentA();
  ComponentA a2 = componentA();

  Assert.isTrue(a1 == a2);

  return new ComponentB(a1);
}
```

Both instances of `ComponentA` are surprisingly the same (I have not overridden any `equals` methods) even though the
`componentA()` method is invoked twice. In normal Java code, you _would not_ expect referential equality in this
example.

An initial reaction would be to say that there is clearly some Spring 'magic' that is forcing us to use the same
instance of `ComponentA` every time. That might be a sufficient conclusion for some, but let's scratch this surface a
bit more.

#### Understanding the bean definition magic

As I alluded to earlier, bean definitions are actually an example of proxy usage in Spring. When the application context
is being prepared, multiple `BeanFactoryPostProcessor`s are invoked to perform additional modifications to bean
definitions on the `postProcessBeanFactory` hook.

For a typical annotation-driven Spring application, we will initially only have beans defined via classes annotated
with `@Component`, `@Service`, etc. You might be surprised to find out that `@Configuration` is actually one of these
annotations as well. In fact, it actually inherits from `@Component`:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {}
```

One of the default `BeanFactoryPostProcessor`s is the `ConfigurationClassPostProcessor`. Upon invoking its
`postProcessBeanFactory` hook, it actually wraps any `@Configuration` bean definitions with proxy subclasses using
the `ConfigurationClassEnhancer`.

The proxy created (at least in Spring 5.0) is a CGLIB subclass that registers some interceptors/callbacks which will
only be called if certain conditions are met. The one we are most interested in is the `BeanMethodInterceptor` (nested
in `ConfigurationClassEnhancer`). This interceptor is only called when methods with `@Bean` annotations are invoked.

Now we're getting somewhere!

Any `@Bean` annotated methods act as bean definitions and are registered into the application context by the
`ConfigClassPostProcessor` via another hook - `postProcessBeanDefinitionRegistry`. Once they are registered, these bean
definitions can be accessed via the application context's `BeanFactory` (usually `DefaultListableBeanFactory`).

Going back to our proxy subclass, it is also created with a 'hidden' `$$beanFactory` field which holds a reference to
(you guessed it) the application context's `BeanFactory`. Combining these things, the `BeanMethodInterceptor` is then
able to resolve beans via the `BeanFactory` whenever the `@Bean` annotated methods are called directly.

This gives us the result that we were able to see earlier when we called `componentA()` directly. If you've been able
to keep up with me this far, you should be able to appreciate that this is quite an interesting use of proxies.

It should be noted that this proxying is all manually performed and does not hook into Spring's AOP infrastructure
(unlike other Spring proxies - more on this next).

#### Why is this useful?

The Spring team have clearly gone to a lot of trouble to wrap our configuration code in proxies. But why bother at all?

It appears the main motivation of this proxying is to keep developers working within the application context. By
default, Spring assumes that beans are meant to be consumed with **singleton scope**. This means that there are only
**single instances** of beans managed within the context, and if we try to resolve a bean from it, we will retrieve
the same instance every time.

This is generally the policy many popular dependency injection (DI) containers take towards managing their objects. In
the majority of cases, this is the simplest and most efficient approach and is probably most in-line with developer's
expectations of what DI containers should do.

With our new knowledge on how configuration classes behave, it would be perfectly legitimate (although somewhat odd) to
do something like this:

```java
@Service
public class MyService {

  private ComponentA componentA;

  @Autowired
  public MyService(AppConfig appConfig) {
    componentA = appConfig.componentA();
  }
}
```

The result would be equivalent to autowiring in `componentA` directly:

```java
@Service
public class MyService {

  private ComponentA componentA;

  @Autowired
  public MyService(ComponentA componentA) {
    this.componentA = componentA;
  }
}
```

Proxying our bean definitions means that we respect bean definition scopes, regardless of if we autowire it in or call
it directly from the configuration class.

#### Bonus tangent on DI scopes

An interesting decision made recently by the PHP-DI library's maintainer, was to [scrap scopes](http://php-di.org/doc/scopes.html#scopes)
completely and make everything singleton scope. I think the maintainer's justification is pretty valid as having
non-singleton scopes in a DI container are not very useful in the majority of cases.

Regardless, in Spring you still have the option to use different bean scopes such as prototype, and even more esoteric
ones such as request and session scopes (for when you have some really weird requirements).

### Proxies in Spring Security

Another library that takes advantage of proxies is Spring Security. This library supports method-level security through
annotations such as `@PreAuthorize`. For example:

```java
@PreAuthorize("hasRole('USER')")
public Profile getProfile();
```

Adding these annotations to methods e.g. in your service code, essentially tells Spring that the class requires security
proxying when the application starts. The proxy subclass that is created has an attached `MethodSecurityInterceptor`.
Method invocations on the secured target class are proxied through this interceptor.

The `MethodSecurityInterceptor` coordinates with a bunch of other security components to resolve the security rules,
before deciding if the method invocation should be allowed. In the event that authentication/authorization fails, an
`AccessDeniedException` is thrown and the client code will not be able to invoke the secured target method.

This allows for quite fine-grained control over the security logic on each individual service method. Additionally, this
approach allows us to separate out the security logic (a cross-cutting concern) from the service-level business logic
in a clean way.

Sounds awfully like AOP to me!

#### How secured proxies are created

Unlike the bean definition proxying mechanism described earlier, the security proxying hooks into Spring's AOP
infrastructure. This could be because the initial application context needs to be bootstrapped first before the AOP
infrastructure is available for use (needs confirmation).

To get started, we need to enable this security via the `@EnableGlobalMethodSecurity` annotation, which can be attached
to any configuration class. This causes the `GlobalMethodSecuritySelector` to be applied, as
`@EnableGlobalMethodSecurity` inherits an `@Import` annotation specifying that this configuration should be loaded.

`GlobalMethodSecuritySelector` then registers `InfrastructureAdvisorAutoProxyCreator` and
`MethodSecurityMetadataSourceAdvisor` in the application context. The `InfrastructureAdvisorAutoProxyCreator` performs
the heavy lifting here and matches advisors to beans that can be advised (based on their pointcuts). If a match is
found, it automatically creates the required CGLIB subclass proxy and adds the required interceptor/callbacks (in AOP
terminology this an advisor's 'advice').

`MethodSecurityMetadataSourceAdvisor` is one such advisor and is responsible for adding the `MethodSecurityInterceptor`
to any proxy subclasses that require securing.

#### More on Spring's AOP infrastructure

If you have used Spring AOP before, then you might be most interested in `InfrastructureAdvisorAutoProxyCreator` - an
example of an [auto-proxy creator](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/core.html#aop-autoproxy).
These play a crucial role in Spring applications as they enable AOP proxies to be **automatically** created for any
required beans in the application context. In essence, they _are_ Spring's AOP infrastructure.

Auto-proxy creators are actually `BeanPostProcessor`s under-the-hood and perform the proxying during bean instantiation
hooks such as `postProcessBeforeInstantiation` and `postProcessAfterInitialization`. There are a number of default
auto-proxy creators in the application context and they are largely the culprits for most of Spring's AOP 'magic'.

Auto-proxy creators are typically configured to work with certain types of advisors and/or target beans. For example,
`InfrastructureAdvisorAutoProxyCreator` is only concerned with advisor beans that play an 'infrastructure role' (i.e.
an internal framework role). `MethodSecurityMetadataSourceAdvisor` is one such infrastructure advisor bean.

## Conclusion

We've gone through quite a lot here, but to recap:

- Proxies are an AOP mechanism and allow us to separate out cross-cutting concerns such as security, logging, caching,
  etc. They allow libraries do more work for us, and can remove boilerplate code.
- In Spring, bean definitions themselves are proxied. This is to respect bean scopes regardless of how beans are
  retrieved - either directly from the context or via a DI method.
- Spring Security leverages the internal AOP infrastructure to implement its method-level security features. The AOP
  infrastructure plays a crucial role in providing lots of other implicit functionality in Spring as well.

Hopefully you've grown more of an appreciation for the underlying scaffolding that holds up the Spring Framework and
(usually) makes our lives easier.

On a final note, I hope to write a follow-up post on some of the more advanced intricacies of generating proxies. It
should be less Spring orientated!
