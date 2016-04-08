---
layout: post
title:  "What's wrong with Unity - part 1 (unfinished)"
date:   2016-04-07 12:29:10 +0200
categories: unity
---
Unity encourages bad software design practices. In a series of blog posts
I will explain the problems that the Unity API imposes on its users
and propose solutions to them. Each installment will focus on one particular
API call or one particular type of design problem.

Hopefully you can use these posts as a guideline on how to deal with Unity's
API and as a guideline when writing your own code.

In this installment I will address some of the most basic issues of the Unity API.
Let's take a look at the Unity function [`Physics.Raycast`](https://docs.unity3d.com/530/Documentation/ScriptReference/Physics.Raycast.html) as an
example.

`Physics.Raycast` is a very commonly used function in almost
any game. It traces a given ray through the world and returns the first thing
that was hit. As of Unity 5.3.4 it has over 14 overloads. I will start
by redefining the function in small steps until we arrive at a single function definition
that covers all use-cases provided by the 14 overloads and more. More importantly
we will arrive at a function definition that will be easier to understand and easier to reason about.

~~~ csharp
RaycastHitInfo hitInfo;
int layers = LayerMask.Create("UI");
float maxDistance = float.PositiveInfinity;
Ray ray = new Ray(position, direction);
Physics.Raycast(ray, out hitInfo, maxDistance, layers, QueryTriggerInteraction.UseGlobal);
~~~

<!-- - TODO: Why are there so many overloads for this function? It does a poor job at separating concerns
    - It does filtering
    - It does raycasting but not with the right data structure.
- TODO: Why is there hidden global state? Because Unity wants to be perceived as being easy to use
- TODO: But it isn't simple. Explain easy vs. simple, refer to Rich Hickey talk.
- TODO: How do we prevent from creating APIs that are so convoluted. -->

Let's look at what this function does. It sends a ray into the world filling the hitInfo
object with the first UI element that was hit and returns true. If nothing was hit it returns
false and the hitInfo object remains empty.

<!-- - Uses global state to perform ray check
- It resides in a stateful static class (Singleton considered harmful)
- It is tied to a specific implementation of a physics library (Non-extensibility)
- Does not use Ray data structure (requires extra programmer comprehensions,
    makes it feel detached from other parts of the API) -->

This sets an example to other programmers using Unity and actively encouraging the use of:

- Global state
- Singletons
- Non-extensibility
- No abstraction

# Defining good data structures

<!-- TODO Explain why good data structures are important -->

Unity has a Ray data structure which looks like this:

{% highlight csharp %}
public struct Ray {
    public Vector3 Origin;
    public Vector3 Direction;
}
{% endhighlight %}

The Ray struct represents a line from a point in space into a given direction.
This is a meaningful data structure that can be used by all sorts of library functions and
is also used by `Physics.Raycast`. Adding a length property to the Ray actually enhances it
without breaking what it represents:

{% highlight csharp %}
public struct Ray {
    public Vector3 Origin;
    public Vector3 Direction;
    public float Length;
}
{% endhighlight %}

<!-- - TODO: ray length might be redundant and can be part of the Direction? -->

We can then rewrite the `Physics.Raycast` function by removing the `maxDistance` parameter:

~~~ csharp
int layers = LayerMask.Create("UI");
Ray ray = new Ray(origin, direction, length);
Physics.Raycast(ray, out hitInfo, layers, QueryTriggerInteraction.UseGlobal);
~~~

Now this HitInfo variable has been bugging me. It means the `Physics.Raycast` function
actually has two return values, namely the `hitInfo` as `out` parameter and a `boolean` as return value.
This can easily be factored into a single return value, like so:

~~~ csharp
int layers = LayerMask.Create("UI");
Ray ray = new Ray(origin, direction, length);
HitInfo? hit = Physics.Raycast(ray, layers, QueryTriggerInteraction.UseGlobal);
~~~

The function now simply returns a C# [nullable](https://msdn.microsoft.com/en-us/library/b3h38hb0%28v=vs.90%29.aspx) type `HitInfo?`.
It returns `null` if nothing was hit and returns the `HitInfo` if something was hit.

# Removing hidden state

Let's look at a hidden variable that isn't part of the function definition
but should be:

~~~ csharp
int layers = LayerMask.Create("UI");
Ray ray = new Ray(origin, direction, length);
var physicsWorld = ??? // This is the hidden variable made explicit
HitInfo? hit = Physics.Raycast(physicsWorld, ray, layers, QueryTriggerInteraction.UseGlobal);
~~~

Unity actually hides the data in the physics world preventing you
from creating your own. Which means any raycast is always performed on all
objects in a Unity scene. This significantly reduces the re-usability of the
Raycast function. We can see that the API designer has tried to mitigate this
problem by introducing the `layers` parameter.

The `layers` parameter filters all object in the world based on the layer they
are on. Each object can be attached to exactly one layer. The parameter becomes unnecessary when you can provide the `physicsWorld`
yourself. In fact the purpose of raycasting has nothing to do with filtering objects in the world. According
to Rich Hickey's definition of simple this should be done separately. By doing the filtering separately we
gain much more control over what and how
we actually filter the objects that will be given to the raycast function.

What might this physicsWorld variable look like:

~~~ csharp
public class PhysicsWorld {
    public Set<Collider> Objects;
}
~~~

Before doing a raycast we can filter the set of colliders based on arbitrary properties
instead of just `layers`:

~~~ csharp
PhysicsWorld physicsWorld;
int uiLayer = LayerMask.Create("UI");
physicsWorld = physicsWorld.Filter(o => (o.layer & uiLayer) != 0);

Ray ray = new Ray(origin, direction, length);
HitInfo? hit = Physics.Raycast(physicsWorld, ray, QueryTriggerInteraction.UseGlobal);
~~~

# Using object orientation

By using [extensions methods](https://msdn.microsoft.com/en-us/library/bb383977.aspx) we can use object orientation to add the raycast
function in the proper scope such that we can easily understand in which context
the raycast can be performed.

~~~ csharp
public static HitInfo? Raycast(this PhysicsWorld world, Ray ray) {
    ...
}
~~~

A raycast now becomes:

~~~ csharp
PhysicsWorld physicsWorld;
HitInfo? hit = physicsWorld.Raycast(new Ray(origin, direction, length));
~~~

# Conclusion

Final code:

~~~ csharp
PhysicsWorld physicsWorld = ...;
int uiLayer = LayerMask.Create("UI");
physicsWorld = physicsWorld.Filter(o => (o.layer & uiLayer) != 0);

HitInfo? hit = physicsWorld.Raycast(new Ray(origin, direction, length));

if(hit.HasValue) {
    hit.Value.Trigger();
}
~~~

This is more code than we started out with so how can this possibly be better? It's better
because we have separated the concerns and have arrived at a pure definition of what raycasting is
without other unrelated concerns mixed in. This is much more important than being able
to write certain cases succinctly while being unable to write others.
