---
layout: post
title: "Refactoring examples: Refused-ish bequest"
date: 2013-06-03 01:04
comments: true
categories: [Java, refactoring] 
---

Before finishing up the first version of my terminal view game, I moved strings like the welcome message and player prompts to a [properties file](http://docs.oracle.com/javase/tutorial/essential/environment/properties.html). This requires reading and storing the strings before using them in the view, but has two big advantages. First, tests no longer break when I edit a string (at least, as long as my tests are using the same properties). Second, if I wanted to translate my program into another language, it's as easy as swapping out the properties file. 

Unfortunately, loading strings got ugly fast. Here's an example from the `GameController` class:

{% codeblock lang:java %}
public class GameController implements Controller {

    private String welcome;
    private String divider;
    private String yourMove;
    private String yourMoveThreeSquares;
    private String playAgain;
    private String gameOverDraw;
    private String gameOverWin;
    private String xWins;
    private String oWins;
    private String choosePlayerOne;
    private String choosePlayerTwo;
    private String boardSize;

    public GameController(View gameView) {
        loadViewStrings();
        view = gameView;
        board = new GameBoard();
    }

    private void loadViewStrings() {
        Properties viewStrings = new Properties();
        try {
            viewStrings.load(getClass().getResourceAsStream("/viewstrings.properties"));
        } catch (IOException e) {
            System.out.println(e);
        }

        welcome = viewStrings.getProperty("welcome");
        divider = viewStrings.getProperty("divider");
        yourMove = viewStrings.getProperty("yourmove");
        yourMoveThreeSquares = viewStrings.getProperty("yourmovethreesquares");
        playAgain = viewStrings.getProperty("playagain");
        gameOverDraw = viewStrings.getProperty("gameoverdraw");
        gameOverWin = viewStrings.getProperty("gameoverwin");
        xWins = viewStrings.getProperty("xwins");
        oWins = viewStrings.getProperty("owins");
        choosePlayerOne = viewStrings.getProperty("chooseplayerone");
        choosePlayerTwo = viewStrings.getProperty("chooseplayertwo");
        boardSize = viewStrings.getProperty("boardsize");

    }
 }
 {% endcodeblock %}
 
 It gets worse. When I started writing my Swing view, I needed to ignore certain strings, like those that prompt for keyboard input. A good idea: avoid creating a brand new SwingController class, but somehow filter out unneeded strings. A bad idea: do it by checking and ignoring certain strings from the controller. This is logic that really shouldn't be in the view, implemented in a very fragile way. In fact, it undoes all the abstraction of the properties file—as soon as a string changes, `displayMessage()` will probably break. Plus, there's plenty of duplicated code.
 
 {% codeblock lang:java %}
 public class SwingView extends JFrame implements View {

    private String divider;
    private String playAgain;
    private String choosePlayerOne;
    private String choosePlayerTwo;
    private String boardSize;

    private Set<String> ignoreThese = new HashSet<String>();

    public SwingView() {
        loadViewStrings();
    }

    private void loadViewStrings() {
        Properties viewStrings = new Properties();
        try {
            viewStrings.load(getClass().getResourceAsStream("/viewstrings.properties"));
        } catch (IOException e) {
            System.out.println(e);
        }

        divider = viewStrings.getProperty("divider");
        playAgain = viewStrings.getProperty("playagain");
        choosePlayerOne = viewStrings.getProperty("chooseplayerone");
        choosePlayerTwo = viewStrings.getProperty("chooseplayertwo");
        boardSize = viewStrings.getProperty("boardsize");

        ignoreThese.add(divider);
        ignoreThese.add(playAgain);
        ignoreThese.add(choosePlayerOne);
        ignoreThese.add(choosePlayerTwo);
        ignoreThese.add(boardSize);
    }

    public void displayMessage(String message) {
        if (message.contains("move")) {
            int endStart = message.indexOf("player") + 6;
            String ending = message.substring(endStart);
            message = "Your move, player" + ending;
        } else if (ignoreThese.contains(message)) {
            return;
        }
        JLabel messageLabel = messagePanel.getLabel();
        messageLabel.setText(message);
   }   
 }
 {% endcodeblock %}
 
 This is similar, though not identical to Fowler's "Refused bequest," where a subclass inherits lots of methods and then ignores them. Here' I'm loading lots of strings, then ignoring them.
 
 Refused bequest is fixed with a refactor called "Replace inheritance with delegation:" put the superclass in a field on the old subclass, remove the inheritance, and simply delegate to the superclass methods when needed. Here, I haven't even shared code through inheritance, but by good old copy-and-paste (so it's also part of the duplicated code stink parade).
 
 The cleanup strategy here is similar too: create a separate class to handle loading properties, and use it in place of the duplicated methods. I'll start by creating a `StringLoader` class that includes the duplicated fields and methods:
 
 {% codeblock lang:java %}
 public class StringLoader {
    String welcome;
    String divider;
    String yourMove;
    String yourMoveThreeSquares;
    String playAgain;
    String gameOverDraw;
    String gameOverWin;
    String xWins;
    String oWins;
    String choosePlayerOne;
    String choosePlayerTwo;
    String boardSize;

    public StringLoader() {
    }

    void loadViewStrings() {
        Properties viewStrings = new Properties();
        try {
            viewStrings.load(null.getClass().getResourceAsStream("/viewstrings.properties"));
        } catch (IOException e) {
            System.out.println(e);
        }

        welcome = viewStrings.getProperty("welcome");
        divider = viewStrings.getProperty("divider");
        yourMove = viewStrings.getProperty("yourmove");
        yourMoveThreeSquares = viewStrings.getProperty("yourmovethreesquares");
        playAgain = viewStrings.getProperty("playagain");
        gameOverDraw = viewStrings.getProperty("gameoverdraw");
        gameOverWin = viewStrings.getProperty("gameoverwin");
        xWins = viewStrings.getProperty("xwins");
        oWins = viewStrings.getProperty("owins");
        choosePlayerOne = viewStrings.getProperty("chooseplayerone");
        choosePlayerTwo = viewStrings.getProperty("chooseplayertwo");
        boardSize = viewStrings.getProperty("boardsize");

    }
}
{% endcodeblock %}

Instead of all these fields, storing strings in a map is much cleaner:

{% codeblock lang:java %}
public class StringLoader {
    private HashMap<String, String> viewStrings = new HashMap<String, String>();

    public StringLoader() {
    }

    void loadViewStrings() {
        Properties viewStrings = new Properties();
        try {
            viewStrings.load(getClass().getResourceAsStream("/viewstrings.properties"));
        } catch (IOException e) {
            System.out.println(e);
        }

        welcome = viewStrings.getProperty("welcome");
        divider = viewStrings.getProperty("divider");
        yourMove = viewStrings.getProperty("yourmove");
        yourMoveThreeSquares = viewStrings.getProperty("yourmovethreesquares");
        playAgain = viewStrings.getProperty("playagain");
        gameOverDraw = viewStrings.getProperty("gameoverdraw");
        gameOverWin = viewStrings.getProperty("gameoverwin");
        xWins = viewStrings.getProperty("xwins");
        oWins = viewStrings.getProperty("owins");
        choosePlayerOne = viewStrings.getProperty("chooseplayerone");
        choosePlayerTwo = viewStrings.getProperty("chooseplayertwo");
        boardSize = viewStrings.getProperty("boardsize");

    }
}
{% endcodeblock %}

The repeated calls to `viewStrings.getProperty()` are still pretty ugly and very difficult to read. One solution is to extract a `load()` method, then iterate over an array of the property names:

{% codeblock lang:java %}
public class StringLoader {
    private HashMap<String, String> viewStrings = new HashMap<String, String>();
    private Properties viewStringProperties = new Properties();

    public StringLoader() {
        loadViewStrings();
    }

    private void load(String propertyName) {
        String propertyString = viewStringProperties.getProperty(propertyName);
        viewStrings.put(propertyName, propertyString)
    }

    private void loadViewStrings() {
        try {
            viewStringProperties.load(getClass().getResourceAsStream("/viewstrings.properties"));
        } catch (IOException exception) {
            exception.printStackTrace();
        }

        String[] properties = { "welcome",
                                "divider",
                                "yourmove",
                                "yourmovethreesquares",
                                "playagain",
                                "gameoverdraw",
                                "gameoverwin",
                                "xwins",
                                "owins",
                                "chooseplayerone",
                                "chooseplayertwo",
                                "boardsize"};

        for (String property : properties) {
            load(property);
        }

    }
}
{% endcodeblock %}

Finally, let's replace the hardcoded filepath by passing a path to the view constructor (and on to the `loadViewStrings()` method:

{% codeblock lang:java %}
public StringLoader(String filepath) {
      loadViewStrings(filepath);
}
{% endcodeblock %}

Now, reading in strings from the properties file looks like this, instead of the forty-line monster we started with:

{% codeblock lang:java %}
public class GameController implements Controller {
    private HashMap<String, String> viewStrings =  
      new StringLoader().getViewStrings("/viewstrings.properties");
}
{% endcodeblock %}

To use a string, I can just get it from the map:

{% codeblock lang:java %}
viewStrings.get("welcome");
{% endcodeblock %}

This requires trading off a little clarity, but on balance it's much cleaner. Best of all, now I can use properties as they were intended. Preventing my view from displaying certain strings just requires creating a new properties file and removing the content from the unneeded messages—another win for decoupling and abstraction.
