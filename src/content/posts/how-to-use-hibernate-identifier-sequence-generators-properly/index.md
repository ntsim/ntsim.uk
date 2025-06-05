---
id: 'a7dc1b69-23dd-4dfc-8b62-f9568a9e3677'
title: 'How to use Hibernate identifier sequence generators properly'
description: "Do you find Hibernate identifier sequence generators confusing? Here's my definitive answer on how to use them properly."
date: '2018-05-06 12:56:00'
tags:
  - Hibernate
  - SQL
  - Java
---

Recently I was helping to build out a domain model using JPA/Hibernate when we came upon the interesting topic of using
sequences for identifier generation. This was not my first encounter with this particular topic, but like with many of
Hibernate's bevy of features, I had forgotten the intricacies and needed some refreshing.

There are quite a few posts and StackOverflow questions about this topic, but I felt they were generally lacking a
single definitive answer covering its full breadth. Time to lay it to rest once and for all.

## More than one way to skin an ID

When using simpler ORMs, the question of identifier generation is typically left to the database itself to handle.
Persisted models will automatically be assigned an auto-incremented ID upon insertion into the database. In other words,
insert statements do not contain IDs.

```sql
INSERT INTO user (name) VALUES ('Joe Bloggs');
```

Hibernate is quite an advanced and complex ORM and actually offers a
[few ways](http://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html#identifiers-generators)
to generate identifiers. To achieve the above database-driven approach, we simply use the `IDENTITY` generation
strategy. Typically this would look like the following:

```java
@Entity
public class User {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
}
```

This is a nice simple approach, and if we're happy enough to not dig any deeper the story ends there. Unfortunately,
simpler does not always mean better. Hibernate actually advises **against** this strategy as there are implications with
using it. From the Hibernate 5.2 documentation:

> It is important to realize that this imposes a runtime behavior where the entity row **must** be physically inserted
> prior to the identifier value being known. This can mess up extended persistence contexts (conversations). Because of
> the runtime imposition/inconsistency Hibernate suggest other forms of identifier value generation be used.

> There is yet another important runtime impact of choosing IDENTITY generation: Hibernate will not be able to JDBC
> batching for inserts of the entities that use IDENTITY generation. The importance of this depends on the application
> specific use cases. If the application is not usually creating many new instances of a given type of entity that uses
> IDENTITY generation, then this is not an important impact since batching would not have been helpful anyway.

So it sounds like there's some clear trade-offs to using the simplistic `IDENTITY` strategy. What should we do instead?

## Using the SEQUENCE strategy

Digging into the Hibernate documentation and doing some research (Google) quickly reveals that the `SEQUENCE` strategy
is the most viable alternative. This is especially the case if we're using Postgres or SQL Server and have access to
native sequences.

Vlad Mihalcea, an expert on Hibernate, even concludes in
[this post](https://vladmihalcea.com/postgresql-serial-column-hibernate-identity/) (on using `IDENTIFIER` vs `SEQUENCE`
in Postgres):

> Although convenient, and even suggested in many PostgreSQL book, the SERIAL and BIGSERIAL column types are not a very
> good choice when using JPA and Hibernate. Using a `SEQUENCE` generator is a better alternative since the identifier
> can be generated prior to executing the INSERT statement.
>
> Behind the scenes, the SERIAL and BIGSERIAL column types use a database sequence anyway, so the only difference is
> that the `SEQUENCE` generator calls the sequence is a separate database roundtrip. However, this can also be optimized
> with the [pooled and pooled-lo optimizers](https://vladmihalcea.com/hibernate-hidden-gem-the-pooled-lo-optimizer/).
>
> If the database server is close to the application servers and networking is fast, the extra database roundtrip is not
> going to be a performance bottleneck. For all these reasons, you should
> [prefer using the `SEQUENCE` generator over `IDENTITY`](https://vladmihalcea.com/hibernate-identity-sequence-and-table-sequence-generator/)
> no matter if you use PostgreSQL, Oracle or SQL Server.

Sounds great on paper, and should just be a case of changing the annotations slightly. We're sold!

```java
@Entity
public class User {
  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE)
  private Long id;
}
```

That wasn't too difficult, but it's naive to assume the work is complete. Upon firing up your application, you'll
probably actually see an exception when you try to persist your entity. If you are using Postgres, it might look like
the following:

```
org.postgresql.util.PSQLException: ERROR: relation "public.hibernate_sequence" does not exist
```

By default, Hibernate will try to work off of a single sequence called `hibernate_sequence`, so to use the default
behaviour, we should create this sequence before running our application (maybe in a migration).

```sql
CREATE SEQUENCE public.hibernate_sequence INCREMENT 1 START 1 MINVALUE 1;
```

This is not too bad and works out of the box. If we persist some Users, Hibernate will insert them with the correct
identifiers incrementing by 1. The following SQL will get executed by Hibernate:

```sql
INSERT INTO user (id) VALUES (1);
INSERT INTO user (id) VALUES (2);
```

We should note that Hibernate actually knows what IDs to use when executing the insert statements. We no longer rely on
the database to generate our IDs for us and peace has been restored... right?

Actually, something interesting will happen when we persist different types of entities like so:

```java
userRepository.save(new User());
userRepository.save(new User());

postRepository.save(new Post());
```

In the database, we will see the following:

```
postgres=# SELECT id FROM public.user;
id
--
1
2
(2 rows)

postgres=# SELECT id from public.post;
id
--
3
(1 row)
```

You might be surprised to see that the post was saved with an ID of 3! The reason for this is because the default
`hibernate_sequence` will be shared between multiple entity types and consequently, any entities that are persisted will
increment the sequence for any other entity types as well.

For people with OCD about things looking consistent, this clearly sticks out like a sore thumb and is quite undesirable.
Fortunately we can define our own sequences for Hibernate to use instead.

If we are using Postgres, defining the `id` column as the `SERIAL` type will automatically generate a sequence we can
use. For the `user.id` column this would be `user_id_seq`. The User entity can then be made to look like the following:

```java
@Entity
public class User {
  @Id
  @GeneratedValue(generator = "user_id_seq", strategy = GenerationType.SEQUENCE)
  @SequenceGenerator(name = "user_id_seq", sequenceName = "user_id_seq")
  private Long id;
}
```

At this point, you might think you're in the clear and everything should _just work_. Unfortunately for you, Hibernate
has other ideas.

## The weirdness begins

Consider the following actions:

```java
userRepository.save(new User());

// ... restart the application

userRepository.save(new User());
```

What would you expect the database to look like?

You might reasonably expect that there would be two rows in the `user` table with IDs of 1 and 2. On that basis, you
would probably be pretty surprised to see this instead:

```
postgres=# SELECT id FROM public.user;
id
--
1
-46
(2 rows)
```

The second ID being -46 is clearly unexpected behaviour, but unfortunately this is Hibernate working as intended. I
would argue that this is not good API design if the default behaviour creates such an unexpected result, but that's a
discussion for another time.

So why is the second ID being set to such a weird number?

## How sequence generation works

Firstly, we have to understand the interaction between Hibernate and the database's sequence. Before Hibernate performs
any inserts of an entity, it will query the database to receive the next value in the identifier sequence. It can then
use that value as the ID of the following insert. In terms of SQL, it would look like this:

```sql
SELECT nextval('public.user_id_seq'); -- Returns 1
INSERT INTO user (id) VALUES (1);

SELECT nextval('public.user_id_seq'); -- Returns 2
INSERT INTO user (id) VALUES (2);
```

Now this is not efficient as there is now a round trip to the database. We have an additional query for every insert
statement, which could be bad if we have many entities to persist.

To avoid this performance penalty (and make the `SEQUENCE` generator even worth using), Hibernate utilises an
optimization where it assumes that it can **allocate** itself blocks of IDs from the sequence. It can make this
assumption by incrementing the database sequence to the starting value of the next block, and then allocating itself the
previous block.

By default it sets the allocation block size at 50. For a brand new sequence, it will initially increment the sequence
up to 51 before it begins inserting with IDs starting from 1. Once it finishes inserting all of the IDs for its current
block, it gets the next sequence value and continues the cycle.

This is actually a 'pooled hi/lo' generation strategy that is carried out by the `PooledOptimizer` class. The resulting
SQL looks like this:

```sql
-- Retrieves the initial sequence value
SELECT nextval('public.user_id_seq'); -- Returns 1

-- Gets the next sequence value as we need the hi value
SELECT nextval('public.user_id_seq'); -- Returns 51

-- ID generated by 51 (hi) - 50 (lo)
INSERT INTO user (id) VALUES (1);
-- ID generated by 51 (hi) - 49 (lo)
INSERT INTO user (id) VALUES (2);
-- Can insert with IDs up to 51...
INSERT INTO user (id) VALUES (51);

SELECT nextval('public.user_id_seq'); -- Returns 101

-- ID generated by 101 (hi) - 49 (lo)
INSERT INTO user (id) VALUES (52);
```

This entire strategy relies on the backing database sequence incrementing in line with Hibernate's expectations. If we
have a mismatch and the database sequence increments only by 1, whilst Hibernate expects it to increment by 50, the
following happens if we restart the application:

```sql
SELECT nextval('public.user_id_seq'); -- Returns 1
SELECT nextval('public.user_id_seq'); -- Returns 2

-- ID generated by 51 (hi) - 50 (lo)
INSERT INTO user (id) VALUES (1);

-- Restart the application...

SELECT nextval('public.user_id_seq'); -- Returns 3

-- ID generated by 3 (hi) - 49 (lo)
INSERT INTO user (id) VALUES (-46);
```

This explains the weird IDs we were seeing earlier! So how do we prevent it?

## The solution

Armed with the knowledge we now have, it should be fairly easy to see what we need to do to stop getting weird
identifier values. Firstly we should reconfigure the `@SequenceGenerator` annotation to use an allocation size of our
choosing:

```java
@Entity
public class User {
  @Id
  @GeneratedValue(generator = "user_id_seq", strategy = GenerationType.SEQUENCE)
  @SequenceGenerator(
      name = "user_id_seq",
      sequenceName = "user_id_seq",
      allocationSize = 50
  )
  private Long id;
}
```

Secondly, we need to make sure the database sequence's increment is correctly set to match this allocation size:

```sql
ALTER SEQUENCE public.user_id_seq INCREMENT 50;
```

That's all there is to it. However, we will still be able to observe some things we won't necessarily be happy about,
but will have to live with.

### Some caveats

If we restart the application as we have done so before, we will still see erroneous looking IDs. For example, if we
have the allocation size set to 50, we will see the following:

```sql
INSERT INTO users (id) VALUES (1);
INSERT INTO users (id) VALUES (2);

-- Restart the application...

INSERT INTO users (id) VALUES (52);
INSERT INTO users (id) VALUES (53);
```

As I've explained previously, this is pretty much Hibernate working as intended. Upon restarting the application, the
next sequence value from the database will be 101. Hibernate will calculate that the last block will have had at most,
a value of 51, hence the next allowed value is assumed to be 52.

This isn't very pretty as most people are probably not used to seeing weird gaps in their IDs like this. Unfortunately
it's just one of the trade-offs with using Hibernate's `SEQUENCE` strategy. We have to accept this behaviour if we are
to take advantage of its proposed performance optimizations.

It also makes sense from a concurrency standpoint. If multiple copies of the application connect to the same database,
we do not want identifier collisions upon inserts. As each application is able to reserve its own block of identifiers,
we can prevent this from happening.

Something you can do to minimize the potential for ID gaps is to reduce the allocation size to something smaller
like 10. However, allocation size can also vary depending on your requirements.

If you can feasibly expect lots of inserts to occur for an entity, then it is probably worth having a larger allocation
size. If there are unlikely to be many inserts, then it is perfectly okay to use smaller allocation sizes instead.

## Conclusion

Identifier generation is a surprisingly deep and complicated topic in Hibernate. There are weird caveats that can trip
up newcomers and surprise even experienced users. To summarise:

- Avoid using the `IDENTITY` generation strategy when possible. Hibernate cannot optimize itself well for this strategy.
  Instead, it is recommended to use `SEQUENCE` instead, especially with databases like Postgres or SQL Server.
- Make sure that the correct sequences in your database have been created beforehand. By default, Hibernate will try to
  use a shared `hibernate_sequence`, but it is a good idea to use custom sequences for individual entities.
- The sequence generator allocation size should be the same as the database sequence's incrementation size to avoid
  unexpected identifiers being generated (especially negative ones).
- You will have to accept that gaps in your IDs are going to happen at some point if you use the `SEQUENCE` generator
  strategy.

Hopefully I've shed some light on this somewhat convoluted topic and you haven't been totally scared away from
Hibernate. Most of the time it is trying to be helpful by providing some interesting performance optimizations, but we
do tend to have to work a little bit harder to fully understand what's actually going on!
