---
layout: post
title:  "Reactive Street Fighter"
date:   2016-08-24 10:51:10 +0200
published: true
categories: unity, software design, reactive programming
---

# Introduction
Programming game behavior over time and why it is painful. Scripting game behavior is hard. You
have to answer questions like: *How do I organize the interaction between entities?*, *How do I
detect when an event has occurred?*. There is an inherent complexity to answering those questions,
but there is also an aspect that makes it unnecessarily difficult: the lack of proper programming
tools.

Let’s look at various descriptions for game behaviors: *when the player presses the A button his
character jumps*, *when two seconds have passed put a new enemy into the game*, *when the player
stands next to an item and he presses the left mouse button the item is picked up*. The word *when*
in those sentences is a keyword: *when this happens I want to react by doing that*. A lot of game
behavior is about describing when to react.  In order to remove unnecessary (accidental) complexity
from programming such game behavior we need a tool that helps us to efficiently and accurately
describe when things happen.

The reactive programming paradigm is solely concerned with describing events. Where an event can
literally be anything: from a low-level ‘button being pressed’ to an application specific event like
'the player jumps'. Unlike other systems that work with events (like .NET events), the reactive
programming paradigm turns an event into a first-class value meaning it is able to construct new
events from existing ones by transforming, combining and filtering them.

A few years ago Microsoft released Reactive Extensions, also known as Rx, an implementation of the
reactive paradigm, and with it putting the paradigm in reach of the .NET and Mono developers.

Unity3D is a popular game engine that also runs on the Mono platform.

In this article we’ll compare two approaches to programming a fighting game’s input system with
Unity3D. The first implementation is completely based on polling, a common programming technique in
game programming. The second implementation is completely based on the reactive paradigm. We’ll use
C#, and Rx to for the reactive implementation. For the polling approach we’ll use plain C#.

When reading this article you should already have a basic understanding of Unity3D and the C#
programming language, experience with Rx is not required.

Reading this article will provide you with a better understanding of reactive programming and how it
can be applied to games. It should become clear what the differences and advantages of reactive
programming are compared to a more standard approach to programming game behavior.  A fighting game
input system Fighting games have complex input mechanics. To program such a system requires us to
know exactly when keys are being pressed and when a valid move was entered by the player. We compare
a traditional polling implementation to a completely reactive implementation.

Our input system has the following characteristics:

A fighter has the ability to perform moves.
Moves are entered by the player pressing keys on the keyboard.
The following moves can be performed:

- `q` triggers a light punch.
- `w` triggers a medium punch.
- `e` triggers a heavy punch
- `right arrow` followed by `q` triggers a fireball.
- `q`, `w`, `e` pressed simultaneously triggers a super punch.
Keys are pressed simultaneously if they are pressed within 20 milliseconds of each other.  Keys are
pressed sequentially if they are pressed within 20-400 milliseconds of each other. Any key presses
out of that time range cannot be considered to belong to a single move.

# The combat system’s data structures.

Moves are defined by an input sequence of keystrokes that are mapped to a MoveType , e.g. forward +
right is mapped to the Fireball MoveType. The following data structures provide all that is
necessary to start implementing both versions of the combat system:

~~~ csharp
public enum MoveType {
    LightPunch, MediumPunch, HeavyPunch, SuperPunch, Fireball
}

public class InputSequence {
    public IList<IList<KeyCode>> sequence;
}
~~~

The InputSequence class represents a series of keystrokes where the inner list are the keys that are pressed simultaneously and the outer list are the keys that are pressed sequentially.

~~~ csharp
public class Move {
    public MoveType type;
    public InputSequence inputSequence;
}
~~~

The Move class maps the InputSequence to a MoveType so that we know which series of keystrokes
belong to which move.

~~~ csharp
public static class BoxerData {
    public static readonly IList<Move> Moves = new List<Move>();

    static BoxerData() {
        Moves.Add(new Move(new InputSequence(new[,] { { KeyCode.Q,
                                                        KeyCode.W,
                                                        KeyCode.E } }),
                           MoveType.SuperPunch));
        Moves.Add(new Move(new InputSequence(new[,] { { KeyCode.RightArrow },
                                                      { KeyCode.Q } }),
                           MoveType.Fireball));
        Moves.Add(new Move(new InputSequence(new[,] { { KeyCode.Q } }),
                           MoveType.LightPunch));
        Moves.Add(new Move(new InputSequence(new[,] { { KeyCode.W } }),
                           MoveType.MediumPunch));
        Moves.Add(new Move(new InputSequence(new[,] { { KeyCode.E } }),
                           MoveType.HeavyPunch));
    }
}
~~~

Finally, we create the mapping as specified by the requirements and store the result in a publicly
accessible list. The list can later be used to check if incoming input matches a Move.

# A combat system implemented using polling

We present an implementation using polling to handle the incoming input. Polling is a common
technique in programming game logic, where at a certain interval, e.g. each frame, state is checked
in order to make a decision in the program. This is contrary to a push-based approach which we will
show later, where code is executed when an event occurs (for example, when a key is pressed)

For starters let’s create a buffer for simultaneous key presses, which we can later store in another
buffer that contains all sequential key presses. The buffers are cleared if keys are not pressed
fast enough: 20 milliseconds for the simultaneous buffer and 400 milliseconds for the sequential
buffer.

~~~ csharp
public class InputBuffering : MonoBehaviour {
    private static readonly TimeSpan BufferTimeOut = TimeSpan.FromMilliseconds(400);
    private static readonly TimeSpan MergeInputTime = TimeSpan.FromMilliseconds(20);

    private TimeSpan _lastSequentialKeysBufferUpdateTime;
    private InputSequence _buffer;

    private TimeSpan _mergeInputTime;
    private List<KeyCode> _mergeBuffer;

    private IEnumerable<KeyCode> _keys;

    ...
}
~~~

There are four notable variables that are used for the buffer logic:

- mergeInputTime - The time that the merge window started.
- mergeBuffer - Holds the keys pressed in the current merge window.
- buffer - Holds all current sequential input.
- lastBufferUpdateTime - The time that the buffer was last updated.

~~~ csharp
void Update() {
    // Expire old input.
    TimeSpan time = TimeSpan.FromSeconds(Time.time);
    TimeSpan timeSinceLastUpdate = time - _lastBufferUpdateTime;
    if (timeSinceLastUpdate > BufferTimeOut) {
        _buffer.Clear();
    }
}
~~~

Before we actually check which keys are pressed we clear the buffer if the time since the last
update of the buffer was earlier than the sequential buffer timeout (400 milliseconds), i.g. if the
keys that are being pressed in this frame are sequential to the keys that are already in the buffer
we don’t clear it. We also store the current time in a variable for later use.

~~~ csharp
// Get all of the keys pressed this frame.
var keysPressed = new List<KeyCode>();
foreach (var key in _keys) {
    if (Input.GetKeyDown(key)) {
        keysPressed.Add(key);
    }
}
~~~

Now we fetch all the keys that are being pressed during this frame and put them into a list.

~~~ csharp
TimeSpan timeSinceMergeWindowOpen = time - _mergeInputTime;
// It is very hard to press two buttons on exactly the same frame.
// If they are close enough, consider them pressed at the same time.
bool isMergableInput = timeSinceMergeWindowOpen < MergeInputTime;

if (isMergableInput) {
    _mergeBuffer.AddRange(keysPressed);
} else {
    if (_mergeBuffer.Count > 0) {
        _buffer.Add(_mergeBuffer);
        _lastBufferUpdateTime = time;

        // Clear the merge buffer
        _mergeBuffer = new List<KeyCode>();
    }

    // Start a new merge buffer
    if (keysPressed.Count > 0) {
        _mergeBuffer = keysPressed;
        _mergeInputTime = time;
    }
}
~~~

The boolean value `isMergableInput` indicates if we are merging simultaneous input. Merging input takes
place if the keys being pressed fall into a merge window, i.g. if the time since the merge window
has opened is within the bounds of simultaneous input (20 milliseconds).

If the currently pressed keys do not belong in the merge buffer that means that the merge buffer
window is closed and the merge buffer keys might (only if it isn’t empty) need to be copied to the
sequential input buffer.

if there were any keys pressed we open a new merge window by setting the merge input time to the
    current time.

~~~ csharp
private bool MatchSimultaneousKeys(IEnumerable<KeyCode> bufferedInput, IEnumerable<KeyCode>
template) {
    foreach (var action in template) {
        if (!bufferedInput.Contains(action)) {
            return false;
        }
    }
    return true;
}
~~~

A helper method that is used for matching buffer input to Moves that determines if buffered
simultaneous input matches the specified template input. The order of the keys in the lists does not
matter as it does not matter in what order the keys were pressed since they’re regarded as being
pressed simultaneously.

~~~ csharp
public bool Matches(Move move) {
    // If the move is longer than the buffer, it can't possibly match.
    if (_buffer.Count < move.InputSequence.Length) {
        return false;
    }

    // Loop backwards to match against the most recent input.
    for (int i = 1; i <= move.InputSequence.Length; ++i) {
        var bufferedValue = _buffer[_buffer.Count - i];
        var moveValue = move.InputSequence.Value[move.InputSequence.Length - i];

        if (!MatchSimultaneousKeys(bufferedValue, moveValue)) {
            return false;
        }
    }

    _buffer.Clear();

    return true;
}
~~~

Finally the Matches method can be called to check if the input currently in the buffer matches a
given Move. We do this by looping backwards over the buffered input and the specified move’s
InputSequence, if at some point during the loop the keys do not match, we break out of the loop and
return false. If that didn’t happen that means we have a match, we clear the buffer to prevent it
from using the same input again for matching and return true.

If we don’t want to be one frame behind on the InputBuffering to check for matching moves we have to
use Unity’s LateUpdate method, as LateUpdate is always called after Update, we always receive the
most recent buffered input:

~~~ csharp
public class PlainUnityFighter : MonoBehaviour {
    private InputBuffering inputBuffering;

    ...

    private void LateUpdate() {
        foreach (Move move in BoxerData.Moves) {
            if (inputBuffering.Matches(move)) {
                Debug.Log("Move " + move.Type);
            }
        }
    }
}
~~~

Conceptually there are three independent parts to distinguish: buffering simultaneous key presses,
buffering sequential key presses, checking if the key presses in the buffer match the input sequence
of a move. In the implementation these parts are not separated, making them hard to write and hard
to understand. Why is this the case? To build the sequential buffer we need to know when to put the
merge buffer into the sequential buffer. To know this we rely on state: the time that a merge buffer
opened, state that conceptually belongs and is also used in the buffering of simultaneous input. The
same goes for the concept of matching input, we clear the sequential buffer if we match a move so
that we will not match against the same input again. The consequence is that other code that uses
the sequential input buffer doesn’t work if it reads the same input buffer after the matching is
done because the buffer is cleared before this code can do its work. To solve this problem we need
to coordinate clearing the buffer, which adds additional complexity on its own.  

# An introduction to Rx

Rx is a framework for .NET (and javascript and the JVM and a whole lot of other platforms) offering
"a natural paradigm for dealing with sequences of events."

Central to Rx is the `IObservable` class. The `IObservable` represents a stream of events. The
`IObservable`’s most important method is called `Subscribe` which lets you get events out of an
observable. This is what hello world looks like in Rx:

~~~ csharp
IObservable<String> obs = Observable.Return("hello world");
obs.Subscribe(value => Debug.Log(value));
~~~

In the first line we create an `IObservable`, a stream of events, with one event: a string with the
text `"hello world"`. In the second line we subscribe to that observable by passing in a delegate that
consumes the values produced by this observable. Once Subscribe is called the observable will start
producing its events for this subscription. In this case it produces the event "hello world" and
calls the delegate that we added with our subscription, passing in the event. The result is that
Debug.Log is called, printing the string "hello world" to the Unity debug console.

Now take a look at the following example:

~~~ csharp
IObservable<String> obs = Observable.Return("hello world");
IObservable<String> delayedObs = obs.Delay(TimeSpan.FromSeconds(2), scheduler);
delayedObs.Subscribe(value => Debug.Log(value));
~~~

The first line is the same as in the first example, we create an observable stream with one value:
"hello world". The Observable class has many methods for transforming itself into another
observable, one of which is a method called Delay that returns a new observable with all values from
the original observable delayed by a certain amount of time. Note that calling Delay does not change
the original observable itself, but returns a new observable. When we subscribe to this delayed
observable we will get all of its events two seconds after they are produced.

Let’s expand the example into something that begins to resemble the basis of a combat system.

By default, Unity provides a poll-based mechanism for processing input, like this:

~~~ csharp
public void Update() {
    if (Input.GetKey("d")) {
        player.MoveForward();
    }
}
~~~

Each frame we check if the "d" key is pressed, if it is we move the player forward.

We’re going to take Unity’s poll-based Input mechanism and convert it into an Rx Observable so that
we can observe when keys are pressed.

First, we create an enum called `KeyEventType` so we can indicate if an event is of type `KeyDown` event
or of type `KeyUp`. Then we create a class called KeyboardEvent which represents a specific instance
of a key press (KeyDown) or a key release (KeyUp). The keyCode variable represents the key being
pressed/released:

~~~ csharp
public enum KeyEventType { KeyDown, KeyUp }

public struct KeyboardEvent {
    public KeyEventType type;
    public KeyCode keyCode;
}
~~~

Second, we create a new `MonoBehaviour` called `RxKeyboard` with an Rx Subject and an array of Unity
KeyCodes that we want to poll. The Rx Subject is one of the many ways Rx allows you to create an
Observable. The Subject is in itself an Observer and an Observable. An Observer is a construct that
takes events as input, through a method called OnNext in order to publish those events through an
Observable. Since a Subject is both an Observer and an Observable we can add new KeyboardEvents by
calling OnNext() on it and we can observe those values by calling Subscribe on it:

~~~ csharp
public class RxKeyboard : MonoBehaviour {
    IEnumerable<KeyCode> _keys;
    ISubject<KeyboardEvent> _keyEvents;

    void Awake() {
        _keys = Enum.GetValues(typeof(KeyCode)) as KeyCode[];
        _keyEvents = new Subject<KeyboardEvent>();
    }
        ...
}
~~~

Each frame, we poll all the keys by calling `Input.GetKeyDown` and `Input.GetKeyUp` and create a new
`KeyboardEvent` each time we have a match, which we then publish on the keyEvents subject by calling
`OnNext` on it. Finally, we add a property called Events that exposes the keyboard events as an
observable to other scripts:

~~~ csharp
public class RxKeyboard : MonoBehaviour {

    ...

    public IObservable<KeyboardEvent> Events {
        get { return _keyEvents; }
    }

    // Update is called once per frame
    void Update() {
        foreach (KeyCode key in _keys) {
            if (Input.GetKeyDown(key)) {
                _keyEvents.OnNext(new KeyboardEvent(KeyEventType.KeyDown, key));
            }
            else if (Input.GetKeyUp(key)) {
                _keyEvents.OnNext(new KeyboardEvent(KeyEventType.KeyUp, key));
            }
        }
    }
}
~~~


To demonstrate how to use the newly created event stream: this is how you observe the keyboard
observable and print each event to the Unity Debug console:

~~~ csharp
[RequireComponent(typeof(RxKeyboard))]
class ExampleScript : Monobehavior {
    void Start() {
        var keyboard = GetComponent<RxKeyboard>().Events;
        keyboard.Subscribe(keyEvent => Debug.Log(keyEvent.type + ", " + keyEvent.key));
    }
}
~~~

After cranking at the keyboard for a while the Debug console might look like this:

~~~
KeyDown Q
KeyUp Q
KeyDown RightArrow
KeyUp RightArrow
KeyDown L
KeyUp L
...
~~~

A combat system implemented in Unity and Rx Now that we have the keyboard working through Rx we can
start implementing the fighting game input system.

The great thing about Rx is that you can transform and filter an Observable into just the shape you
want without losing any of the characteristics of an event. Let’s transform the initial
KeyboardEvents observable into an observable that only produces KeyDown events and only published
the KeyCode pressed instead of the complete KeyboardEvent:

~~~ csharp
IObservable<KeyCode> keyPresses = keyboard
    .Where(keyEvent => keyEvent.Type == KeyEventType.KeyDown)
    .Select(keyEvent => keyEvent.KeyCode);
~~~

All methods on an Rx observable return a new observable. The Where method filters an observable, and
returns a new observable where all KeyUp events are left out. The Select method transforms the
incoming KeyboardEvents by extracting the KeyCodes, because we don’t care about the event’s type
anymore, and returns a new observable. This process is easier to understand visually as a marble
diagram, where each line represents the observable that is derived from the one above:

<img width="690" src="/assets/images/street-fighter-1.png" />

Now that we have an observable of key presses we can start transforming it into buffers. The first
thing to do is to check for simultaneous key presses: we want to check which keys are pressed at the
same time. We do this by creating a buffer for all keys that are pressed within a 20 millisecond
time-frame.

~~~ csharp
TimeSpan bufferTime = TimeSpan.FromMilliseconds(20);
IObservable<KeyCode> closeBuffer = keyPresses.Delay(bufferTime, scheduler);
IObservable<IList<KeyCode>> simultaneousKeyPresses =
keyPresses.Buffer(bufferClosingSelector: () => closeBuffer);
~~~

Once again, this process is much easier to understand visually:

<img width="690" src="/assets/images/street-fighter-2.png" />

There are three observables into play here:

- keyPresses - the original observable, representing the keys pressed on the keyboard.
- closeBuffer - an observable that denotes the closing of a buffer 20 milliseconds after a key is pressed.
- simultaneousKeyPresses - an observable representing keys that are pressed within a 20 millisecond
  time-frame. This observable is built using Rx’s Buffer method. The Buffer method
  (http://www.introtorx.com/Content/v1.0.10621.0/13_TimeShiftedSequences.html#Buffer) immediately
  opens a buffer and calls the bufferClosingSelector to know when it should close the buffer. When
  the buffer closes the buffered input is published and a new buffer is started.

By transforming the keyboard event observable we have created a new observable that produces
buffered key presses, hence our new observable produces lists of keys where a list represents a
completed buffer.

Buffering sequential input is almost the same as buffering simultaneous input with two notable
differences: the buffer window is extended with 400 milliseconds each time a key is pressed, and we
want to produce a value each time a key is pressed, instead of waiting for the buffer to be closed
for a value to be produced. Let’s look at how we create a buffer where closing is delayed each time
    a key press is produced:

~~~ csharp
IObservable<IObservable<long>> bufferTimeOutStream =
    simultaneousInput.Select(keys => Observable.Timer(BufferTimeOut));
IObservable<long> closeBuffer = bufferTimeOutStream.Switch();
~~~

We take the previously created stream and use it to create a buffer timeout stream. The buffer
timeout stream is unlike anything we’ve seen before: it’s a observable of observables. An observable
of observables is almost like an observable of regular values except that it provides us with a bit
more flexibility: every time the `simultaneousKeyStream` produces a value we create a new Timer
observable that will produce a value after 400 milliseconds have passed. Then we define a
`closeBuffer` stream that takes the timeout stream and calls Switch on it. Switch is a special
operator that takes an Observable of Observables as input and outputs a regular Observable that
always represents the latest observable produced by the input. More concretely, every time a key is
pressed a timer observable is created then switch will use that, if another key is pressed after
that then another timer observable will be created and switch will use that timer and discard the
first one. Visually we can represent this process as follows:

<img width="690" src="/assets/images/street-fighter-3.png" />

Now we have an observable that produces a value that we can use to close a buffer, allowing us to
finish the implementation by adding a Window operation:

~~~ csharp
IObservable<IObservable<IList<KeyCode>>> sequentialInput =
    simultaneousInput.Window(windowClosingSelector: () => closeBuffer);
~~~

We create a Window from our simultaneous keys by calling the Window method and providing the
closeBuffer observable that we have just created. Rx’s Window method is similar to the Buffer method
except that it exposes the buffer itself as an `IObservable` instead of a list. On that `IObservable` a
value is produced each time the buffer is updated, which is exactly where Window differs from
Buffer: it produces a value each time a value is published on a buffer instead of waiting for the
buffer to be closed before producing a list of values. This characteristic allows us to check if an
active buffer matches a Move immediately after a key is pressed for the purpose of responsive
gameplay, like this:

~~~ csharp
IObservable<Move> moves =
    sequentialInput.SelectMany(inputBuffer => {
        IObservable<InputSequence> bufferedInput =
            inputBuffer.Scan(new InputSequence(), (inputSequence, key) => {
                inputSequence.Value.Add(key);
                return inputSequence;
            });

        return bufferedInput.SelectMany(inputSequence => {
            foreach (Move move in CombatSystemData.Moves) {
                if (Matches(inputSequence, move)) {
                    inputSequence.Value.Clear();
                    return Observable.Return(move);
                }
            }
            return Observable.Empty<Move>();
        });
    });
~~~

Each time we receive a `sequentialKeys` variable we create a `bufferedInput` observable for it by using
the Scan method. Scan takes an initial value and an aggregator function as input, each time a value
is produced the aggregator function is called passing in the aggregated value and the newly received
value. This way we can produce an InputSequence each time a new buffer is opened and keep updating
that InputSequence as new values are produced on the buffer.

We then use that `bufferedInput` variable again to see if it matches a move each time the buffer is
updated by checking if it matches a move. If there is a match we produce a new observable of one
value containing the move, if no match was found we produce an observable without any values on it,
Since we return an observable inside another observable we again have an Observable of Observables
while instead we want to have an observable of moves. To solve this issue we use `SelectMany` which
takes all values produced in all the nested observables and combines them into a regular
observable. Again, a visual representation of how values flow through all observables will most
certainly help to understand what we have just coded:

<img width="690" src="/assets/images/street-fighter-4.png" />

Finally to complete our exercise we subscribe to the `MoveObservable` to let the player actually
perform a move:

~~~ csharp
moveObservable.Subscribe(move => {
    player.PerformMove(move.Type);
});
~~~

Looking back at the problems that the polling implementation introduced, we can conclude that we
tackled them in the following way:

Rx allows us to work with time very directly by programming events to be produced at a specific
moment to create and close buffers instead of keeping track of time through state variables.  The
input buffering concepts are not intertwined: buffering simultaneous input is completely isolated
and doesn’t know anything about sequential input buffering. Sequential input buffering only knows
about simultaneous input buffering through the values it produces but not its internals. Likewise,
we get sequential input buffers through the Window observable and then aggregate and match Moves on
the fly without looking at or changing the internals of the sequential buffering code.  Conclusion
With Rx we get programming constructs that allow us to describe game behavior over time. We’ve seen
that this makes code for buffering input for a fighting game much more concise and readable than its
traditionally programmed ‘polling’ counterpart.

Rx allows us to break the concept of input buffering apart into small and understandable pieces of
code that can be composed together. No variables are needed for the purpose of time-tracking since
events can be configured to be produced at exactly the right moment.

All code can be found on [Github](https://github.com/fversnel/UnityRxExamples), feel free to run the
code, play around with it and so on.

To learn more about Rx, check out <http://www.introtorx.com>.

# Notes on performance

Rx .NET implementation is tailored for concurrency but Unity is single-threaded.
Lambda’s create garbage?
Notes on scheduling
Rx uses the concept of Scheduling underneath to plan the execution of actions in the future.
Using the default scheduler introduces problems when used with Unity as it uses up Unity’s thread to do the scheduling.
To solve this problem a specific Scheduler for Unity has been written that plays nice with Unity’s threading model.

Scheduling is a wide and complex concept that deserves an article on its own so we won’t go into the
details here. Just remember that the Unity needs a specific scheduler to plan work ahead of time,
like Observable.Timer does.  

# Future work

To verify Rx’s usefulness in the field we could go about and do some more case studies that stress
the concepts of Reactive programming a bit more by: Extending the combat system to:

- Adding a combo system, allowing moves to be linked.
- Adding a blocking and hit/recovery system.
- Add another example, that uses multiple input sources instead of one (the keyboard)

# Links

- <https://github.com/fversnel/UnityRxExamples>
- <http://www.introtorx.com>
- <http://www.reactivemanifesto.org>

# Appendix: How to setup Rx in Unity

To get Rx up and running download the installer
[here](http://www.microsoft.com/en-us/download/details.aspx?id=28568) and select ‘Rx for .NET 3.5’.
Then copy `C:\Program Files (x86)\Microsoft Reactive Extensions
SDK\v1.0.10621\Binaries\.NETFramework\v3.5\System.Reactive.dll` into your Unity project’s
`Assets\Plugins`
folder. Now you should then be able to use the System.Reactive.Linq, System.Reactive.Concurrency
namespace in your code.
