---
layout: post
title: "Enforcing Bottom-up Design"
date: 2013-05-10 00:21
comments: true
categories: [Lisp, TDD]
---
> "Experienced Lisp programmers divide up their programs differently.
As well as top-down design, they follow a principle which could be
called bottom-up design–changing the language to suit the problem.
In Lisp, you don't just write your program down toward the language,
you also build the language up toward your program. As you're writing
a program you may think 'I wish Lisp had such-and-such an operator.'
So you go and write it. Afterward you realize that using the new
operator would simplify the design of another part of the program, and
so on. Language and program evolve together. Like the border between
two warring states, the boundary between language and program is drawn
and redrawn, until eventually it comes to rest along the mountains and
rivers, the natural frontiers of your problem. In the end your program
will look as if the language had been designed for it. And when
language and program fit one another well, you end up with code which
is clear, small, and efficient."

–[Paul Graham](http://www.paulgraham.com/progbot.html), from the
introduction to [_On Lisp_](http://www.paulgraham.com/onlisptext.html)

Lisp programmers have a reputation (earned or otherwise) for considering their language of
  choice [uniquely powerful](http://xkcd.com/224/), capable of
  extending the [limits of the
  world](https://tractatus-online.appspot.com/Tractatus/Ajaxs/tlpA.html#56)
  with its special
  [expressiveness](http://stackoverflow.com/questions/267862/what-makes-lisp-macros-so-special).
  I find writing Lisp a joy, and for a long time I bought into the
  mythos. It still _feels_ unique. But I've come to learn that good bottom-up design is possible in any language. 

  For the past two weeks, I've been working in Java, a
  language many programmers consider 
  [Blub](http://www.paulgraham.com/avg.html) incarnate. (You don't have to take [my word for
  it](http://hammerprinciple.com/therighttool/items/java/clojure)).
  But even without clever macros and first-class functions, following
  the rules of [test-driven
  development](http://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530)
  and the principles of [clean
  code](http://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
  feels a lot like the process of natural, iterative evolution Paul
  Graham describes.
  
  A good language can encourage bottom-up, evolutionary design, and
  this is one of Lisp's great strengths. But
  writing good tests—and writing them first—can go a step further and
  actually enforce it.

 Writing tests first requires describing abstractions before
 they exist—writing the program you want to read from the very start. Using meaningful names transforms the language you have
 into the one you want. Revising after every passing test makes
 simplifying design second nature. And building up a program test by
 tiny test is an evolutionary process that generates clean, efficient
 code, whether you're writing Common Lisp or COBOL.

Here's a function that returns a given game board's winner from my
  first crack at Tic Tac Toe in Clojure:
  
{% codeblock tictactoe.core/get-win lang:clojure %}
(defn get-win
  "Takes a 3x3 game board. Returns a vector
  [winner start middle  end] of the winning player,
  and (row, col) grid coordinates of the three-in-a-row elements."
  [board]
    (let [wins           (check-for-wins board)
          winner         (if (= 1 (first (remove nil? wins))) 1 0)
          [row col diag] (unflatten wins)]
      (cond (not-empty-row? row)  [winner [(get-row-win row) 0]
                                          [(get-row-win row) 1]
                                          [(get-row-win row) 2]]
            (not-empty-row? col)  [winner [0 (get-row-win col)]
                                          [1 (get-row-win col)]
                                          [2 (get-row-win col)]]
            (not-empty-row? diag) (if (= 0 (get-row-win diag))
                                    [winner [0 0] [1 1] [2 2]]
                                    [winner [0 2] [1 1] [2 0]]))))
{% endcodeblock %}

And here's the equivalent I wrote in Java:
  
{% codeblock Board.winnerIs( ) lang:java %}
  public int winnerIs() {
      if (hasWin()) {
          return getWinningRow().winner();
      }
      return _;
  }
{% endcodeblock %}

Which excerpt reads more like a domain-specific language? Which
  would you rather read a year from now? I don't doubt that I could
  clean up the Clojure into something just as simple and readable. But
merely using an elegant language is no guarantee of elegant design.
