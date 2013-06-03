---
layout: post
title: "Network closures in Clojure"
date: 2013-05-25 12:38
comments: true
categories: [Clojure, Lisp] 
---
Clojure may be a new language, but Lisp has a long history. 
Translating Scheme and Common Lisp classics into Clojure is always an
interesting exercise, and often illuminates the differences and
comparative advantages of various Lisp-y languages. (For more
Lisp-to-Clojure resources see [this
list](http://juliangamble.com/blog/2012/07/13/amazing-lisp-books-living-again-in-clojure/).
Or, if you'd like to try porting some Scheme, consider [helping
translate SICP](https://github.com/ecmendenhall/sicpclojure)).

This weekend, I spent some time with Paul Graham's classic [_On
Lisp_](http://www.paulgraham.com/onlisp.html).
In Chapter 6, Graham shows how to use
[closures](https://en.wikipedia.org/wiki/Closure_(computer_science\))
to model nodes in a network, representing a 20 questions game as a self-traversing binary
tree. Here's his original code, and my best attempts at Clojure translations.

The most obvious model for a set of connected nodes is a nested data
structure, like a map of maps. In Common Lisp, Graham uses a mutable
hashmap of [structured
types](http://www.lispworks.com/documentation/HyperSpec/Body/m_defstr.htm),
each pointing to their neighbors in the network:

{% codeblock lang:common-lisp %}
(defstruct node contents yes no)

(defvar *nodes* (make-hash-table))

(defun defnode (name conts &optional yes no)

(setf (gethash name *nodes*)
  (make-node :contents conts
    :yes yes
    :no no)))
{% endcodeblock %}

A simple map seems like a sufficient replacement in Clojure. A single
node looks like this:

{% codeblock lang:clojure %}
{:people {:contents "Is the person a man?", :yes :male, :no :female}}
{% endcodeblock %}

And the full tree looks like the following. Each node's `:yes` or `:no` keyword
points to the next node in the tree:

{% codeblock lang:clojure %}
{:penny {:contents "Abraham Lincoln.", :yes nil, :no nil},
 :coin  {:contents "Is the coin a penny?", :yes :penny, :no :other-coin},
 :USA   {:contents "Is he on a coin?", :yes :coin, :no :no-coin},
 :dead  {:contents "Was he from the USA?", :yes :USA, :no :elsewhere},
 :male  {:contents "Is he living?", :yes :live, :no :dead},
 :people {:contents "Is the person a man?", :yes :male, :no :female}}
{% endcodeblock %}

Here's my first draft for defining nodes in Clojure. Since the Common
Lisp version used a mutable variable, I used a Clojure atom to store
network state:

{% codeblock lang:clojure %}
(def nodes (atom {}))

(defn defnode [name contents & [yes no]]
  (swap! nodes assoc name {:contents contents :yes yes :no no}))

(defn make-nodes []
  (defnode :people "Is the person a man?" :male :female)
  (defnode :male "Is he living?" :live :dead)
  (defnode :dead "Was he from the USA?" :USA :elsewhere)
  (defnode :USA "Is he on a coin?" :coin :no-coin)
  (defnode :coin "Is the coin a penny?" :penny :other-coin)
  (defnode :penny "Abraham Lincoln."))
{% endcodeblock %}

Traversing the network is simple: get a node, print the associated
question, prompt for input, get the next node, and repeat. Here's the original Common Lisp:

{% codeblock lang:common-lisp %}
(defun run-node (name)
  (let ((n (gethash name *nodes*)))
    (cond ((node-yes n)
           (format t "~A~%>> " (node-contents n))
           (case (read)
             (yes (run-node (node-yes n)))
             (t (run-node (node-no n)))))
          (t (node-contents n)))))
{% endcodeblock %}

And here's an equivalent in Clojure:

{% codeblock lang:clojure %}
(defn run-node [name]
  (let [node     (@nodes name)
        contents (node :contents)
        yes      (node :yes)
        no       (node :no)]
    (if yes
      (do
        (println contents)
         (if (= "yes" (read-line))
           (run-node yes)
           (run-node no)))
      (println contents))))
{% endcodeblock %}

Of course, there's no reason to bother swapping and dereferencing an
atom as long as the tree won't need to change at runtime. This
immutable version works just as well:

{% codeblock lang:clojure %}
(defn defnode [nodes name contents & [yes no]]
  (assoc nodes name {:contents contents :yes yes :no no}))

(defn make-nodes []
  (-> {}
    (defnode :people "Is the person a man?" :male :female)
    (defnode :male "Is he living?" :live :dead)
    (defnode :dead "Was he from the USA?" :USA :elsewhere)
    (defnode :USA "Is he on a coin?" :coin :no-coin)
    (defnode :coin "Is the coin a penny?" :penny :other-coin)
    (defnode :penny "Abraham Lincoln.")))

(def nodes (make-nodes))

(defn run-node [nodes name]
  (let [node     (nodes name)
        contents (node :contents)
        yes      (node :yes)
        no       (node :no)]
    (if yes
      (do
        (println contents)
         (if (= "yes" (read-line))
           (run-node nodes yes)
           (run-node nodes no)))
      (println contents))))
{% endcodeblock %}

Using a closure rolls the data structure and traversal code into one,
by associating the `yes` and `no` fields with anonymous functions that
handle the same logic as `run-node`. Here's Graham's CL version:

{% codeblock lang:common-lisp %}
(defun defnode (name conts &optional yes no)
  (setf (gethash name *nodes*)
        (if yes
          #’(lambda ()
              (format t "~A~%>> " conts)
              (case (read)
                (yes (funcall (gethash yes *nodes*)))
                (t (funcall (gethash no *nodes*)))))
          #’(lambda () conts))))
{% endcodeblock %}

And here's mine in Clojure. The double-parens around `(@nodes yes)`
and `(@nodes no)` call the anonymous function, instead of just
returning it. 

{% codeblock lang:clojure %}
(def nodes (atom {}))

(defn defclosure [name contents & [yes no]]
  (swap! nodes assoc name
         (if yes
           (fn []
             (println contents)
             (if (= "yes" (read-line))
               ((@nodes yes))
               ((@nodes no))))
           (fn []
             (println contents)))))

(defn make-nodes []
  (defclosure :people "Is the person a man?" :male :female)
  (defclosure :male "Is he living?" :live :dead)
  (defclosure :dead "Was he from the USA?" :USA :elsewhere)
  (defclosure :USA "Is he on a coin?" :coin :no-coin)
  (defclosure :coin "Is the coin a penny?" :penny :other-coin)
  (defclosure :penny "Abraham Lincoln."))
{% endcodeblock %}

Now, traversing the tree is as simple as calling:
{% codeblock lang:clojure %}
((nodes :people))
{% endcodeblock %}
And watching the tree traverse itself. You can find my code from this
post [here](https://gist.github.com/ecmendenhall/5646594). For more on closures in Lisp,
check out the rest of [Chapter 6](http://lib.store.yahoo.net/lib/paulgraham/onlisp.pdf).

