---
layout: post
title:  "About Scrum (and Agile)"
date:   2018-04-09 17:00:00 +0100
published: true
categories: ['agile', 'scrum', 'software process']
---

The sometimes religious-like following of this movement has always scared me a little bit. 
I was really keen on reading Bertrand Meyer's 'Agile: the good, the hype, the ugly' because I 
felt I didn't have enough knowledge about Agile to criticise it.

That being said, Agile has brought the industry great benefits while at the same time making things worse,
according to Meyer. This book serves as a guide for learning about various Agile methods (XP, Scrum, etc.) 
and which elements can truly benefit a software project and which things can harm a project.

I will cover some of the Agile practices covered in the book and will tell you some of my
own experiences and opinions about Scrum related to the criticisms in this book.

# Refactoring

Changing the structure of the code without changing the behavior to accomodate for new features
is crucial to iterative development. New features will challenge the existing structure and
show cracks in the original design. To fix this the design, code needs to change
i.e. needs to be refactored. 

Refactoring is an important activity often not appreciated
enough by non-developers because it doesn't seem to generate immediate business value. Not refactoring
also has a cost though. It will become harder to add more features in the future, due to the fact
that the current design was built on an older model: an older view of the world when life was simpler,
so to speak.

# Sprints, short iterations, and a focus on short-term

Meyer's biggest beef is not with the short iteration cycle per se, but the fact that Agile is
anti-anything upfront. So no upfront design, upfront architecture or anything else really.

Focussing on the short-term has many benefits. You get a deliverable every few weeks, software
doesn't remain on the shelf for long, you can iterate on features by releasing early and often.
When something's not right the first time around, you just refactor the code to conform to the new
requirements.

For some things this approach just doesn't work and is actually harmful. Any application relies
on some kind of architecture. The architecture cannot be constructed haphazardly on a case-by-case
basis because this would result in a constant need for refactoring. Refactoring would become so
expensive that the case for upfront design for these types of activities becomes very clear. 

Not just architecture but many activities require larger-than-sprint vision, planning, and
studying. The fact that Agile doesn't want you to do any of that work upfront means it has to be
done in a sprint, which can very well harm the project.

# User stories

A system built on user stories will always be geared towards what the user will be doing with the system.
By definition these stories will be about the concrete instead of the abstract. Not having any general
requirements for the system means that it will be much harder to build proper abstractions that
allow you to cover a range of user stories.

From what I've experienced, great programmers design abstractions by learning the system and domain really well,
and then trying to fit user stories with the existing abstractions, or they create new abstractions if
the current ones are insufficient. It all comes back to the no upfront design mantra that can really
harm a project. Every successful project needs a few visionaries that will look beyond 
the user stories. Unfortunately Agile doesn't seem to provide any tools for them.

# Collective code ownership

Collective code ownership means that everyone is a generalist and should be able to change all code.
While this sounds great, for big projects it isn't feasible. Some people tend to specialize in
databases, while others specialize in UI. It would be silly to let the database specialist write
some mission-critical UI and let the UI expert write a complicated database query. You would get an
unoptimized query and a hard to maintain piece of UI code in return.

# Pair programming

Pair(ed) programming involves two programmers working on the same computer, with one of them typing
and the other paying attention to the overall structure of the code.

This usually doesn't work in my experience. I tend to think quite differently from other programmers
which means that the code I write before it's finalized might be hard to comprehend simply because
the other programmer can't follow my thoughts. Having to explain your own thoughts while they're
still in development is close to impossible. I'd much rather discuss the code after it's finished.

Meyer's adds that it is actually harmful when combining paired programming with teaching: the
teacher has to juggle between getting the job done in a reasonable amount of time and teaching the
other (paired) programmer. It's better to have a separate activity for teaching where speed is not
an important factor and learning is.

# Planning poker

Estimating development is critical to planning and keeping the costs of a software project under
control. Scrum proposes a technique for estimating development called planning poker. In theory,
everyone gets an equal vote in estimating the complexity of a story (through story points).
Afterwards individual estimates are discussed and concensus is reached to define the final estimate.

In practice, more experienced programmers will often estimate higher than less experienced ones
because they know every nook and cranny of the system foreseeing all kinds of difficulty. Novices
will not see this complexity and will estimate lower. The danger being that the expert opinion gets
overruled by the sheer number of novice estimates. Moreover, one might wrongly conclude the expert
is simply not a very good programmer exactly because of this higher estimate.

# The importance of testing

Automated testing is one of the best things Agile brought us and I tend to agree. Having good tests
to make sure you didn't break anything is extremely important. Especially since refactoring is such
an important activity in a sprint-based process.

# Reviewing code

Code reviews are great! Having another developer check your design and code keeps everyone
aligned. Not only that, it's an extra insentive for programmers to keep up with quality standards.
If you're code is not up to par it will simply not pass the review. Developers will keep this
in mind while coding and won't take unnecessary shortcuts.

# Concluding

Agile brought us many benefits but also harmed software development. This book explains, mostly
just by using logical reasoning, that some of the Agile practices are incomplete or wrong, while
others are great and have proven their worth.

Meyer's biggest problem is that Agile doesn't allow for upfront design. A lack of oversight
or means to contribute, change, improve the overall structure of the software is a real problem.
We need solid structures to build on, stuff that will last for a while. We need people that think
about, accomodate, and design for change rather than designing the code such that it's good enough
for one particular story.

I definitely recommend people involved in software development, not just programmers, to read this book.
