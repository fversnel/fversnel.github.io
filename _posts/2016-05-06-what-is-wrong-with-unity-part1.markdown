---
layout: post
title:  "What's wrong with Unity - part 1"
date:   2016-05-06 14:00:00 +0200
categories: unity, software design
---
During the development of our Unity-based game [Volo Airsport](https://volo-airsport.com) we fought
numerous battles with the Unity API. Unity is a wonderful tool solves very fundamental 3D engine
problems  but it also uses and encourages bad software design practices.

Unity's approach to API design seems to be concerned with being **easy** to pick up but not **simple** to
use. Where easy means being 'near' or 'at hand' and simple means 'not braded together'. They focus
on making trivial cases really easy to write and at the same time making the
'real world' cases complex to write.

Simplicity facilitates:

- Changing requirements
- Software quality and correctness
- Code comprehension

Simplicity is all about focussing on one thing at a time. For example, it means that all the
capabilities that an API exposes are available for use separately and can be reasoned about
separately. You can see why this would facilitate code comprehension and code quality:
when reading some code you no longer have to untangle it in your head. If you get better code
comprehension it means you can reason better about code correctness and can see clearer what parts
need to change when new requirements are introduced. Rich Hickey explains this very well in
his talk ['Simple Made Easy'](http://www.infoq.com/presentations/Simple-Made-Easy-QCon-London-2012).

The opposite of simplicity is complexity: conflating things together in such a way that they
cannot be used separately.

Simplicity gives us a great measure for design quality. It allows us to verify if a certain part of
the API is concerned with doing one thing and if it isn't we can reason if there's any validity to
its complexity.

In this blog post series I will explain some of the complexity that the Unity API imposes on its users
and I will propose simple(r) solutions to them. Each entry will focus on one particular API call and a
particular set of issues. Hopefully you can use these posts as a guideline on how to deal
with Unity's API and as a guideline when writing your own code.

# Physics.Raycast

This installment will address some of the most basic issues of the Unity API.  We will look at
Unity's
[`Physics.Raycast`](https://docs.unity3d.com/530/Documentation/ScriptReference/Physics.Raycast.html)
function and reason about why it isn't simple and how making it simple improves its
comprehensability and usability.

`Physics.Raycast` is a very commonly used function in almost any game. It traces a given ray through
the world and returns the first thing that was hit.
Let's look at the code that does this:

~~~ csharp
RaycastHit hit;
int layers = LayerMask.Create("UI");
float maxDistance = float.PositiveInfinity;
Ray ray = new Ray(position, direction);
Physics.Raycast(ray, out hit, maxDistance, layers,
                QueryTriggerInteraction.UseGlobal);
~~~

It sends a ray into the world filling the `hit` object with the first UI element that was hit and
returns true. If nothing was hit it returns false and the `hit` object remains empty.

Here's what's wrong:

- It operates on the global scene state.
- It is complex because it mixes three orthogonal concerns together:
    1. Filtering the objects in the world.
    2. Raycasting the filtered objects.
    3. Firing a trigger on the object that was hit.
- It has two return values: a `boolean` and an `out` parameter.
- It has 13 overloads to facilitate all its use-cases.

I will rewrite the definition of this function in small steps to something will no longer use global
state, no longer mixes concerns, and has zero overloads.

# Defining good data structures

Unity has a Ray data structure which looks like this:

~~~ csharp
public struct Ray {
    public Vector3 Origin;
    public Vector3 Direction;
}
~~~

The Ray struct represents a line from a point in space into a given direction.  This is a meaningful
data structure that can be used by all sorts of library functions and is also used by
`Physics.Raycast`. Adding a length property to the Ray actually enhances it without breaking what it
represents:

~~~ csharp
public struct Ray {
    public Vector3 Origin;
    public Vector3 Direction;
    public float Length;
}
~~~

Defining meaningful, coherent data structures is important. Once you have a good data structure you
can use it over and over again in your code. People using your code will recognize this data
structure and can soon grasp what this new function is they're looking at because they're already
familiar with the data structure it uses.

We can now rewrite the `Physics.Raycast` function by removing the `maxDistance` parameter because
the length is now part of the `Ray`:

~~~ csharp
RaycastHit hit;
int layers = LayerMask.Create("UI");
Ray ray = new Ray(origin, direction, length);
Physics.Raycast(ray, out hit, layers,
                QueryTriggerInteraction.UseGlobal);
~~~

`Physics.Raycast` actually has two return values: `RaycastHit` as `out` parameter and a `boolean` as
return value. The boolean indirectly says something about the `RaycastHit` and it is up to you to
fill in the gap that this indirection introduces.  Refactoring the return value into a single
instance alleviates you from this task:

~~~ csharp
int layers = LayerMask.Create("UI");
Ray ray = new Ray(origin, direction, length);
RaycastHit? hit = Physics.Raycast(ray, layers,
                                  QueryTriggerInteraction.UseGlobal);
~~~

It now simply returns a C# [nullable](https://msdn.microsoft.com/en-us/library/b3h38hb0%28v=vs.90%29.aspx) type `RaycastHit?`
returning `null` if nothing was hit and returning a `RaycastHit` struct if something was hit.

# Removing hidden state

Let's look at a hidden variable that isn't part of the function definition:

~~~ csharp
int layers = LayerMask.Create("UI");
Ray ray = new Ray(origin, direction, length);
var physicsWorld = ??? // The hidden variable made explicit
RaycastHit? hit = Physics.Raycast(physicsWorld, ray, layers,
                                  QueryTriggerInteraction.UseGlobal);
~~~

Unity hides the data in the physics world preventing you from creating your own. A raycast is always
performed on all objects in a Unity scene which significantly reduces the function's re-usability.
We can see that the API designer has tried to mitigate this problem by introducing the `layers`
parameter.

The `layers` parameter filters all object in the world based on the layer they are on (each object
can be attached to exactly one layer). The parameter becomes superfluous when you can provide the
`physicsWorld` yourself. In fact the purpose of raycasting has nothing to do with filtering objects
in the world. By doing the filtering separately we gain much more control over what and how we
actually filter the objects that will be given to the raycast function.

*If a new way for filtering objects
is ever introduced in the Unity API the raycast function
needs to rewritten even though the act of raycasting has nothing to do with filtering.*

This is what the physics world might look like:

~~~ csharp
public class PhysicsWorld {
    public Set<Collidable> Objects;
}
~~~

Even though this is a simplified representation it will help us to illustrate what happens when a
variable like this is available to us.

Before doing a raycast we can filter the set of collidables based on arbitrary properties
instead of just `layers`, in this case we filter based on the `layer` and the `IsVisible` property:

~~~ csharp
PhysicsWorld physicsWorld;
int uiLayer = LayerMask.Create("UI");
physicsWorld = physicsWorld.Filter(collidable => {
    var isUiElement = (collidable.layer & uiLayer) != 0;
    return isUiElement && collidable.IsVisible;
});

Ray ray = new Ray(origin, direction, length);
RaycastHit? hit = Physics.Raycast(physicsWorld, ray,
                                  QueryTriggerInteraction.UseGlobal);
~~~

Unfortunately we cannot provide a version of `Physics.Raycast` that exposes the `PhysicsWorld` as a
parameter unless we reverse engineer the Unity code.  Nevertheless it is important to see what kind
of possibilities open up when the parameter is exposed.

The trigger that is fired when a raycast hits an object is another concern that is also completely
separate from raycasting. Imagine we are working with colliders that don't support triggers. The
current implementation makes it impossible to do that. Instead we can fire the trigger separately
after we've done the raycast if we so desire:

~~~ csharp
Ray ray = new Ray(origin, direction, length);
RaycastHit? hit = Physics.Raycast(physicsWorld, ray);

if(hit.HasValue) {
    hit.Value.Collider.Trigger();
}
~~~

# Using object orientation

By using [extensions methods](https://msdn.microsoft.com/en-us/library/bb383977.aspx) we can add the
raycast function in the proper scope such that we can easily understand in which context it's
executed:

~~~ csharp
public static RaycastHit? Raycast(this PhysicsWorld world, Ray ray) {
    ...
}
~~~

A raycast now becomes:

~~~ csharp
PhysicsWorld physicsWorld;
RaycastHit? hit = physicsWorld.Raycast(new Ray(origin, direction, length));
~~~

# Conclusion

The final code looks like this:

~~~ csharp
PhysicsWorld physicsWorld = ...;
int uiLayer = LayerMask.Create("UI");
physicsWorld = physicsWorld.Filter(collidable => {
    var isUiElement = (collidable.layer & uiLayer) != 0;
    return isUiElement && collidable.IsVisible;
});

RaycastHit? hit = physicsWorld.Raycast(new Ray(origin, direction, length));

if(hit.HasValue) {
    hit.Value.Collider.Trigger();
}
~~~

This is more code than we started out with so how can this possibly be better?
We have separated concerns and have arrived at a simple definition of what raycasting is.
Because of that we are now able to filter the objects in the world based custom predicates instead of
being forced to use layers. Triggering has become a seperate concept as well that we can choose to
be concerned with but is not forced upon us. Changes in the code of one concern do not
directly impact the others anymore.
