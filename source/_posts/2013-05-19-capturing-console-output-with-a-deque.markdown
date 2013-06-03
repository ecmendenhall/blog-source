---
layout: post
title: "Capturing console output with a deque"
date: 2013-05-19 14:43
comments: true
categories: [Java, TDD, testing recipes]
---

Redirecting stdout to a new `PrintStream` is an [easy way](http://ecmendenhall.github.io/blog/blog/2013/05/05/testing-console-output-with-junit-and-infinitest/)
to test simple console output in Java. But as my Tic-Tac-Toe game has
grown more complex, the tests I wrote using this pattern have started
to stink. There's a lot of duplicated code (create the stream,
redirect, tear down with each test), each test uses a hand-constructed
string filled with finicky newline characters, and tests are prone to
break when unrelated view components change the way they print to the
screen. Inspired by my [input
queue](http://ecmendenhall.github.io/blog/blog/2013/05/15/mocking-user-input-with-a-queue/),
I created an `OutputRecorder` class that extends `PrintStream` and
captures output string by string for later playback:

{% codeblock OutputRecorder.java lang:java %}
public class OutputRecorder extends PrintStream {
    private Deque<String> outputStack;

      public OutputRecorder(OutputStream outputStream, boolean b,
      String s) throws UnsupportedEncodingException {
          super(outputStream, b, s);
          outputStack = new LinkedList<String>();
        }

        private void catchOutput(String output) {
            outputStack.addFirst(output);
        }

        public String popLastOutput() {
            return outputStack.removeFirst();
        }

        public String popFirstOutput() {
            return outputStack.removeLast();
        }


        public String peekLastOutput() {
            return outputStack.peekFirst();
        }

        public String peekFirstOutput() {
            return outputStack.peekLast();
        }

        public void discardLastNStrings(int n) {
            for (int i=0; i < n; i++) {
                popLastOutput();
            }
       }

       public void discardFirstNStrings(int n) {
            for (int i=0; i < n; i++) {
                popFirstOutput();
            }
       }

       public void replayAllForwards() {
            String output = popFirstOutput();
            int i = 0;
            while (output != null) {
                System.out.println("Element" + i + ":");
                System.out.println(output);
                i += 1;
                output = popFirstOutput();
            }
       }

       public void replayAllBackwards() {
            String output = popLastOutput();
            int i = outputStack.size();
            while (output != null) {
                System.out.println("Element" + i + ":");
                System.out.println(output);
                i -= 1;
                output = popLastOutput();
            }
       }

       @Override
       public void println(String output) {
            catchOutput(output);
       }

       @Override
       public void print(String output) {
            catchOutput(output);
       }

}
{% endcodeblock %}
Since the recorder stores strings in a
[deque](https://en.wikipedia.org/wiki/Deque),
it's easy to replay output in forward or reverse order. Now a test
like this:

{% codeblock lang:java %}                              
@Test
public void controllerShouldPassErrorMessageToViewOnInvalidMove() {
    view.pushInput("middle center");
    view.pushInput("middle center");

    exit.expectSystemExitWithStatus(2);
    controller.newGame();

    System.setOut(outputStream);

    controller.playRound();
    String expected = yourMove + "\n" + xInCenter.toString() + "\n";

    assertEquals(expected,
    output.toString());

    System.setOut(stdout);
    view.clearInput();
}

{% endcodeblock %}                              

Become a little friendlier...

{% codeblock lang:java %}
@Test
public void controllerShouldPassErrorMessageToViewOnInvalidMove() throws GameOverException {
    view.enqueueInput("middle center");
    view.enqueueInput("middle center");

    controller.newGame();

    System.setOut(outputRecorder);

    try {
        controller.playRound();
    } catch (NoSuchElementException e) {

          outputRecorder.discardFirstNStrings(4);
          String output = outputRecorder.popFirstOutput();

          assertEquals("Square is already full.", output);
    }
    System.setOut(stdout);
    view.clearInput();
}
{% endcodeblock %}

The utility might not be immediately obvious, but capturing output
string by string has already put an end to tracking down small
differences between expected and actual output that come from an extra
space or misplaced newline.                               
