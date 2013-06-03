---
layout: post
title: "GTD and TDD"
date: 2013-05-02 00:29
comments: true
categories: [TDD, Java, GTD, uncertainty]
---

My apprenticeship at 8th Light began with the [Three Laws of TDD](http://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd):

> - Don't write any production code unless it makes a failing test pass.
> - Write the most minimal test that is sufficient to fail.
> - Don't write any more production code than [necessary and sufficient](http://plato.stanford.edu/entries/necessary-sufficient/) to pass a failing test.

My first project obeying these new commandments is a rewrite of [Tic-Tac-Toe](https://github.com/ecmendenhall/clojurescript-tic-tac-toe) in Java, a language completely new to me. Like all new projects, I began in a wilderness of error. IntelliJ was totally unfamiliar. Type declarations felt fussy and foreign. I just wanted to map over an array but had to settle for another for loop. And yet test by test, things slowly started to work.

After three days, I can't claim to know Java. But I can claim that I know _my_ Java works.

The clarity and certainty that come from good tests are feelings I've experienced before. Over the past few years, the practices and principles of [Getting Things Done](http://www.43folders.com/2004/09/08/getting-started-with-getting-things-done) have transformed the way I work, think, and act. (Have I told you about our church? Would you like to read some [inspirational literature](http://www.amazon.com/Getting-Things-Done-Stress-Free-Productivity/dp/0142000280)?) Here is a formulation of the basic idea that might look familiar:

> - Don't spend time and effort on anything but a project's next incomplete action.
> - Choose the smallest [next action](http://www.43folders.com/2004/09/17/next-actions-both-physical-and-visible) sufficient to accomplish something useful.
> - Don't spend more time or effort than necessary to finish an incomplete action.

To the uninitiated, writing tests and making lists might seem inflexible, [robotic](https://en.wikipedia.org/wiki/Three_laws_of_robotics), and a little dorky. But TODO lists and test suites are really tools that externalize uncertainty. The vague sense that I really should do that thing becomes a clear action that will be in my inbox later. The nagging fear that a method might misbehave becomes proof that it does and will always do exactly what I expect.

I skated through school and college on cleverness and a keen balance of terror. And it worked! I wrote code that seemed to do what I told it long before I started writing tests. (Tests aren't necessary for working code: they're sufficient to prove that code works). But to claim that other ways work just fine is to miss the point. GTD and TDD are systems that radically, ruthlessly eliminate uncertainty—not because the rules are necessary in the strictest sense, but because they are sufficient to get things done, make things work, and do it all with a clear mind.