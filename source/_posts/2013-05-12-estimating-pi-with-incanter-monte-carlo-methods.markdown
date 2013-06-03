---
layout: post
title: "Estimating Pi with Incanter and Monte Carlo Methods"
date: 2013-05-12 14:26
comments: true
categories: [Clojure, Incanter] 
---

Of all the things [John von
Neumann](https://en.wikipedia.org/wiki/John_von_neumann#Career_and_abilities)
invented (including the [Minimax
theorem](https://en.wikipedia.org/wiki/Minimax_theorem#Minimax_theorem)
behind my Tic Tac Toe AI, and the [architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture) of the computer it runs
on), Monte Carlo simulation is one of my favorites. Unlike some of his
other
[breakthroughs](https://en.wikipedia.org/wiki/Ergodic_theory#Mean_ergodic_theorem),
the idea behind Monte Carlo simulation is simple: use probability
and computation to estimate solutions to hard problems.

Think of the craziest integral you've ever solved, and the sheaves of
notebook paper spent working out the area under that ridiculous curve. 
The Monte Carlo approach is a clever hack: tack a graph of the
function to a dartboard, and start throwing darts at
random. As more and more darts hit the board, the ratio of darts that
land under the curve to total darts thrown will approximate the
proportion of the dartboard under the curve. Multiply this ratio by
the area of the dartboard, and you've computed the integral. As a student whose
favorite fourth grade problem solving strategy was "guess and check,"
and whose favorite tenth grade problem solving strategy was "plug it
into your TI-83," the idea has a lot of appeal.

Wikipedia has a great example of [estimating the value of
Pi](https://en.wikipedia.org/wiki/Monte_carlo_simulation#Introduction)
with a Monte Carlo simulation. I wrote a similar simulation [in Python](https://gist.github.com/ecmendenhall/5054266) a couple
months ago, but wanted to check out Clojure's [Incanter](http://incanter.org/) library for
statistical computing and try my hand at a TDD solution.

To start, you'll want the [speclj](https://github.com/slagyr/speclj)
test framework for Clojure, which uses Rspec-like syntax and provides
a helpful autorunner. Create a new Leiningen project and add it as a dev
dependency and plugin in `project.clj`. Make sure you set up your spec
directory as [described](https://github.com/slagyr/speclj) in the
documentation. You'll also want to add Incanter to your project
dependencies. Here's my final `project.clj`:

{% codeblock project.clj lang:clojure %}
(defproject montecarlopi "0.1.0"
  :dependencies [[org.clojure/clojure "1.5.0"]
                 [incanter "1.5.0-SNAPSHOT"]]
  :main montecarlopi.core
  :profiles {:dev {:dependencies [[speclj "2.5.0"]]}}
  :plugins [[speclj "2.5.0"]]
  :test-paths ["spec/"])
{% endcodeblock %}

To start, we'll need a way to tell whether a given point is inside the
circle. Here's a simple test:

{% codeblock core_spec.clj lang:clojure %}
(ns montecarlopi.core-spec
  (:require [speclj.core :refer :all]
            [montecarlopi.core :refer :all]))

(describe "in-circle?"
  (it "should return true for points inside the circle."
      (should (in-circle? [0 0.1])))

  (it "should return false for points outside the circle."
      (should-not (in-circle? [0.8 0.9]))))

{% endcodeblock %}

Fire up the testrunner with `lein spec -a`, and you'll see an
exception, since the function doesn't exist. Time to add it to
`core.clj`:

{% codeblock core.clj lang:clojure %}
(ns montecarlopi.core
  (require [incanter.core   :refer [sq]]

(defn in-circle? [point]
  (let [[x y] point]
    (if (<= (+ (sq x)
               (sq y))
            1.0)
      true
      false)))
{% endcodeblock %}

The `let` binding pulls x and y coordinates out of a vector
representing a point. If x squared plus y squared are less
than 1, the point is inside the circle. Save and watch the
autorunner turn green.

Next, we need a way to generate points at random, with x and y values
between 0 and 1. Here's a test:

{% codeblock core_spec.clj lang:clojure %}
(describe "generate-random-point"
  (it "should return a point with x and y values between 0 and 1."
      (let [[x y] (generate-random-point)]
        (should (<= x 1))
        (should (>= x 0))

        (should (<= y 1))
        (should (>= y 0)))))
{% endcodeblock %}

And a dead-simple implementation (`clojure.core/rand` conveniently
generates random floats between 0 and 1).:

{% codeblock core.clj lang:clojure %}
(defn generate-random-point []
  [(rand) (rand)])
{% endcodeblock %}

We need a lot of random points, so we'd better add a function to
generate them. Here's a test and a function that passes: 

{% codeblock core_spec.clj lang:clojure %}
(describe "generate-random-points"
  (it "should return the specified number of random points."
    (should= 100 (count (generate-random-points 100)))))
{% endcodeblock %}

{% codeblock core.clj lang:clojure %}
(defn generate-random-points [n]
  (take n (repeatedly generate-random-point)))
{% endcodeblock %}

Once we've generated lots of points, we'll want to know how many lie
inside the circle. This test is a little more complicated, since it
requires some mock points to test against:

{% codeblock core_spec.clj lang:clojure %}

(describe "points-in-circle"
  (it "should return only points inside the circle."
    (let [inside  [[0.0 0.0]
                   [0.2 0.3]
                   [0.1 0.8]
                   [0.0 1.0]
                   [1.0 0.0]]
            
          outside [[1.0 0.2]
                   [0.8 0.8]
                   [0.5 0.9]
                   [0.8 0.7]
                   [0.9 0.9]]
          points  (concat inside outside)]
      (should= 5 (count (points-in-circle points)))
      (should= inside (points-in-circle points))
      (should-not= outside (points-in-circle points)))))
{% endcodeblock %}

But a passing solution is as easy as using `in-circle?` as a filter:
            
{% codeblock core.clj lang:clojure %}
(defn points-in-circle [points]
  (filter in-circle? points))
{% endcodeblock %}

In order to plot the points, we'll need to sort them into groups. A
map should do the trick. Let's also take this chance to
pull `inside`, `outside`, and `points` out of the `let`
binding and define them as vars so we can reuse them.
Here's the result after a quick refactor:

{% codeblock core_spec.clj lang:clojure %}
(def inside  [[0.0 0.0]
              [0.2 0.3]
              [0.1 0.8]
              [0.0 1.0]
              [1.0 0.0]]) 

(def outside [[1.0 0.2]
              [0.8 0.8]
              [0.5 0.9]
              [0.8 0.7]
              [0.9 0.9]])

(def points (concat inside outside))


(describe "points-in-circle"
  (it "should return only points inside the circle."
    (should= 5 (count (points-in-circle points)))
    (should= inside (points-in-circle points))
    (should-not= outside (points-in-circle points))))
            
(describe "sort-points"
  (it "should return a map of correctly sorted points."
      (should= {:inside  inside  
                :outside outside}
               (sort-points points))))
{% endcodeblock %}

This code passes, but it's not as clear as it could be:

{% codeblock core.clj lang:clojure %}
(defn sort-points [points]
  {:inside  (filter in-circle? points)
   :outside (filter #(not (in-circle %)) points)})
{% endcodeblock %}

I'd like to rename the inline function `outside-circle?`. So let's add
a test:

{% codeblock core_spec.clj lang:clojure %}
(describe "outside-circle?"
  (it "should return true for points outside the circle."
      (should (outside-circle? [0.9 0.7]))))
{% endcodeblock %}

Write the function:

{% codeblock core.clj lang:clojure %}
(defn outside-circle? [point]
  (not (in-circle? point)))
{% endcodeblock %}

And refactor `sort-points` once speclj gives us the green light:

{% codeblock core.clj lang:clojure %}
(defn sort-points [points]
  {:inside  (filter in-circle? points)
   :outside (filter outside-circle? points)})
{% endcodeblock %}

It shouldn't be hard to convert sorted points into a ratio. Here's a
test and solution:

{% codeblock core_spec.clj lang:clojure %}
(describe "point-ratio"
  (it "should return the correct ratio of inside to outside points"
      (should= 0.5 (point-ratio points))))
{% endcodeblock %}

Make sure you return a float instead of a rational:

{% codeblock core.clj lang:clojure %}
(defn point-ratio [points]
  (float (/ (count (points-in-circle points))
            (count points))))
{% endcodeblock %}

And now we're just a step away from estimating Pi. We're considering a
quarter circle inside a unit square. The area of the
square is 1, and the quarter circle 1/4 * Pi. So
multiplying by four gives us our estimate:            
            
{% codeblock core_spec.clj lang:clojure %}
(describe "estimate-pi"
 (it "should return four times the ratio of inside to outside points."
      (should= 2.0 (estimate-pi points)))

  (it "should return something close to pi when given a lot of points."
      (let  [estimate (estimate-pi (generate-random-points 70000))]
        (should (> estimate 3.13))
        (should (< estimate 3.15)))))
{% endcodeblock %}

The second part of this test is a little tricky. Our estimate is
probabilistic, but with enough points it should almost always generate
an estimate within range. Passing the test is easy:

{% codeblock core.clj lang:clojure %}
(defn estimate-pi [points]
  (* 4 (point-ratio points)))
{% endcodeblock %}

All that's left is to create an Incanter
chart. We'll start by plotting the circle with Incanter. First, we
need to write a function. Here's a test for points on 1 = x^2 + y^2:

{% codeblock core_spec.clj lang:clojure %}
(describe "circle"
  (it "should equal 1 when x is 0."
    (should= 1.0 (circle 0)))
  (it "should equal 0 when x is 1."
      (should= 0.0 (circle 1)))
  (it "should equal 0.866 when x is 0.5"
      (should= "0.866" (format "%.3f" (circle 0.5)))))
{% endcodeblock %}

To write the function, make sure you refer `incanter.core/sqrt` to
your namespace:

{% codeblock core.clj lang:clojure %}
(ns montecarlopi.core
  (require [incanter.core   :refer [sqrt sq]]))
                   
(defn circle [x] (sqrt (- 1.0 (sq x))))
{% endcodeblock %}

At this point, writing tests gets a little hairy. The `JFreeChart`
object on which Incanter plots are based doesn't
[offer much](http://www.jfree.org/jfreechart/api/javadoc/index.html)
in the way of public fields to test against. But we
can at least check that the function returns a chart:  

{% codeblock core_spec.clj lang:clojure %}
(describe "draw-circle"
  (it "should be a JFreeChart object."
      (should= "class org.jfree.chart.JFreeChart"
               (str (.getClass (draw-circle))))))
{% endcodeblock %}

Refer `incanter.charts/function-plot` to plot the function: 

{% codeblock core.clj lang:clojure %}
(ns montecarlopi.core
  (require [incanter.core   :refer [sqrt sq view]]
           [incanter.charts :refer [function-plot]]))

(defn draw-circle []
  (function-plot circle 0 1)) 
{% endcodeblock %}

Running `(view (draw-circle))` from the REPL should produce a chart
like this:

{% img /images/circle-plot.png %}

The last step is to add the points and some annotations to the
Incanter plot:                   

{% codeblock core.clj lang:clojure %}                   
(ns montecarlopi.core
  (require [incanter.core   :refer [sqrt sq view]]
           [incanter.charts :refer [function-plot
                                    add-points
                                    add-text
                                    set-x-label
                                    set-y-label
                                    set-y-range
                                    xy-plot]]))


(defn plot-points [chart points label]
  (let [xs (map first  points)
        ys (map second points)]
    (add-points chart xs ys :series-label label)))

(defn make-plot [n]
  (let [points (generate-random-points n)
        sorted (sort-points points)]
    (doto (draw-circle)
      (set-y-range -0.25 1)
      (plot-points (:inside  sorted) "inside")
      (plot-points (:outside sorted) "outside")
      (set-x-label "")
      (set-y-label "")
      (add-text 0.10 -0.10 (str "Total: "
                                (count points)))
      (add-text 0.10 -0.05 (str "Inside: "
                                (count (:inside sorted))))
      (add-text 0.10 -0.15 (str "Ratio: "
                                (point-ratio points)))
      (add-text 0.10 -0.20 (str "Pi: "
                                (format "%4f" (estimate-pi points))))
      (view :width 500 :height 600))))
{% endcodeblock %}

Here's a plot and estimate with 100 points:

{% img /images/pi-100.png %}

With 10000:

{% img /images/pi-10000.png %}

And with 100000:

{% img /images/pi-100000.png %}

A gist with all the code from this post is [available here](https://gist.github.com/ecmendenhall/5565604).

