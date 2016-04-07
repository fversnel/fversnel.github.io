---
layout: post
title:  "What's wrong with Unity - part 1 (draft)"
date:   2016-04-07 12:29:10 +0200
categories: unity
---
Unity encourages bad software engineering practices.

It makes the trivial cases more trivial and the real world cases very hard.

What's wrong:

- Basic explanation on what's wrong using Physics.Raycast as an example.

{% highlight csharp %}
RaycastHitInfo hitInfo;
int layers = LayerMask.Create("UI");
float length = float.PositiveInfinity;
Physics.Raycast(position, direction, out hitInfo, length, layers, QueryTriggerInteraction.UseGlobal);
{% endhighlight %}

Let's look at what this function does. It sends a ray into the world filling the hitInfo
object with the first thing that was hit and returns true. If nothing was hit it returns
false and the hitInfo object remains empty.

- Uses global state to perform ray check
- It resides in a stateful static class (Singleton considered harmful)
- It is tied to a specific implementation of a physics library (Non-extensibility)
- Does not use Ray data structure (requires extra programmer comprehensions,
    makes it feel detached from other parts of the API)

This sets an example to other programmers using Unity and actively encouraging the use of:

- Global state
- Singletons
- Non-extensibility
- No abstraction

# Defining good data structures

Unity has a Ray data structure which looks like this:

{% highlight csharp %}
public struct Ray {
    public Vector3 Origin;
    public Vector3 Direction;
}
{% endhighlight %}

The Ray struct represents a ray that is cast from a point into a given direction.
This is a meaningful data structure that can be used by all sorts of library functions
to do meaningful things but it isn't used by Physics.Raycast even though it could be easily modified
to do so:

{% highlight csharp %}
Ray ray = new Ray(origin, direction);
float length = float.PositiveInfinity;
Physics.Raycast(ray, out hitInfo, length, layers, QueryTriggerInteraction.UseGlobal);
{% endhighlight %}

Adding a length property to the Ray actually enhances it
without breaking what a Ray represents:

{% highlight csharp %}
public struct Ray {
    public Vector3 Origin;
    public Vector3 Direction;
    public float Length;
}
{% endhighlight %}

We can then rewrite the Physics.Raycast function again making it even clearer:

{% highlight csharp %}
Ray ray = new Ray(origin, direction, length);
Physics.Raycast(ray, out hitInfo, layers, QueryTriggerInteraction.UseGlobal);
{% endhighlight %}

Now this HitInfo variable has been bugging me. It means the Physics.Raycast function
actually has two return values that could be easily factored into one improving the
readability of the function:

{% highlight csharp %}
Ray ray = new Ray(origin, direction, length);
RaycastHit? hit = Physics.Raycast(ray, layers, QueryTriggerInteraction.UseGlobal);
{% endhighlight %}

The function now simply returns a nullable HitInfo and returns null if nothing was hit and
actually returns the HitInfo if something was hit.

Now the function definition is much simpler and is easier to understand especially
for people that are familiar with the Ray struct.

Let's look at a nasty hidden variable that isn't part of the function definition
but should be:

{% highlight csharp %}
Ray ray = new Ray(origin, direction, length);
var physicsWorld = ???
RaycastHit? hit = Physics.Raycast(physicsWorld, ray);
{% endhighlight %}

Unity actually hides the data in the physics world from you preventing you
from creating your own which means any raycast is always performed on all
objects in a Unity scene. This significantly reduces the re-usability of the
Raycast function and you can see that the API designer has tried to mitigate this
problem by introducing the layers and QueryTriggerInteraction parameters. We'll see
later why these parameters are completely orthogonal to raycasting.

Now we also no longer need the QueryTriggerInteraction and layers parameter because we determine
ourselves which objects inside the physicsWorld are being checked in the raycast.

What might this physicsWorld variable look like:

{% highlight csharp %}
public class PhysicsWorld {
    public Set<Collider> Objects;
}
{% endhighlight %}

Before doing a raycast we can simply filter the set of colliders based on arbitrary properties
like the layer property and the QueryTriggerInteraction property and much more.

# Conclusion

We've successfully introduced two data structures and redefined what it means to
do raycasting in Unity.
