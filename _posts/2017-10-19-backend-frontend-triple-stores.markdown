---
layout: post
title:  "Back-end and front-end: two sides of the same coin; or why triple stores
are fundamental to application development."
date:   2017-10-19 16:25:00 +0200
published: true
categories: ['triple store', 'software design', 'software architecture', 'database']
---
People often ask me if I'm a front-end developer or a back-end developer. I'm
always glad to answer I'm both, or neither depending on your point of view. My
issue with specializing in either one of them is that you lose perspective on
the bigger picture. You're no longer need to think about what happens on the
other side of the fence.

When I set out to build an app where you can review music and recommend music to
your friends, I thought I'd try a different approach. My quest became to share
as much code between front-end and back-end as good software design would allow.
I heard about triple stores and how they can store data in a very loosely
defined structure without losing the ability to do complicated queries.

*Hats off to the people in the Clojure community who've been advocating the use
of triple stores (Rich Hickey, David Nolen, Nikita Prokopov). Without them I
would have never even thought of using triple stores. Furthermore, Clojure makes
it ridiculously simple to share code between server and client, which was
essential to make this whole idea work.*

# Triple stores

A triple store consists of a set of triples. A triple always consists of:

1. A subject (a user, a piece of music, a review etc.)
2. A relation (something that the subject has or refers to)
3. A value (this can be another subject, a string, or any other value for that matter)

This is a triple:

`frank likes pizza`

Where `frank` is the subject, `likes` is the relation, and `pizza` is the value.

Consider this database:

```
frank has-password "passw0rd"
frank has-email "frank@example.com"

review-1 author frank
review-1 title "Glasper's killing it!"
review-1 record double-booked
review-2 author frank
review-2 title "Ryuichi Sakamoto - async"
review-2 record async

double-booked artist "Robert Glasper"
double-booked album "Double Booked"

async artist "Ryuichi Sakamoto"
async album "async"
```

We can now query for example all authors that wrote a review, like so:

```
?subject author ?author
```

Every word starting with `?` is something we don't know and want the database to
look up for us. Words not starting with a `?` are things we do know. Things we
do know can be literal strings like `author` but also variables from our program
code.

The query gets us all reviews and their authors, in this case `review-1 author
 frank`, `review-2 author frank`. 
 
 Taking it a step further we can also get authors that wrote a review for Robert
 Glasper's album Double Booked:

```
?review record double-booked
?review author ?author
```

As you can see the set of triples form a graph where subjects are connected to
one another. If you know SQL you can see we can do implicit joins with the query
language very easily. We get all reviews for `double-booked` and then extract
the author for each of the reviews.

Having such a loosely defined structure while retaining a very rich query
capability got me thinking: what if we would use only one data structure for our
entire application? The entire application state as a triple store, front-end
and back-end. This would solve two fundamental problems:

1. **The data formatting problem**: where the data is never formatted in the right
   way. How often are we not transforming our data structure to fit our
   environment? We would get rid of this glue code once and for all.
2. **The code sharing problem**: Code between back-end and front-end almost
   never gets shared because the data structures are quite different. For
   rendering we often need different structure than for our domain logic.

# The back-end

The back-end is set up as a machine that processes incoming user commands,
validates them, and updates the application database.

An incoming command is simply a set of triples with a single label attached to
it. For example a label `review` and a set of triples `(review-1 title "Robert
Glasper - Double Booked" etc..)`.

Since the database is a triple store we can easily share it with the front-end.
First of all, the back-end sends all its triples to the front-end initially
(filtering out the user passwords and other sensitive data first). Each user
command can result in zero or more domain events which follow the same structure
of `label+triples`. It also sends off these events to the front-end so that it
can update its state accordingly.

# The front-end

The front-end also has a triple store and receives updates from the back-end
through events. However, this isn't enough: the front-end needs more information
to do rendering. For example it needs to know about UI state: which elements are
highlighted etc. The front-end extends the back-end triple store with some more
triples, like:

```review is-highlighted true```

and

```review is-expanded true```

Due to the simple structure of the triple store (after all it's just a flat
unordered set of triples) there's no need to worry something might not fit. We
can just add more triples (or remove them). The domain between the back-end and
front-end is shared, so it's very unlikely the triples in the back-end database
don't match the ones needed by the front-end. Some triples that are used in the
back-end might never get used in the front-end. It's no problem to just ignore
them, they won't break anything.

The most beautiful thing is that no matter how complicated the UI gets, we can
always write a query that gives us the right data back. We can get all music
reviews and their authors, and the bands that played the music etc. No need to
dig into these deeply nested data structures anymore to find what we're looking
for.

# Concluding

In essence triple stores give us the ability to write many functions for a
single data structure. This gives us a tremendous reduction in complexity and a
great increase in code re-use.

While there's a lot of novelty in this approach and the wrinkles are by no means
ironed out, I can only dream of a future where application development is done
this way.

To all developers I would like to say: always be critical of your application's
architecture. Never just take it for granted. Keep looking for alternatives.
Write down what annoys you. But most of all remember that the front-end and
back-end divide is just made up. It doesn't need to be there if you don't want
to.
