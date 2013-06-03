---
layout: post
title: "Testing Console Output with JUnit and Infinitest"
date: 2013-05-05 20:54
comments: true
categories: [Java, JUnit, Infinitest, Today In Yak Shaving]
---

I saved the view components of my Tic-Tac-Toe game for last. The
functionality I need to implement for a command line application is
pretty simple: the view should be able to print game boards and
messages to the terminal, prompt for user input and pass it off to a
controller, and not much more. But doing this the TDD way was harder
than I expected. Here are the solutions I found to two testing problems.

## Testing text output
Printing a board should be a one-liner, especially since I added a
`toString()` method to my `Board` objects. `System.out.println(Board)`
ought to work just fine. But wait—the tests come first.

Writing tests against printed terminal output isn't tough, but it does
require some setup. `System.setOut()` redirects Java's default output
to a byte stream of [your choice](http://docs.oracle.com/javase/7/docs/api/java/lang/System.html#setOut\(Java.io.PrintStream\)).
First, save the default `System.out` (It's a `PrintStream`), so you can switch back to stdout later.
Then, initialize a `ByteArrayOutputStream`:

{% codeblock lang:java %}

    @RunWith(JUnit4.class)
    public class TerminalViewTest extends TicTacToeTest {

        private final PrintStream stdout = System.out;
        private final ByteArrayOutputStream output = new ByteArrayOutputStream();
        private TerminalView terminalview;
    
{% endcodeblock %}        

Add a `setUp()` method with the JUnit `@Before` annotation—the usual
pattern to run some code before your tests. Pass `System.setOut()` a
UTF-8 `PrintStream` object constructed with the byte array (You _are_
using [UTF-8 everywhere](http://www.utf8everywhere.org/), right?):

{% codeblock lang:java %}

        @Before
        public void setUp() throws UnsupportedEncodingException {
            terminalview = new TerminalView();
            System.setOut(new PrintStream(output, true, "UTF-8"));
        }

{% endcodeblock %}

Then test against `output`. If the terminal output is as expected, it
should match the result of the printed object's `toString()` method:

{% codeblock lang:java %}

        @Test
        public void terminalViewShouldPrintBoards() {
            terminalview.print(nowins);
            assertEquals(nowins.toString(), output.toString());

        }

{% endcodeblock %}

Finally, make sure you clean up and redirect output back to stdout, so later print statements don't go missing:

{% codeblock lang:java %}

        @After
        public void cleanUp() {
            System.setOut(stdout);
        }
    }

{% endcodeblock %}

Now you're ready to write that one-liner. You can find the whole test
class as a gist [here](https://gist.github.com/ecmendenhall/5523091).

## Testing Unicode output with Infinitest
With my test in place, I wrote a quick print method and waited for the
green flash. And waited. And waited... I've been using
[Infinitest](http://infinitest.github.io/), an excellent continuous testing
plugin for IntelliJ and Eclipse, but it threw an assertion error, even
though the tests passed when run directly from IntelliJ.

The problem? I wasn't using [Unicode](http://www.joelonsoftware.com/articles/Unicode.html)
everywhere! (Don't say I didn't warn you.) My boards print with Unicode [box
drawing](http://unicode-table.com/en/#box-drawing) characters that
threw off Infinitest. IntelliJ runs in a UTF-8 environment by default,
but character encoding must be passed as an option to the Infinitest
JVM. Fortunately, this is another one-line
[solution](http://infinitest.github.io/doc/user_guide.html):
add a text file named `infinitest.args` in your project's root directory with one argument per
line. In this case, just:

    -D file.encoding=UTF-8

With tests back in the green, I was ready to start testing user
input—but that's a yak shave for another day.
