---
layout: post
title: "Refactoring examples: Duplicated code"
date: 2013-06-02 20:56
comments: true
categories: [Java, refactoring]
---
I've spent the last couple weeks working through [Martin Fowler's](http://martinfowler.com/) [_Refactoring_](http://www.amazon.com/Refactoring-Improving-Design-Existing-Code/dp/0201485672/ref=sr_1_1?ie=UTF8&qid=1370238910&sr=8-1&keywords=refactoring), which includes a taxonomy of useful solutions to common code smells. It's an excellent book, but I found it hard to recognize and remember some of the patterns without digging into my own  code. Over the next few posts, I'll describe some of the smells and refactors I found in my Tic Tac Toe game (as much to commit them to my own memory as to share them with others). First up, the most common code smell of all: duplication.

Despite using a [superclass](https://github.com/ecmendenhall/Java-TTT/blob/master/test/com/cmendenhall/tests/TicTacToeTest.java) to store static methods and data
used by all my tests, I wasn't using inheritance to create universal setup and teardown methods. Although many of my tests used a similar pattern to capture output, none were quite alike, and all of them were pretty messy. Here's an example:

{% codeblock lang:java %}
@RunWith(JUnit4.class)
public class GameControllerTest {

    private MockTerminalView view = new MockTerminalView();
    private GameController controller = new GameController(view);

    private final PrintStream stdout = System.out;
    private final ByteArrayOutputStream output = new ByteArrayOutputStream();
    private PrintStream outputStream;
    private OutputRecorder outputRecorder;
  
    @Before
    public void setUp() throws Exception {
        loadViewStrings();
        outputStream = new PrintStream(output, true, "UTF-8");
        outputRecorder = new OutputRecorder(output, true, "UTF-8");
        Player playerOne = new HumanPlayer(X);
        Player playerTwo = new MinimaxPlayer(O);
        controller.setPlayerOne(playerOne);
        controller.setPlayerTwo(playerTwo);
    }

    @Test
    public void controllerShouldStartNewGame() {
        System.setOut(outputRecorder);

        controller.newGame();

        assertEquals(welcome, outputRecorder.popFirstOutput());
        assertEquals(divider, outputRecorder.popFirstOutput());

        System.setOut(stdout);
    }

    @Test(expected = GameOverException.class)
    public void controllerShouldEndGameOnRestartIfInputIsNo() throws GameOverException {
        System.setOut(outputStream);

        view.enqueueInput("n");

        controller.restartGame();
        assertEquals(playAgain, output.toString());

        System.setOut(stdout);
    }
} 
{% endcodeblock %}

These tests use both a plain `PrintStream` and my custom `OutputRecorder`, though they don't really need to. Some setup is done in the fields, and some during `setUp()`, though there's no clear reason why. And the same lines setting up and tearing down the recorder are repeated across lots of tests. The smell here is duplicated code, which Fowler calls "number one in the stink parade." To solve it, I'll start by extracting a method, and then extracting a superclass.

First, I'll create an empty `TicTacToeTest` class with empty setup and teardown methods.
{% codeblock lang:java %}
@RunWith(JUnit4.class)
public class TicTacToeTest {

  @Before
    public void setUp() {
      }

        @After
          public void cleanUp() {
            }
            }
            {% endcodeblock %}

            Now, extract a `setUpRecorder()` method including all the setup-related lines:

            {% codeblock lang:java %}
            private OutputRecorder recorder;

            private void setUpRecorder() throws UnsupportedEncodingException {
                    PrintStream stdout = System.out;
                    ByteArrayOutputStream output = new ByteArrayOutputStream();
                    recorder = new OutputRecorder(output, true, "UTF-8");
            }
            {% endcodeblock %}

            Then, a `startRecorder()` method to replace calls to `System.setOut()`:

            {% codeblock lang:java %}
            private void startRecorder() {
              System.setOut(recorder);
              }
              {% endcodeblock %}

              This method might seem short, but I find `startRecorder()` much easier to read. Now that these methods are implemented, I can plug them into the tests:

              {% codeblock lang:java %}
              @Before
              public void setUp() throws Exception {
                  loadViewStrings();
                  
                  setUpRecorder();
                  
                  Player playerOne = new HumanPlayer(X);
                  Player playerTwo = new MinimaxPlayer(O);
                  controller.setPlayerOne(playerOne);
                  controller.setPlayerTwo(playerTwo);
              }

              @Test
              public void controllerShouldStartNewGame() {
                  startRecorder();

                      controller.newGame();

                      assertEquals(welcome, outputRecorder.popFirstOutput());
                      assertEquals(divider, outputRecorder.popFirstOutput());

                      System.setOut(stdout);
                  }
                  {% endcodeblock %}

                  The last line of the second test was meant as a mini-teardown, to reset stdout before the next test. I wrote it before I understood the JUnit execution, in which each test runs in its own environment. It survived around 80 commits, but it doesn't actually do anything, so I can safely delete it. (I know because none of the tests fail afterwards).I know because none of the tests fail afterwards. In fact, this whole refactor should be possible while keeping the tests green.

                  After searching for usages of the old pattern and replacing them with the new methods, there's just one thing left: pull up the new methods to a Test superclass. Here's the result:

                  {% codeblock lang:java %}
                  public class TicTacToeTest {

                      protected OutputRecorder recorder;

                      protected void setUpRecorder() throws UnsupportedEncodingException {
                          ByteArrayOutputStream output = new ByteArrayOutputStream();
                          recorder = new OutputRecorder(output, true, "UTF-8");
                      }

                      protected void startRecorder() {
                          System.setOut(recorder);
                      }

                      @Before
                      public void recorderSetUp() throws UnsupportedEncodingException {
                          setUpRecorder();
                      }

                  }
                  {% endcodeblock %}

                  This looks like a simple refactor, but my tests are now much cleaner. More important, if I need to make a change to the `OutputRecorder` class, it will propagate through all tests.
layout: post
title: "Refactoring examples: Duplicated code"
date: 2013-06-03 01:32
comments: true
categories: 
---
