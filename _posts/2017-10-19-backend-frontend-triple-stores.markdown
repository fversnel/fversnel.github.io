---
layout: post
title:  "Back-end and front-end, two sides of the same coin; or why triple stores 
are fundamental to application development."
date:   2017-10-19 14:00:00 +0200
categories: ['triple store', 'software design', 'software architecture', 'database']
---

People often ask me if I'm a front-end developer or a back-end developer. I'm
always glad to answer I'm both, or neither depending or your perspective. My
issue with specializing in either one of them is that you lose perspective on
the bigger picture. You no longer are required to think about the other side of
the fence, that is the front-end fence or the back-end fence.

When I set out to build an app where you can review music and recommend music to
your friends, I thought I'd try a different approach. My quest became to share
as much code between front-end and back-end as good software design would allow.
I heard about triple stores and how they can store data in a very loosely
defined structure without losing the ability to do complicated queries. The
particular triple store I got interested in is called Datascript and runs in
memory on both Javascript (front-end) and the JVM (back-end).

## Triple stores 

A database consists of a set of triples. A triple always consists of:

1. A subject (a user, a piece of music, a review etc.)
2. A relation (something that the entity has or refers to)
3. A value (this can be another entity, a string, or any other value for that matter)

A triple looks like this:

`frank likes pizza`

Where `frank` is the subject, `likes` is the relation, and `pizza` is the value.

Consider this database:

```
frank likes pizza
frank likes programming
frank knows-about pizza
frank knows-about clojure
martijn knows-about clojure
clojure is programming-language
programming is fun
pizza is food
```

We can now query for example all entities that know about the clojure
programming language, like so:

```
?entity knows-about clojure
```

We get all the entities that know about `clojure`. Taking it a step further we
can also get the type of things someone knows about:

```
?subject knows-about ?something
?something is ?type
```

We now get: `frank clojure programming-language`, `frank pizza food`, `martijn
clojure programming-language`. If you know SQL you can see we can do implicit
joins with the query language very easily.

Having such a loosely defined structure while retaining a very rich query
capability got me thinking. What if we would use only one data structure for our
entire application? The entire application state as a triple store, front-end
and back-end. This would solve two fundamental problems:

1. The data formatting problem: where the data is never formatted in the right
   way. How often are we not transforming our data structure to fit our
   environment? We would get rid of this glue code once and for all.
2. Code between back-end and front-end almost never gets shared because the data
   structures are quite different. For rendering we often need different
   structure than for our domain logic. 

## The back-end

The back-end is set up quite simply as a machine that processes incoming user
commands, validates them, and updates the application database.

Since the database is a triple store we can easily share it with the front-end
and the back-end simply sends all its triples to the front-end (filtering out
the user passwords and other sensitive data first). Each user command can result
in zero or more domain events (like `reviewed(user-id, title, etc...)`), which
represent actual updates in the application state. On the back-end these events
get translated into new triples and it sends them off to the client.

## The front-end

The front-end also has a triple store and receives updates from the back-end
through events. However, this is not enough, the front-end needs more
information to do rendering. For example it needs to know about UI state, which
elements are highlighted etc. The front-end extends the back-end triple store
with some more triples, like:

```review is-highlighted true```

and

```review is-expanded true```

Due to the simple structure there's no need to worry something might not fit. We
just add more triples. The domain between the back-end and front-end is shared,
so it's very unlikely the triples in the back-end database don't match the ones
needed by the front-end.

The most beautiful thing is that no matter how complicated the UI gets, we can
always write a query that gives us the right data back. We can get all music
reviews and their authors, and the bands that played the music etc. No need to
dig into these deeply nested data structures anymore to find what we're looking
for.

## Sharing queries

Database queries that are used in the back-end can be re-used by the front-end.
We can re-use the query that fetches all reviews for example. Or the query
that fetches all reviews for a particular user and so on.

TODO Expand on this.

## Concluding

While there's a lot of novelty in this approach and the wrinkles are by no means
ironed out, I can only dream of a future where application development is done
this way.

To all developers I would like to say: stay critical of your application's
architecture. Never just take it for granted. Always keep looking for
alternatives. Write down what annoys you. But most of all remember that the
front-end and back-end divide is just made up. It doesn't need to be there if
you don't want to.
