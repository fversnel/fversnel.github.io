---
layout: post
title:  "Local state considered harmful"
date:   2016-04-07 12:29:10 +0200
published: false
categories: unity, software design
---
In Unity we are forced to write components that encapsulate state and behavior in
the traditional object oriented sense. Any communication between components takes
place through method calls. Component A can be dependent upon Component B
because it has to provide B with some information that it has calculated.

In this post I will explain why this is often problematic and
I will propose solutions to the problems that working with components impose.

Unity components entangle the following concepts:

- Each component models it's process as an implicit function over time.
- Each component owns a bit of state
- Each component has the responsibility to communicate with other components.

If we can extract the implicit model of time from the components and move it to
a single separate process that deals with this problem we can make components drastically simpler.
I will argue that most components will become pure functions that take and produce data.
This approach allows us to no longer having to depend on each others
concrete instances but each component can take each others data instead and can be composed by
a separate process that ties the concrete instances together.

A common problem with encapsulating state in small components is that we cannot
reason about the state on a higher level. A real world problem when dealing with
a multiplayer game where you need to optimize what data to send. The only partial solution
to this is to write a serialize method for each component but this doesn't solve the
optimization problem. Also you give the component the extra burden of taking into
account the multiplayer context of the game.

It would be much easier if we could separate state from components and have some over
arching process that could feed the state into the components no matter where it
came from (e.g. a calculation of the previous frame, data from the network, data
from a file.). As soon as we start to alleviate the component from the burden of
tracking state, we take out the process of time and let a separate process deal with that,
and we change the communication between components to a system that routes data
from one component to the other we get more loosely coupled components that are
much easier to change.
