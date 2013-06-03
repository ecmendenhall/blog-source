---
layout: post
title: "A mock object antipattern"
date: 2013-05-12 23:05
comments: true
categories: [TDD, Mock Objects]
---

For the most part, test-driven development has been a breeze so far.
In addition to
[enforcing](http://ecmendenhall.github.io/blog/blog/2013/05/10/enforcing-bottom-up-design/)
good habits, watching red tests turn green provides an extremely satisfying
dopamine kick every few minutes, all day. Testing simple objects and actions in my Tic Tac Toe game—things
like the board, player behavior, and the Minimax algorithm—was
straightforward. But testing the view and controller classes that glue
them together required a little more thought.

Writing tests required interrupting the game loop to check on the behavior of the view and
controller objects. To start, I added optional flag arguments to many
methods that would break the game loop so I could make assertions
about game state. I quickly came to realize that this was a bad
solution, and I cringed the next day when a chapter of _Clean Code_
described boolean flags as one of the most rancid code smells around.

I came across the concept of [test
 doubles](http://martinfowler.com/articles/mocksArentStubs.html)
and mock objects, and got the idea right away: create fake objects
 with the same methods as real ones, override their behavior, and use
 them as substitutes for their more complicated counterparts in unit tests. 

Or at least, I  _thought_ I got the idea. With my tests passing and my game working, I felt pretty good about my
project. But wiring up a code coverage tool showed that my view and
controller classes were only half covered by my tests. What went
wrong? As it turned out, I was testing my mocks! Here's the pattern I was following:

+ Create a subclass of the object you want to test.
+ Override or stub out methods to return predetermined output.
+ Write assertions against the behavior of the mock objects.

This meant, of course, that I was never actually testing the real
objects, but only the fake ones, as revealed by the code coverage data. Worse, all the tests
that I thought showed my code was working were essentialy tautologies.
This probably appears obvious to experienced TDD practitioners, but it
was surprisingly easy to fall into this antipattern. For the record,
here's how you should really use a mock object:

+ Create mocks of the objects that _interact with_ the one you want to
test.
+ Override or stub out methods to return predetermined output.
+ Write assertions against the behavior of the _real_ object
interacting with the test doubles.

In retrospect, this makes perfect sense. But I'll be going over all my
tests with a careful eye tomorrow. Sometimes the green light isn't
what it seems. 
