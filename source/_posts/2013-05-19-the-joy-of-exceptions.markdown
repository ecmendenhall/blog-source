---
layout: post
title: "The joy of exceptions"
date: 2013-05-19 21:31
comments: true
categories: [TDD, Java, testing recipes]
---
One more recipe for the TDD cookbook before I move on to more
interesting things. You might have noticed a new `GameOverException`
and a try-catch block in the final example of my last post.

Before adding a game over exception, methods that checked for final
game states directly called `System.exit()`. Testing this proved
difficult, and I ended up importing a [third-party library](http://stefanbirkner.github.io/system-rules/) of extra
JUnit rules. This week, I refactored them to throw a custom exception: 

{% codeblock TerminalView.java lang:java %}
public class TerminalView implements View {

    public void endGame() throws GameOverException {
        throw new GameOverException("Game over.");
    }

}
{% endcodeblock %}

This exception will bubble up until it's caught in `Main`, which calls
`System.exit()` on behalf of any method that throws a
`GameOverException`. 

{% codeblock Main.java lang:java %}
public class Main {

    public static void main(String[] args) {
        try {

            View view = new TerminalView();
            GameController controller = new GameController(view);

            controller.newGame();
            controller.setUp();
            controller.startGame();
        } catch (GameOverException e) {
            System.exit(0);
        }
    }
}
{% endcodeblock %}

This is not only a [better practice](http://stackoverflow.com/questions/6171265/best-way-to-exit-a-program-when-i-want-an-exception-to-be-thrown),
but also provides a much better way to test methods that might detect
a completed game—just add a try/catch block to the tests!

Exceptions have come in handy elsewhere in my tests, too. Once the
controller starts a game, there's nothing to break the back-and-forth
game loop but a win, draw, or error. Figuring out how to test game
states without getting stuck in an infinite loop or loading up entire
games was a challenge. "If only there
were some special syntax for breaking normal control flow in special
situations," I wondered to myself more times than I'd like to admit.
Well, duh—use exceptions!  My mock view objects throw a `NoSuchElementException` when their input
queue runs empty. Catching this exception breaks the normal game flow
and allows me to access game state as soon as the fake input I'm
interested in has been sent to the controller. Here's an example: 

{% codeblock GameControllerTest.java lang:java %}
@Test
public void controllerShouldPassErrorMessageToViewOnInvalidInput()throws GameOverException {
    view.enqueueInput("invalid phrase");

    controller.newGame();

    System.setOut(outputRecorder);

    try {
        controller.playRound();
    } catch (NoSuchElementException e) {

        outputRecorder.discardFirstNStrings(1);
        String output = outputRecorder.popFirstOutput();

        assertEquals("That's not a valid board location.", output);

    }

    System.setOut(stdout);
    view.clearInput();
}
{% endcodeblock %}

Normally, `Controller.playRound()` will continue querying players for
moves until the game ends. But once this test catches the empty queue
exception, it tests against the expected output, which should show an
error message. Exceptions have proved extremely handy so far—as long as I remember
that they're in my control flow toolbox, too.
