---
layout: post
title:  "Game programming with clojure.spec (Baba is you)"
date:   2019-04-01 17:00:00 +0100
published: true
categories: ['game development', 'software design', 'clojure']
---

Have you played "Baba is you" yet? It's a fantastic logic puzzle game, one
that I highly recommend. If don't know the game please watch a 
[video](https://www.youtube.com/watch?v=z3_yA4HTJfs)
on it as this post will be about how we implement the game using the Clojure
programming language and a library called 
[clojure.spec](https://clojure.org/guides/spec).

As you saw in the video Baba is a game with simple rules. Rules that can be bend
by the player. You walk around a tile based map pushing objects to change rules to 
reach the level's goal.

How would one implement this game I started thinking... very soon I came
up with a program that heavily relies on clojure.spec to do all the work.

# clojure.spec

If you don't know what clojure.spec is I recommend reading the 
[guide](https://clojure.org/guides/spec), but I'll try to briefly explain why I want
to use it.

clojure.spec allows you to describe your data (e.g. a level of a game) in an
incredible amount of detail. You can state for example
that your level is a 2d vector containing
at least five cells of water, a player, and the number 42.

Furthermore once you've written a spec you can match it against a data structure and:

1. It will validate your data structure.
2. It will annotate your data structure such that you can easily walk over it, 
matching individual elements.
Meaning a data structure that looks like: `[:water 42 :baba]` might be transformed to:
`[[:game-object :water] [:number 42] [:player :baba]]`

It can do quite a bit more, but we'll focus on these two features for now.

# Spec'ing a level

Let's start by describing what a Baba level looks like in clojure:

```clojure
(def example-level
  [[:wall  :empty     :word/baba :word/is   :word/you :empty :wall]
   [:wall  :empty     :empty     :baba      :empty    :empty :wall]
   [:wall  :wall      :wall      :wall      :wall     :flag  :wall]
   [:empty :word/flag :word/is   :word/win  :grass    :grass :empty]])
```

This level has the following properties:

1. It consists of cells that are either baba, a word, a wall, a flag, a patch of grass, or nothing at all.
2. It contains two rules:
    1. `[:word/baba :word/is :word/you]` - meaning you control `:baba`
    2. `[:word/flag :word/is :word/win]` - meaning when you touch the `:flag` you win.

This level is so easy you can actually walk over to the flag to win the level. 
A perfect example for our first implementation of the game.

Let's see if we can write some simple specifications for Baba cells to get started.

```clojure
(ns org.fversnel.baba.spec
  (:require [clojure.spec.alpha :as s]))

(s/def ::game-object
  #{:baba :wall :flag :grass})
```

There. We have our various non-word game-objects. 
This specification means that we allow game objects to be either one of: 
`:baba`, `:wall`, `:flag`, or `:grass`.

Next up are the words. 
They're are bit more complicated:

```clojure
(s/def ::subject
  #{:word/baba :word/flag})
(s/def ::verb
  #{:word/is})
(s/def ::object
  #{:word/you :word/win})
```

The words eventually have to form sentences. 
Sentences in Baba consist of: subject, verb, object.
So we make the same distinction.

Let's define a sentence:

```clojure
(s/def ::sentence
  (s/cat
   :subject ::subject
   :verb ::verb
   :object ::object)))
```

At this point I started loving how much clojure.spec 
allows you to write code that is very close to what the intent of your program is.
Look at this: a sentence is always a sequence (`cat`) of a `::subject`, then a `::verb`, 
and then an `::object`. Beautiful!

We also have to define individual cells. Cells can contain any type of legal thing,
so both words, game objects, but they can also be empty:

```clojure
(s/def ::cell
  (s/or
   :word 
   (s/or 
     :subject ::subject
     :verb ::verb
     :object ::object)
   :empty #{:empty}
   :game-object ::game-object))
```

Just more of the same really. Now we have to compose `::cell` and `::sentence` together
to write a specfication for our example level.
We first want to match sentences, and if the cell
does not belong to a sentence we want to match it as a normal cell. 
This will allow us later to extract the sentences and derive the game's rules from them.

Here's how we match either a cell or a sentence:

```clojure
(s/def ::row
  (s/alt
    :sentence ::sentence
    :cell ::cell))
```

We can match `[:word/flag :word/is :word/win]`, but also `:flag`. 
It will not allow us to match an entire row of the level so we need to add something else:

```clojure
(s/def ::row
  (s/*
   (s/alt
    :sentence ::sentence
    :cell ::cell)))
```

With `s/*` we say: match a sequence of mixed values consisting of sentences and 
regular cells where the first rule (`:sentence`) is applied first. 
Since spec allows for ordering of rules we make sure that we always match sentences first
and if that fails, we try for cells. If it were the other way around (matching cells first),
we would never match sentences as the sentence's components are also regular cells.

Let's try to match a row of a Baba level using our new and shiny specification:

```clojure
=> (s/conform ::row [:wall :empty :word/baba :word/is :word/you :empty :wall])

[[:cell [:game-object :wall]]
 [:cell [:empty :empty]]
 [:sentence {:subject :word/baba, :verb :word/is, :object :word/you}]
 [:cell [:empty :empty]]
 [:cell [:game-object :wall]]]
```

As you can see it not only validated that the row was according to the specification
it also annotated each element of the row according to the rules in the specification.
Structuring it as a vector where the first element is the type of the rule that was matched,
in our case: `:cell` and `:sentence`. The sentence is then split up further where each
individual element of the sentence (`:subject`, `:verb`, `:object`) is put at the appropriate key in a map.

To match an entire level we only have to add one extra level of nesting, like so:

```clojure
(s/def ::level
  (s/coll-of ::row))
```

This is what happens when we try to conform the entire level:

```clojure
=> (s/conform ::level example-level)

[[[:cell [:game-object :wall]]
  [:cell [:empty :empty]]
  [:sentence {:subject :word/baba, :verb :word/is, :object :word/you}]
  [:cell [:empty :empty]]
  [:cell [:game-object :wall]]]
 [[:cell [:game-object :wall]]
  [:cell [:empty :empty]]
  [:sentence {:subject :word/baba, :verb :word/is, :object :word/you}]
  [:cell [:empty :empty]]
  [:cell [:game-object :wall]]]
 [[:cell [:game-object :wall]]
  [:cell [:game-object :wall]]
  [:cell [:game-object :wall]]
  [:cell [:game-object :wall]]
  [:cell [:game-object :wall]]
  [:cell [:game-object :wall]]
  [:cell [:game-object :wall]]]
 [[:cell [:empty :empty]]
  [:sentence {:subject :word/wall, :verb :word/is, :object :word/stop}]
  [:cell [:game-object :grass]]
  [:cell [:game-object :grass]]
  [:cell [:empty :empty]]]]
```

# Using the spec


To top it off we'll write a function that takes a level and returns only the its
rules:

```clojure
(defn rules [level]
  (let [match-sentence
        (filter
         (fn [[type & _]]
           (= type :sentence)))]
    (into
     #{}
     (comp cat match-sentence)
     level)))
```

Running it against our conformed level will produce:

```clojure
=> (rules (s/conform ::level example-level))

#{[:sentence {:subject :word/baba, :verb :word/is, :object :word/you}]
  [:sentence {:subject :word/wall, :verb :word/is, :object :word/stop}]}
```

We're still far away from implementing the entire game of course but the point
I want to make is that the basics of the game of Baba can be written down
as a simple clojure specification. 
There's absolutely no complicated logic involved here, 
no complex algorithm to parse or transform the level. 
Just a spec and a simple function for extracting some rules out the level.

I hope this will be a stepping for me and for others towards simpler software
and game design.

