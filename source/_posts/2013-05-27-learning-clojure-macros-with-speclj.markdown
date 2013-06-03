---
layout: post
title: "Reconstructing Clojure macros with speclj"
date: 2013-05-27 17:01
comments: true
categories: [Clojure, speclj, macros] 
---
> "It is a revelation to compare Menard’s Don Quixote with Cervantes’.
The latter, for example, wrote (part one, chapter nine):
'…truth, whose mother is history, rival of time, depository of
deeds, witness of the past, exemplar and adviser to the present, and
the future’s counselor.' Written in the seventeenth century, written by
the 'lay genius' Cervantes, this enumeration is a mere rhetorical
praise of history. Menard, on the other hand, writes:
'…truth, whose mother is history, rival of time, depository of
deeds, witness of the past, exemplar and adviser to the present, and
the future’s counselor.'"

–From [_Pierre Menard, Author of the Quixote_](http://www.coldbacon.com/writing/borges-quixote.html)
 
[Macros are hard](http://www.infoq.com/presentations/Clojure-Macros),
but one of the most helpful exercises I've found in my limited macro
writing experience is practicing by recreating some of the [core
macros](http://clojure.org/macros) I already know and love, like `or`,
`when-let`, and `->`.

Using [speclj](https://github.com/slagyr/speclj) greatly simplifies
this exercise, and writing macro specs often test my understanding
better than writing the implementations themselves. Here's an example spec for the threading macro:

{% codeblock lang:clojure %}
(describe "-> macro"
  (it "should expand into the code below"
    (let [macro-form '(->-macro 1 (+ 2) (* 3))]

      (should= '(macro-challenges.core/->-macro
                  (macro-challenges.core/->-macro 1 (+ 2)) (* 3))
                (macroexpand-1 macro-form))

      (should= '(* (macro-challenges.core/->-macro 1 (+ 2)) 3)
                (macroexpand macro-form))

      (should= '(* (+ 1 2) 3)
                (macroexpand-all macro-form)))))
{% endcodeblock %}

The functions `macroexpand`, `macroexpand-1`, and `macroexpand-all`
come in very handy. `Macroexpand-1` returns the "first" expansion of a
macro form (macros within macros won't be expanded). `Macroexpand`
calls `macroexpand-1` until the expansion is no longer a macro form.

In the above example, the first expansion of `->-macro`, my threading
macro replacement, returns a form that starts with another call to
`->-macro`. (I was surprised to find out that this is how the
threading macro works under the hood). `Macroexpand` expands [into a
list](http://stackoverflow.com/questions/2296385/homoiconicity-how-does-it-work)
until the first item is `*`, which is not a macro.

When it's macros all the way down, `macroexpand-all` (technically
`clojure.walk/macroexpand-all`) recursively expands all macros in a
given form, resulting in the much simpler expression `(* (+ 1 2) 3)`
in the example above. These functions are all hugely helpful for
writing macros and their associated tests.

Here's my recreation of `->`, which passed the spec:

{% codeblock lang:clojure %}
(defmacro ->-macro
  ([arg] arg)
  ([arg first-form] `(~(first first-form) ~arg ~@(rest first-form)))
  ([arg first-form & more-forms]
     `(->-macro (->-macro ~arg ~first-form) ~@more-forms)))
{% endcodeblock %}

And bears a pretty strong resemblance to the original source:

{% codeblock lang:clojure %}
(defmacro ->
  "Threads the expr through the forms. Inserts x as the
  second item in the first form, making a list of it if it is not a
  list already. If there are more forms, inserts the first form as the
  second item in second form, etc."
  {:added "1.0"}
  ([x] x)
  ([x form] (if (seq? form)
              (with-meta `(~(first form) ~x ~@(next form)) (meta form))
              (list form x)))
  ([x form & more] `(-> (-> ~x ~form) ~@more)))
{% endcodeblock %}

Some macros are pretty easy to reconstruct (but check [their
documentation](http://clojuredocs.org/clojure_core/clojure.repl/doc)
to make sure you really understand how they handle different
arguments). If you're well and truly stuck, it's always possible to check out the
[original
code](http://clojuredocs.org/clojure_core/clojure.repl/source) for inspiration.

The specs and solutions I've written so far (mostly low-hanging fruit) are all
  available on my
  [Github](https://github.com/ecmendenhall/macro-challenges), if you'd
  like to have a hand at reconstructing Clojure yourself.  
