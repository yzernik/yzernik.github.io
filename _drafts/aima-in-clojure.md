---
layout: post
title:  "AIMA in Clojure"
tags: clojure
---

The AIMA textbook comes with implementation of the book's algorithms in both Python and Common Lisp. The algorithms were originally implemented in Common Lisp, but later translated to Python for educational purposes.

Peter Norvig [says][norvig-python-lisp] that Python is easier to use for students who don't have Lisp experience. It has an interactive interpreter. It has better interoperability with Java (using Jython).

He also mentions disadvantages:

> The two main drawbacks of Python from my point of view are (1) there is very little compile-time error analysis and type declaration, even less than Lisp, and (2) execution time is much slower than Lisp, often by a factor of 10 (sometimes by 100 and sometimes by 1). Qualitatively, Python feels about the same speed as interpreted Lisp, but very noticably slower than compiled Lisp. For this reason I wouldn't recommend Python for applications that are (or are likely to become over time) compute intensive (unless you are willing to move the speed bottlenecks into C). But my purpose is oriented towards pedagogy, not production, so this is less of an issue.

Clojure, which didn't exist at the time AIMA was published, has most of the above advantages.

* It has an interactive REPL
* It runs on the JVM
* It is not as complicated as Common Lisp
* It does some compile-time error analysis
* It has good performance.

So Clojure is a good language for me to practice some of the exercises from the AIMA book.

## Search

Chapter 3 is about `problem-solving agents` that use search algorithms to find solutions.

It gives a formal definition of a `problem` with five components: initial state, actions, transition model, goal test, and path cost.

This can be translated into Clojure as a protocol:

{% highlight clojure %}
(defprotocol Problem
  "An abstract formulation of a search problem"
  (initial-state [this] "Initial state in which the agent starts")
  (actions [this state] "Possible actions available to the agent at a state")
  (result [this state action] "The result of taking an action at a state")
  (goal? [this state] "Determines whether a given state is a goal state")
  (step-cost [this state action] "The cost of taking an action in a state"))
{% endhighlight %}

There book also defines a node in a search tree having four components: 
state, parent, action, and path cost.

In Clojure, this can be a record:

{% highlight clojure %}
(defrecord Node [state action parent cost])
{% endhighlight %}

Another important data structure is the fringe or frontier. It has three
 operations: empty?, pop, and insert.

Again, we can use a protocol:

{% highlight clojure %}
(defprotocol Fringe
  "A queue for storing unexplored nodes"
  (insert [this node] "Insert a new node into the fringe")
  (remove-next [this] "Remove the next node from the fringe"))
{% endhighlight %}

Since, we will be using persistent, immutable data structures for the fringe,
the insert function returns a new fringe, and the remove-next function
 returns a vector with the next item, and the remaining items.

Now we can define a general tree search function,  if we first define some utility functions:

{% highlight clojure %}
(defn- make-initial-node
  "Make the initial node for a problem"
  [problem]
  (->Node (initial-state problem) nil nil 0))

(defn- make-successor-node
  "Make a successor node from the current node and the next action"
  [problem node action]
  (let [{state :state parent :parent cost :cost} node
        r (result problem state action)
        sc (step-cost problem state action)]
    (->Node r action node (+ sc cost))))

(defn- successors
  "The successor nodes of a given node for a problem"
  [problem node]
  (let [a (actions problem (:state node))]
    (map (partial make-successor-node problem node) a)))

(defn- insert-nodes
  "Insert multiple nodes into a fringe"
  [fringe nodes]
  (reduce insert fringe nodes))
{% endhighlight %}

{% highlight clojure %}
(defn tree-search
  "General tree search algorithm"
  [problem fringe]
  (loop [f (insert fringe (make-initial-node problem))]
    (if (seq f)
      (let [[node f] (remove-next f)]
        (if (goal? problem (:state node)) node
            (recur (insert-nodes f (successors problem node))))))))
{% endhighlight %}

Graph search is almost the same as tree search, except that it also uses a Clojure
 set (`#{}`) to maintain the closed set of expanded nodes:
 
{% highlight clojure %}
(defn graph-search
  "General graph search algorithm"
  [problem fringe]
  (loop [f (insert fringe (make-initial-node problem))
         c #{}]
    (if (seq f)
      (let [[node f] (remove-next f)
            {state :state} node]
        (cond (goal? problem state) node
              (c state) (recur f c)
              :else (recur (insert-nodes f (successors problem node))
                           (conj c state)))))))
{% endhighlight %}
 
### Fringe implementations

These general search functions are nice, but they are useless without an 
implementation of fringe.

We can start by implementing a LIFO fringe (a stack). In Clojure, the way to
do this is to use a persistent list. So using our previously defined Fringe 
protocol, we have:

{% highlight clojure %}
(extend-type clojure.lang.IPersistentList
  Fringe
  (insert [this node] (conj this node))
  (remove-next [this] [(first this) (rest this)]))

(defn- make-stack []
  ())
{% endhighlight %}

We can do a similar thing for a FIFO fringe, using Clojure's persistent
 queue:

{% highlight clojure %}
(extend-type clojure.lang.PersistentQueue
  Fringe
  (insert [this node] (conj this node))
  (remove-next [this] [(peek this) (pop this)]))

(defn- make-queue []
  clojure.lang.PersistentQueue/EMPTY)
{% endhighlight %}

Now we can use these fringe types to create some different search functions:

{% highlight clojure %}
(defn depth-first-tree-search [problem]
  (tree-search problem (make-stack)))

(defn depth-first-graph-search [problem]
  (graph-search problem (make-stack)))
{% endhighlight %}

and 

{% highlight clojure %}
(defn breadth-first-tree-search [problem]
  (tree-search problem (make-queue)))

(defn breadth-first-graph-search [problem]
  (graph-search problem (make-queue)))
{% endhighlight %}

### Solving problems

We can see how well our search algorithms work on some simple problems:

For example, we can define the N Queens problem like this:

{% highlight clojure %}
(defrecord NQueensProblem [n]
  Problem
  (initial-state [this] [])
  (actions [this state] (filter (partial valid-column? state) (range n)))
  (result [this state action] (conj state action))
  (goal? [this state] (= n (count state)))
  (step-cost [this state action] 1))
{% endhighlight %}

The state is represented as a vector of integers, where the integer at each
index is the column position of the queen in that row.

The valid-column function used to get the actions looks like this:

{% highlight clojure %}
(defn- conflict?
  "Are the two points conflicting"
  [[row1 col1] [row2 col2]]
  (or (= row1 row2)
      (= col1 col2)
      (= (- row1 col1) (- row2 col2))
      (= (+ row1 col1) (+ row2 col2))))

(defn- valid-column?
  "Is the column location of the new queen valid"
  [state col]
  (let [positions (map-indexed vector state)
        new-pos [(count state) col]]
    (not-any? (partial conflict? new-pos) positions)))
{% endhighlight %}



[norvig-python-lisp]:             http://norvig.com/python-lisp.html

