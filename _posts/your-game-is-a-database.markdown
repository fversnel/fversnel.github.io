---
layout: post
title:  "Your game is a database"
date:   2016-04-07 12:29:10 +0200
published: false
categories: unity, software design, reactive programming
---
Components describe change and what data to act upon.

We only want to re-render what changed!

Unity's components are not simple:
- They incorporate the notion of time
- They incorporate the notion of state that often comes with the notion of time
- They have identity (totally unnecessary)
- They know about the game object that they're coupled to.

My attempt to solve this is issue revolves around a few core ideas:
- Extract the notion of time to some separate process and describe the main
  part of the game logic as simple data transformations
- Allow each component to describe (i.e. query) the data that it wants to use
- Allow components to be organized in a hierarchy ala Om.Next. This is the fundamental
to making components re-usable. E.g. a piece of logic should be able manifest itself
inside a larger structure. If we take a human body as example a piece of logic that
can move the arm should be able to describe movements of the arm within the structure
of the whole human body. Recursively that human body is located in a larger world
that contains multiple bodies all acting largely autonomously from each other
until they collide.
- Single source of truth
- Essential state vs. non-essential state

We have data and data transformations. Some data is indirectly dependent upon other
data because one data transformation has to run before others.
The absolute easiest way to perform data transformations that depend upon one another
is through function composition


The arm shouldn't know that it is attached to a human body. It might know about
the joint it is attached to in some abstract sense but the principle of least knowledge
should always apply.

What's better at describing relationships in data than a database?

TODO:
- Use datascript as a basis
    - Define a schema
    - Define example queries
