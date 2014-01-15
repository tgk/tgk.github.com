---
layout: post
title: Taking propagators to the next level
---

# {{page.title}}

## Introduction

In
[my previous post about propagators](http://tgk.github.io/2014/01/getting-hot-with-propagators.html),
we looked at how to define simple one- and two-way relations and we
looked at generic operators, which were used extending the definition of
conflicts to include sets. In this post, we will examine how to define
mathemtical relations between cells containing not only simple values,
but also sets. We will use these relations to build a version of the
celcius to fahrenheit relation that works not only between cells
containing numbers, but also between cells containing sets.

All the code here uses version 0.2.0 of
[the propaganda library](https://github.com/tgk/propaganda). The code in
it's full form is available
[at github](https://github.com/tgk/getting-hot).

## Defining generic mathemtical operations

We want generic versions of addition, subtraction, multiplication and
division that will accept two numbers, a set and a number, and two
sets. To do this, we write a function `generic-set-operator` that takes
an operator and returns a generic operator based on the given operator.

{% highlight clj %}
(defn generic-set-operator
  [op]
  (doto (go/generic-operator op)
    (go/assign-operation
     (fn [s v] (into #{} (for [elm s] (op elm v))))
     set? any?)
    (go/assign-operation
     (fn [v s] (into #{} (for [elm s] (op v elm))))
     any? set?)
    (go/assign-operation
     (fn [s1 s2] (into #{} (for [e1 s1 e2 s2] (op e1 e2))))
     set? set?)))

(def plus (generic-set-operator +))
(def minus (generic-set-operator -))
(def multiply (generic-set-operator *))
(def divide (generic-set-operator /))

(plus 1 2)
;; => 3
(plus #{1 2 3} 4)
;; => #{5 6 7}
(plus 4 #{1 2 3})
;; => #{5 6 7}
(plus #{1 2 3} #{1 2 3})
;; => #{2 3 4 5 6}
{% endhighlight %}

We can lift the operators to propagator constructors.

{% highlight clj %}
(def plusp (function->propagator-constructor plus))
(def minusp (function->propagator-constructor minus))
(def multiplyp (function->propagator-constructor multiply))
(def dividep (function->propagator-constructor divide))
{% endhighlight %}

Finally, we can create the two-way relations for sums and products.

{% highlight clj %}
(defn sum
  [system a b c]
  (-> system
      (plusp a b c)
      (minusp c a b)
      (minusp c b a)))

(defn prod
  [system a b c]
  (-> system
      (multiplyp a b c)
      (dividep c a b)
      (dividep c b a)))

(-> (make-system)
    (prod :a :b :c)
    (add-value :a 10)
    (add-value :b 2)
    (get-value :c))
;; => 20

(-> (make-system)
    (prod :a :b :c)
    (add-value :a 2)
    (add-value :c 84)
    (get-value :b))
;; => 42
{% endhighlight %}

## Defining the relation between temperatures

In the previous post, we did not have sum and product relations, so we
had to write two function for defining the two-way relation. This time
around, we can use these new propagator constructors to define our
relation.

{% highlight clj %}
(defn c-f-relation
  [system f c]
  (let [minus-32    (gensym "minus-32")
        five-ninths (gensym "five-ninths")
        f-minus-32  (gensym "f-minus-32")]
    (-> system
        (add-value minus-32 -32)
        (add-value five-ninths 5/9)
        (sum f minus-32 f-minus-32)
        (prod five-ninths f-minus-32 c))))
{% endhighlight %}

The constants we need to define are stored under three new cells
allocated for the purpose. `gensym` is used to come up with unique
identifiers for those cells.

The `c-f-relation` function works both ways, so our examples from the
previous post will still give us the results we want.

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
    (add-value :temp-c 38N)
    (get-value :temp-c))
;; throws ExceptionInfo Inconsistency
{% endhighlight %}

Using the `extend-merge` function from the previous post, we can have
our relations working on sets, as `c-f-relation` only uses generic
operators that work on both sets and numbers.

{% highlight clj %}
(let [my-merge (doto (default-merge)
                 extend-merge)
      my-contradictory? (default-contradictory?)]
  (-> (make-system my-merge my-contradictory?)
      (c-f-relation :temp-f :temp-c)
      (add-value :temp-f #{100N 200N})
      (get-value :temp-c)))
;; => #{280/3 340/9}
{% endhighlight %}

As conflicts only occur when the there is an empty intersection between
sets, we can have several restrictions on the same cell to refine the
value of the temperature in celcius

{% highlight clj %}
(let [my-merge (doto (default-merge)
                 extend-merge)
      my-contradictory? (default-contradictory?)]
  (-> (make-system my-merge my-contradictory?)
      (c-f-relation :temp-f :temp-c)
      (add-value :temp-f #{100N 200N})
      (add-value :temp-c #{110/4 280/3})
      (get-value :temp-c)))
;; => 280/3
{% endhighlight %}

And, as data flows both ways, the temperature in fahrenheit will
automatically have been restricted.

{% highlight clj %}
(let [my-merge (doto (default-merge)
                 extend-merge)
      my-contradictory? (default-contradictory?)]
  (-> (make-system my-merge my-contradictory?)
      (c-f-relation :temp-f :temp-c)
      (add-value :temp-f #{100N 200N})
      (add-value :temp-c #{110/4 280/3})
      (get-value :temp-f)))
;; => 200N
{% endhighlight %}

## Conclusion

In this second tutorial we have explored how to define mathemtical
relations that not only operate on numbers, but also across compound
data structures, in this case sets. We have redefined our temperature
relation to use generic two-way operators which has two consequences: we
do not need to define the bijective relation explicitly, and the
relation will work on both numbers and sets.

I hope you've enjoyed reading through the tutorial. If you're interested
in other applications of propagators, I can recommend the article
[The Art of the Propagator](http://dspace.mit.edu/handle/1721.1/44215).
