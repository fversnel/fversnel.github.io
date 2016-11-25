---
layout: post
title:  "What's wrong with Unity (2) - Component construction"
date:   2016-11-25 15:02:10 +0100
published: true
categories: unity, software design
---
Unity's components do not rely on C# object constructors. My best guess on why they don't
is the increased flexibility that you get. In this post I will argue why not using
constructors in the design of components is harmful for their understandability and why it
is a potential source for bugs.

Consider the following Unity components:

~~~ csharp
public class ComponentA : MonoBehaviour {
    private int _state;

    void Awake() {
        _state = 42;
    }

    public int State {
        get { return _state; }
    }
}

public class ComponentB : MonoBehaviour {
    [SerializeField] private ComponentA _a;

    private int _initialState;

    void Awake() {
        _initialState = _a.State;
    }    
}
~~~

`ComponentB` relies on `ComponentA` for providing it with some state. We don't know
what the state of `ComponentA` is and whether Awake has been called or not when
we are initializing `ComponentB`. To work around this we could initialize B in
`Start()` to make sure `Awake()` was called but that doesn't really solve the problem
if you have more than one layer of indirection (e.g. `ComponentA` itself depends
    on the initialization of some other component).

This problem could be easily solved by using constructors, like so:

~~~ csharp
public class ComponentA : MonoBehaviour {
    private int _state;

    public ComponentA() {
        _state = 42;
    }

    public int State {
        get { return _state; }
    }
}

public class ComponentB : MonoBehaviour {
    private int _initialState;

    public ComponentB(ComponentA a) {
        _initialState = a.State;
    }    
}
~~~

Now we can have as many layers of indirection as we want. The only limitation is
that we can no longer have circular dependencies this way.
In fact this way of working has been part of many OO languages for a long time
and I wonder why Unity doesn't use it. (Also popular dependency injection frameworks
support constructor injection.) After all a constructor meaningfully communicates
what an object should contain before it is qualified to exist. Unity's components make
it far too easy to end up with `null` or uninitialized references forcing you
to hunt for any runtime initialization bugs yourself while the compiler could
have alleviated you from this task.
