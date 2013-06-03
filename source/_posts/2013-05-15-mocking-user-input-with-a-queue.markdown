---
layout: post
title: "Mocking User Input with a Queue"
date: 2013-05-15 14:53
comments: true
categories: [Java, TDD, testing recipes] 
---
Over the course of my Tic Tac Toe project, I've needed to test against
user input several times. As a newcomer to [mock object
patterns](http://ecmendenhall.github.io/blog/blog/2013/05/12/a-mock-object-antipattern/),
coming up with good solutions to these testing dilemmas has been one
of my biggest challenges.

There are plenty of [heavy-duty
tools](http://stackoverflow.com/questions/3833840/mock-object-libraries-in-java)
for mocks and fakes in Java, but I'd like to stick with my own
solutions as long as possible, since writing them myself has been
enlightening.

Here's a solution I came up with today to simulate full Tic Tac Toe
games with two human players: a mock View object that returns fake input
from a queue. By pushing mock input onto the queue during test setup,
I can configure games in advance and replay them later. Here's the
very simple mock View object:

{% codeblock MockTerminalView.java lang:java %}
import java.util.NoSuchElementException;
import java.util.Queue;
import java.util.concurrent.LinkedBlockingQueue;

public class MockTerminalView extends TerminalView {

    private Queue<String> inputQ = new LinkedBlockingQueue<String>();

    public void enqueueInput(String fakeInput) {
        inputQ.add(fakeInput);
    }

    public void clearInput() {
        inputQ.clear();
    }

    @Override
    public String getInput() {
        try {
            return inputQ.remove();
        } catch (NoSuchElementException e) {
            return "";
        }
    }
}
{% endcodeblock %}

And here's an example test that plays through an entire game and exits
when Player 2 wins (comments added for some context): 

{% codeblock GameControllerTest.java lang:java %}
@Test
public void gameShouldEndOnWin() {
    exit.expectSystemExit();

    controller.newGame();

    // Select two human players
    view.enqueueInput("h");
    view.enqueueInput("h");

    controller.setUp();

   
    view.enqueueInput("middle center"); // Player 1's first move
    view.enqueueInput("top left");      // Player 2's first move
    view.enqueueInput("top right");     // Player 1 goes for the diagonal...
    view.enqueueInput("middle left");   // Player 2 goes for the column...
    view.enqueueInput("lower right");   // Player 1 chokes!
    view.enqueueInput("lower left");    // Player 2 wins! What an upset! 
    
    controller.startGame();
}
{% endcodeblock %}

I'm sure I'll discover the shortcomings of this approach sooner or
later, but for now it's a pretty good way to test events inside
the main game loop.
