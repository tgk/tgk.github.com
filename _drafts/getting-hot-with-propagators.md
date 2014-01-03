---
layout: post
title: Getting hot with propagators
---

# {{page.title}}

## Introduction

Propagators are a form of declarative programming, and are in many ways
similar to logic programming, but with very different semantics. This
means that some things, such as numerical calculations, are much easier
to do using propagators as opposed to logic programming.

In this blog post I'll demonstrate how to set up simple systems for
maintaining a temperature in both fahrenheit and centigrade,
automatically derive facts from values in a system, and to re-define
what is meant by conflicting values in a propagator system. This is a
technical tutorial, not a brief overview of propagators. I suggest
checking out the presentation if you're more interested in why
propagator systems are a good idea, and how they relate to other
declarative programming techniques, such as logic programming. The
building height code from the presentation can be found
[in the example directory of the project](https://github.com/tgk/propaganda/tree/master/examples).

If you need a two-minute quick overview of propagators,
[here's one page of text and animations to get you started](http://tgk.github.io/propaganda/).

All the code here uses version 0.2.0 of
[the propaganda library](https://github.com/tgk/propaganda).

## Simple two-way relations

Let's imagine we are building a system to represent the current
temperature outside. Our system is fed by different sources of data,
some of them using centigrade and some of them using fahrenheit to
represent temperature. We wish to built a system where we avoid spending
too much time focusing on our current reference system, and where we can
detect when sources disagree.

Our first order of business is to define functions for converting
between the two systems.

{% highlight clj %}
(defn f->c
  [f]
  (* 5/9 (- f 32)))

(f->c 100N)
;; => 340/9 (37.8)

(defn c->f
  [c]
  (+ (* 9/5 c) 32))

(c->f (f->c 100N))
;; => 100N
{% endhighlight %}

So far, everything is just plain old Clojure. Our next order of business
is to promote these functions to propagators that monitor cell values in
our system and maintain consistency.

{% highlight clj %}
(defn c-f-relation
  [system f c]
  (-> system
      ((function->propagator-constructor f->c) f c)
      ((function->propagator-constructor c->f) c f)))
{% endhighlight %}

We've generated a function that takes a system and enforces the
bijective relation between `c` and `f`, the temperature in centigrade
and fahrenheit. `function->propagator-constructor` takes a function and
returns a function that installs a propagator which observers all the
first cells it's given as a paramenter, and ensure that the last cell
it's given has the result of applying the function to those. If any of
the cells do not yet hold anything (the `nothing` object in propaganda),
the propagator does nothing.

That's all we need -- we're now ready to automatically convert between
the two systems and detect conflicting temperature readings.

{% highlight clj %}
(-> (make-system)
    (c-f-relation :temp-f :temp-c)
    (add-value :temp-f 100N)
    (get-value :temp-c))
;; => 340/9

(-> (make-system)
    (c-f-relation :temp-f :temp-c)
    (add-value :temp-c 340/9)
    (get-value :temp-f))
;; => 100N

(-> (make-system)
    (c-f-relation :temp-f :temp-c)
    (add-value :temp-f 100N)
    (add-value :temp-c 340/9)
    (get-value :temp-c))
;; => 340/9

(-> (make-system)
    (c-f-relation :temp-f :temp-c)
    (add-value :temp-f 100N)
    (add-value :temp-c 38N)
    (get-value :temp-c))
;; throws ExceptionInfo Inconsistency
{% endhighlight %}


## Even simpler one-way relation

We can also use propagators where information only flows one way. This
is less powerful, but sometimes it will be the nature of our problem,
that we can derive A from B, but not B from A. For example, if it's less
than 0 degress celcius outside, we know it's cold, but if we just know
it's cold outside, we don't know the exact temperature.

Let's define this one-way relation and put it into our system.

{% highlight clj %}
(defn c->cold-or-hot
  [c]
  (cond
   (< c  0) :cold
   (> c 30) :hot
   :else    nothing))

(def cold-or-hot-relation
  (function->propagator-constructor c->cold-or-hot))
{% endhighlight %}

We can now derive how it feels outside, and we can detect when we've
reached a conflict.

{% highlight clj %}
(-> (make-system)
    (cold-or-hot-relation :temp-c :how-it-feels)
    (add-value :temp-c -30N)
    (get-value :how-it-feels))
;; => :cold

(-> (make-system)
    (cold-or-hot-relation :temp-c :how-it-feels)
    (c-f-relation :temp-f :temp-c)
    (add-value :temp-f 100N)
    (get-value :how-it-feels))
;; => :hot

(-> (make-system)
    (cold-or-hot-relation :temp-c :how-it-feels)
    (c-f-relation :temp-f :temp-c)
    (add-value :temp-f 100N)
    (add-value :how-it-feels :cold))
;; throws ExceptionInfo Inconsistency
{% endhighlight %}

Notice, that even though our `cold-or-hot-relation` only knows how to
handle temperatures in centigrade, it still works when fahrenheit values
are fed in. This is an example of how building a system using
propagators can yield looser coupling between components, by separating
value and representation.

## Generic operators

The next thing we want to do is to extend the definition of what is
understod by conflicting values in cells of our system. However, before
we can do so, we'll have to take a quick look at *generic operators*.

A generic operator is a function which reacts differently to differnt
types of input. In OO programming, we are used to simple generic
operators which works purely on types: you pass a string to a method of
an object and one action is performed, you pass an integer, and another
action is performed. In Clojure we have multimethods for doing more
advanced matching. In multimethods you have to define your pattern
function up front. This is a constraint we can't live with when doing
propagators, and we therefore turn to generic operators.

Generic operators are functions which can be extended to cover new types
of input based on a predicate dispatch of the input
parameters. propaganda comes with a generic operator library built in
which can be used for defining new generic operators and extending
them. The library is available in the `propaganda.generic-operators`
namespace.

Here are two simple examples, both defining a new generic operator
called `plus` and extending it from numbers to also cover vectors, with
two different types of semantics.

{% highlight clj %}
(let [plus (go/generic-operator +)]
  (doto plus
    (go/assign-operation concat vector? vector?))
  [(plus 1 2)
   (plus [1 2 3] [4 5])])
;; => [3 [1 2 3 4 5]]

(let [plus (go/generic-operator +)]
  (doto plus
    (go/assign-operation (partial map +) vector? vector?))
  [(plus 1 2)
   (plus [1 2 3] [4 5])])
;; => [3 (5 7)]
{% endhighlight %}

As can be seen, extending a generic operator is a destructive action. We
do not get a new operator with the added operation, but alter the
existing operator. For a discussion of why this is done,
[please see this post](http://tgk.github.io/2013/06/generic-operator-predicate-dispatch-semantics-in-clojure.html).

The example above might seem too simple, and it can be argued that the
same behaviour could have been achieved using multimethods or
protocols. However, we will encounter more tricky situations where these
methods will not be powerful enough.


## Extending the definition of a conflict

When values propagate around our system and arrive back at a cell that
already has a value, this might cause a conflict. By default, a conflict
is simply two values that are not exactly the same. However, this is not
always what we want. For some value types, more refined definitions of a
conflict can be used, and the propagation of values around the system
can be seen as the refinement of values, not just a simple check.

In this tutorial, we will relax the definition of a conflict when the
value types in a cell is a set. In this example, we are going to define
two sets as being in conflict iff their intersection is empty.

Let's write a helper function for checking if two sets intersect. If
they do, we return their intersection. If they do and the intersection
only contains one element, we return the element. This will allow us to
mix set values and all other values. If they do not intersect, we return
a `contradiction`, which is a propaganda datatype, indication that two
values contradict each other.

{% highlight clj %}
(defn check-intersection
  [s1 s2]
  (let [i (intersection s1 s2)]
    (if (seq i)
      (if (= 1 (count i))
        (first i)
        i)
      (contradiction
       (format "Intersection of %s and %s is empty" s1 s2)))))

(check-intersection #{:foo :bar} #{:foo :bar :baz})
;; => #{:foo :bar}

(check-intersection #{:foo :bar} #{:foo})
;; => :foo

(check-intersection #{:foo} #{:baz})
;; => #propaganda.values.Contradiction{:reason "Intersection of #{:foo} and #{:baz} is empty"}
{% endhighlight %}

To fully support any type of values with sets, we also need a helper
function for checking if an element is in a set. If it is, the value is
returned, if it is not, we return a `contradiction`.

{% highlight clj %}
(defn check-in-set
  [e s]
  (if (contains? s e)
    e
    (contradiction
     (format "%s is not in %s" e s))))

(check-in-set :foo #{:foo :bar})
;; => :foo

(check-in-set :foo #{:bar})
;; => #propaganda.values.Contradiction{:reason ":foo is not in #{:bar}"}
{% endhighlight %}

propaganda determines if two values are in conflict by invoking a merge
function on them. In it's base implementation, if one of the values are
`nothing`, the other value wins. If they are different, that is a
contracdiction. We can supply our own merge function, and by extending
the base implementation using generic operators, we can add support for
sets.

{% highlight clj %}
(defn extend-merge
  [merge]
  (doto merge
    (go/assign-operation
     (fn [content increment]
       (check-in-set increment content))
     set? any?)
    (go/assign-operation
     (fn [content increment]
       (check-in-set content increment))
     any? set?)
    (go/assign-operation
     check-intersection
     set? set?)))
{% endhighlight %}

We can now maintain set values in our system, add values to refine the
understanding in our system and detect conflicts.

{% highlight clj %}
(let [my-merge (doto (default-merge)
                 extend-merge)
      my-contradictory? (default-contradictory?)]
  (-> (make-system my-merge my-contradictory?)
      (add-value :cell #{:foo :bar :baz})
      (add-value :cell #{:foo :bar})
      (get-value :cell)))
;; => #{:foo :bar}

(let [my-merge (doto (default-merge)
                 extend-merge)
      my-contradictory? (default-contradictory?)]
  (-> (make-system my-merge my-contradictory?)
      (add-value :cell #{:foo :bar})
      (add-value :cell #{:foo})
      (get-value :cell)))
;; => :foo

(let [my-merge (doto (default-merge)
                 extend-merge)
      my-contradictory? (default-contradictory?)]
  (-> (make-system my-merge my-contradictory?)
      (add-value :cell #{:foo})
      (add-value :cell #{:bar})))
;; => throws ExceptionInfo Inconsistency
{% endhighlight %}

## Conclusion

In this tutorial we have defined simple relations between numbers, we
have demonstrated how to derive facts from those numbers and to maintain
them in our system, we have studied generic operators, and we have
extended the definition of a conflict in a system.

In the next post we will show how we can use generic operators to define
more generic numerical operations which will allow us to define a
simpler version of our two-way mapping. It will also allow us to define
numerical operations on sets, immediately making the temperature
converter work across sets of values.

I appreciate that this introduction has been pretty heavy-going. My
focus has been on covering a lot of ground, rather than going in to
great detail. If you have found any step of the tutorial difficult to
understand, please do not hesitate to leave a comment or get in touch on
twitter at `@tgkristensen`.

I will post a link to the next tutorial here when it becomes available.
