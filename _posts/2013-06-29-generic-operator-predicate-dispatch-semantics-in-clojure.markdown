---
layout: post
title: Generic operator predicate dispatch semantics in Clojure
---

# {{page.title}}

Lately, I have been working on a propagation library in Clojure as a [Lisp in Summer Project](http://lispinsummerprojects.org/). In doing so, I found the need for a generic operator predicate dispatch system that would fit into a general library, and I've therefore worked on some different generic operator implementation strategies.

I found that different semantics of generic operator strategies leads to different usage trade-offs. Performance is only one side of the story; [core.match](https://github.com/clojure/core.match/) has efficient predicate dispatch as a final goal, but it isn't there yet, and even if it was done, it wouldn't nessecarialy fit the particular needs of this project.

In this post I will go through some different strategies for implementing generic operators based on predicate dispatch and their usage trade-offs. The aim is not to present efficient implementations, but to present different use-cases and possible solutions.

## Problem description

In this particular case, I needed a predicate dispatch system that would allow me to perform operator overloading using predicates without tarnishing the runtime. These two reasons rule out the use of multimethods, as they do not support predicate dispatch, and they tarnish the runtime.

In an ideal scenario, it should be possible to do something like this:

{% highlight clj %}
;; default operator is just +
(def basic-+ (generic-operator +))

;; we add string concationation
(def str-+ (assign-operation
            basic-+
            str
            string? string?))

;; we define + of vectors to be + of pairs of elements
(def str&vec-+ (assign-operation
                str-+
                (fn [v1 v2]
                  (for [[val1 val2] (map vector v1 v2)]
                    ;; A special recur-op is needed as we do not know the
                    ;; name of the final operator
                    (recur-op val1 val2)))
                vector? vector?))

;; and add an odd meaning of + of even numbers
(def str&vec&even-+ (assign-operation
                     str&vec-+
                     (comp inc +)
                     #(and (number? %) (even? %)) #(and (number? %) (even? %))))

;; the basic operation works as expected
(str-+ "foo" "bar")
;; => "foobar"

;; the odd str&vec&even-+ works
(str&vec&even-+ [2 "foo"] [4 "bar"])
;; => (7 "foobar")

;; but it hasn't destroyed the meaning of str-+
(str-+ 2 4)
;; => 6

;; or str&vec-+
(str&vec-+ [2 "foo"] [4 "bar"])
;; => (6 "foobar")
{% endhighlight %}

How to get there is non-trivial. Here are two approaches that each get some of the way, but both fall short of the final solution.

## Simple recursion

Using simple recursion to create new functions, much like wrappers in Ring, allows for different extensions of a generic operator without tarnishing the runtime, but recursion is not possible.

{% highlight clj %}
;;; Implementation:

(defn all-preds?
  "Returns true iff (pred vals) is truthy for all preds paired with
  vals."
  [preds vals]
  (and (= (count preds) (count vals))
       (every?
        identity
        (for [[pred val] (map vector preds vals)] (pred val)))))

(defn assign-operation
  "Takes a generic operator and returns a new genric operator where
  operation is invoked if preds are truthy for arguments passed to
  operator. If one or more predicates are not thruthy, the original
  generic-operator is tried. A plain function can be used as initial
  generic-operator."
  [generic-operator operation & preds]
  (fn [& args]
    (if (all-preds? preds args)
      (apply operation args)
      (apply generic-operator args))))

(def str-+ (assign-operation
            +
            str
            string? string?))

(def str&vec-+ (assign-operation
                str-+
                (fn [v1 v2]
                  (for [[val1 val2] (map vector v1 v2)]
                    ;; No operator recur-target available, so we must use str+
                    ;; which hinders us in reaching operators added in
                    ;; later steps
                    (str-+ val1 val2)))
                vector? vector?))

(def str&vec&even-+ (assign-operation
                     str&vec-+
                     (comp inc +)
                     #(and (number? %) (even? %)) #(and (number? %) (even? %))))

;;; Usage:

(str-+ "foo" "bar")
;; => "foobar" - OK

(str&vec&even-+ [2 "foo"] [4 "bar"])
;; => (6 "foobar") - NOT what we wanted (7 "foobar")

(str-+ 2 4)
;; => 6 - OK

(str&vec-+ [2 "foo"] [4 "bar"])
;; => (6 "foobar") - OK
{% endhighlight %}


## Function with predicates stored in meta-data

We can store the predicates in the meta-data for the generic operator instead. This will gain us the ability to recur, but we will no longer be able to use old versions of a generic operator. Every time we add an operator with predicates to our generic operator, this is what will be seen by the system. This somewhat resembles how multimethods work, but we can use general predicates.

{% highlight clj %}
;;; Implementation:

(defn all-preds?
  "Returns true iff (pred vals) is truthy for all preds paired with
  vals."
  [preds vals]
  (and (= (count preds) (count vals))
       (every?
        identity
        (for [[pred val] (map vector preds vals)] (pred val)))))

(defn val-with-predicates
  "Returns the first val in pred&vals seq where all preds saitsfy args."
  [pred&vals & args]
  (second
   (first
    (filter
     (fn [[preds _]] (all-preds? preds args))
     pred&vals))))

(defn execute-op
  "Choses an operation from pairs of predicates and operators and
  executes it on args. Executes default on args if no predicates match."
  [default pred&ops & args]
  (if-let [op (apply val-with-predicates pred&ops args)]
    (apply op args)
    (apply default args)))

(defn generic-operator
  "Returns a generic operation with default operator default."
  [default]
  (let [pred&ops (atom nil)]
    (with-meta
      (fn [& args]
        (apply execute-op default @pred&ops args))
      {:pred&ops pred&ops})))

(defn assign-operation
  "Alters generic-operator, adding operation given preds."
  [generic-operator operation & preds]
  (let [preds&ops (:pred&ops (meta generic-operator))]
    (swap! preds&ops conj [preds operation])))

;;; Usage:

;; We define our generic operator, but we only have one reference
(def generic-+ (generic-operator +))

;; We now loose the intermediate generic operation and tarnish our runtime
(assign-operation generic-+
                  str
                  string? string?)

;; ... but recursion works
(assign-operation generic-+
                  (fn [v1 v2]
                    (for [[val1 val2] (map vector v1 v2)]
                      (generic-+ val1 val2)))
                  vector? vector?)

(assign-operation generic-+
                  (comp inc +)
                  #(and (number? %) (even? %)) #(and (number? %) (even? %)))

(generic-+ "foo" "bar")
;; => "foobar" - OK

;; the odd generic-+ works
(generic-+ [2 "foo"] [4 "bar"])
;; => (7 "foobar") - OK

;; but it has destroyed the earlier meaning, and we can't get it back
;; without creating a new operator
(generic-+ 2 4)
;; => 7 - NOT 6, and we are unable to re-create that generic operator
{% endhighlight %}

## Conclusion

This post described two types of semantics for generic operators using predicate dispatch in Clojure, but was unable to give an elegant solution to the original problem definition. The question is, does the original problem definition even describe an approach that is desirable to use as a generic operator system?

In my use case, I want to supply users with a particular generic operator, which they need to extend tu support to add new datatypes or behaviours to the library. Given these constraints, the different options can be summarised as follows:

- Multimethods - Well understod by users. Only support type dispatch. Tarnishes generic operator (but a new one can be generated using macros). Support recursion. Implemented in Clojure.
- Simple recursion - Simple for users to understand. Supports predicate dispatch. Does not tarnish generic operator. Does not support recursion. Implemented and described here.
- Function with predicates in meta-data - Simple for users to understand. Supports predicate dispatch. Tarnishes generic operator (but a new one can be returned by function-call). Supports recursion. Implemented and described here.
- Solution from problem definition - Harder for users to understand. Supports predicate dispatch. Doe not tarnish generic operator. Supports recursion. Is not currently implemented.

Given these properties, I think I will go forward with the meta-data approach. If you need generic operators with predicate dispatch, I suggest you try out a couple of different strategies. If you come up with a nice solution for the original problem, do get in touch, thanks!
