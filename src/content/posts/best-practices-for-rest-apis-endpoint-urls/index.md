---
id: 'aeafdc0b-f8de-4345-960a-4540b1b921f6'
title: 'Best practices for REST APIs: How to write endpoint URLs'
description: 'What makes a good REST API endpoint URL? Here are my recommendations for writing URLs that are maintainable and intuitive.'
date: '2020-11-29 23:30:00'
tags:
  - REST
  - APIs
---

Over the years, I've found that choosing the right URLs for your REST API endpoints can be deceptively difficult. REST doesn't prescribe any hard and fast rules for URLs and there certainly isn't a one-size fits all solution. With that said, I'd like to share some tips that I've found useful when building APIs.

## Resources not actions

When deciding on your endpoint URLs, it can often be tempting to use URLs describe what they _do_ (actions), rather than what they _are_ (resources). There's a subtle difference, but it has a huge impact on the result. Let's see this in practice with a simple example endpoint that fetches a `user`. We could write the URL in the following ways:

1. An endpoint that describes what it does: `GET /api/users/getById/{id}`
2. An endpoint that describes what it is: `GET /api/users/{id}`

The second one should be preferable as it's shorter and only uses the `GET` HTTP verb to express what it's doing. The first endpoint has unnecessary redundancy as it states its intentions twice - once in the HTTP verb and once in the URL. This isn't good RESTful design, in fact, it's not even REST!

The first endpoint is a remote procedure call, or RPC, endpoint and this type of endpoint typically maps directly to a backend function (perhaps with a name like `getById` from our example). There isn't too much thought about the semantics and HTTP is just considering a mechanism by which to call the code. We could just as easily implement our example with a `POST` and it would still be considered a valid RPC endpoint.

### REST vs RPC

Ignoring the fact that you're reading a post about recommendations for REST APIs (that means you're already bought in, right?), let's briefly think more about why we should prefer REST over RPC. Feel free to skip this section if the difference is obvious to you already.

RPC is perhaps the simplest way for developers to build APIs as there aren't really any rules to follow. For the most part, you just need to expose your function as a HTTP endpoint and away you go. This style is excellent for meeting ad-hoc requirements in a timely fashion, but comes with massive trade-offs in terms of maintainability, design and coupling.

Building an API like this is more likely to lead to snowflake-like design where there is little to no coherence. Duplication will be everywhere, in both endpoints and in code. This leads to a potentially massive surface area of endpoints that must be maintained in fear of breaking existing contracts with consumers.

Consumers can find themselves having to learn many endpoints to integrate with this API and may be constantly surprised by their different behaviours. In the worst cases, they may find themselves often on the sharp end of accidental breaking changes.

On the other hand, REST encourages us to model our API in terms of resources and to leverage the full power of HTTP to express things we can do with those resources. Resources are analogous to database tables and allow consumers to use your API like they would a database schema. When done correctly, this enables consumers to more easily use your API and typically results in a more coherent design with fewer surprises.

For further reading, Google's Martin Nally discusses this topic in much greater detail in his post on [REST vs RPC](https://cloud.google.com/blog/products/application-development/rest-vs-rpc-what-problems-are-you-trying-to-solve-with-your-apis).

### Making the most out of resources

Hopefully, our earlier example should have highlighted the difference between REST and RPC endpoints. What about more complicated cases where it may not be as straightforward?

Consider the following example - we need an endpoint where you must approve or decline an `application` with a reason. How would you model this in a REST API? Perhaps you would use two separate endpoints where we submit an 'approve' or 'decline' sub-resource:

```
POST /api/applications/{id}/approve
{ "reason": "Some reason to approve the application" }

POST /api/applications/{id}/decline
{ "reason": "Some reason to decline the application" }
```

This feels like REST on the surface, but unfortunately, it's still RPC. One of the giveaways is that we have an unnecessary level of redundancy by having multiple endpoints that do the same 'thing'. In this case, they are both concerned with changing the status of the application.

So how can we make this RESTful? Let's try use a `status` sub-resource instead:

```
PATCH /api/applications/{id}/status
{ "status": "APPROVED", "reason": "Some reason to approve the application" }
```

Much cleaner. It removes the redundancy of multiple endpoints and allows us to easily add more status types in the future.

We could take this further by removing the `status` sub-resource completely and just update the `application` itself with something like:

```
PATCH /api/applications/{id}
{ "status": "DECLINED", "status_reason": "Some reason to reject the application" }
```

The correct approach is up for debate as it will likely depend on your domain model. For example, it may make more sense to have a sub-resource if you persist the statuses in an `application_status` table. Even if you don't have an additional table, you may just prefer to use the sub-resource if it helps to organise your backend code.

Regardless of what approach you take, you should always look out for opportunities to abstract over redundant endpoints and resources. An API designed in this way will be smaller, easier to maintain and be less likely to confuse consumers over its usage.

### When actions may be better

Despite my earlier advice surrounding REST vs RPC, it can sometimes be difficult to model your endpoints in a truly RESTful fashion. For example, you might need endpoints to perform specific maintenance tasks, such as cleanup or migrations. These types of endpoints are naturally well suited to the RPC style.

In some cases, you might need to perform an aggregate task that works with a bunch of different resources, or the task doesn't fit well with any of the HTTP verbs. Pure REST semantics may not correctly model your APIs intent and trying to shoehorn it in may cause more harm than good in the long term.

Whilst we should strive to achieve consistent RESTful API design, it's also important to be pragmatic. Certain types of domains will be simpler to represent as actions than resources, and it's okay to do so now and then!

## Queries not endpoints

Continuing with our earlier example, we now want to list applications based on their `status`. Perhaps, we have two pages that show the accepted and rejected applications, respectively. How would you achieve this?

A simple approach would be to create some `GET` endpoints like the following:

```
GET /api/applications/approved
[
  { "status": "APPROVED", "status_reason": "Some reason to approve the application" }
]

GET /api/applications/declined
[
  { "status": "DECLINED", "status_reason": "Some reason to reject the application" }
]
```

Hopefully, it should be obvious that this is the same kind of endpoint redundancy that we've discussed previously. As we're working with the same resource, we should be able to abstract multiple endpoints into one. We can simply filter our list of applications based on a `status` query parameter:

```
GET /api/applications?status=APPROVED
GET /api/applications?status=DECLINED
```

This approach is great as we don't need to add more endpoints whenever there is a new status type. More importantly, it opens the door to having query parameters for other properties such as `id`, `name`, `created_at`, etc. This would quickly become unmanageable if you decided to build new endpoints each time there were new filtering requirements!

One of the most common use-cases for query parameters is for listing items. It's usually a good idea to paginate these lists so that consumers don't end up receiving hundreds or even thousands of resources. Pagination can be implemented in lots of different ways, but some variation of `page` and `page_size` query parameters is usually a good place to start.

### Building complex queries

As you develop your REST API further, you'll usually end up in a situation where you need to provide more complex querying options. Rather than build new endpoints for specific queries, good APIs should look to provide richer query parameter options. This kind of flexibility allows consumers to tailor queries for their specific use-case in an ad-hoc fashion.

For example, it's often useful to filter results using comparison operators such as `!=`, `<` and `>`. There's plenty of ways to express these, but a good starting point is to use bracket notation like `created_at[ne]`, `created_at[gt]`, `created_at[lt]`. You could also try building the expression directly into the parameter's value itself e.g. `created_at=ne:2020-11-29`.

Frameworks and libraries can help you implement all sorts of operators via query parameters. One particularly powerful framework I've used for this purpose was FeathersJS (check out their [query parameter DSL](https://docs.feathersjs.com/api/databases/querying.html#querying)). Whilst the framework itself had its rough edges, it really showcased to me how powerful you could make query parameters.

## Consistency is key

When developing an API as a team, it's usually unsurprising that inconsistencies appear over time. Some endpoints might be pluralised, `PUT` may be used when `PATCH` is more appropriate, and RPC might be used for whole sections of the API.

Unfortunately, as there are no hard rules on what constitutes as a good REST API, many important choices have to be made at the individual level. People's understandings of REST can differ wildly and this can often lead to chaotic results.

An API with too many inconsistencies becomes hard to work with in the long-term. These inconsistencies eventually get noticed and become technical debt that cannot be easily fixed, especially if your API is public and widely used.

Additionally, if you want to help consumers integrate as easily as possible with your API, consistency allows them to more quickly understand how your API works. For example, it would be a horribly frustrating experience if pagination parameters differed per endpoint!

Try to agree on your API conventions, as a team, at the start of the project. Enforce them in code reviews, think carefully about how your API looks to consumers and try to regularly audit all of your endpoints (use auto-generated API docs to help with this).

## Final thoughts

When thinking about your REST API's endpoint URLs, try to keep the following in mind:

- What is the resource at its core?
- Do you need a new resource, or is there already one that fulfils the same responsibility?
- Can query parameters achieve the same thing on an existing endpoint?

By thinking about these things, we can reduce the amount of duplication in our API, make it easier to maintain and try to provide flexibility for our consumers. We should try to avoid the temptation of RPC-like endpoints as these can result in a much larger API that is less flexible and more prone to breaking.
