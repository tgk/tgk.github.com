---
layout: post
title: Creating Datomic Data Sources
---

# {{page.title}}

Queries in Datomic are primarily performed on Datomic databases, but other sources can also be used. Anything that implements the `clojure.lang.Indexed` protocol is queryable by the `datomic.api/q` function.

A commonly used example is vectors of vectors. Stuart Halloway has assembled some interesting examples in [this gist](https://gist.github.com/2645453). An example taken from there is presented here

{% highlight clj %}
(d/q '[:find ?first ?height
       :in $a $b
       :where [$a ?last ?first ?email]
              [$b ?email ?height]]
     [["Doe" "John" "jdoe@example.com"]
      ["Doe" "Jane" "jane@example.com"]]
     [["jane@example.com" 73]
      ["jdoe@example.com" 71]])
;; => #<HashSet [["Jane" 73], ["John" 71]]>
{% endhighlight %}

This strategy requires you to structure your data in such a way that the attribute parameter is removed from the `:where` clauses. Using maps as entities instead would allow us to have more structured information as input. In [this recent thread](https://groups.google.com/forum/?fromgroups=#!topic/datomic/cBlhFSLDR64) Rich Hickey suggests providing an indexed view.

{% highlight clj %}
(defn maprel [maps & keys] 
  (let [ik (zipmap (range) keys)] 
    (map #(reify clojure.lang.Indexed 
            (nth [_ i] (if (< i (count keys)) 
                         (get % (ik i)) 
                         (throw (IndexOutOfBoundsException.)))) 
            (nth [_ i not-found] (get % (ik i) not-found))) 
         maps)))
(d/q '[:find ?b :where [?c ?b]] 
     (maprel [{:a 1 :b 2 :c 3} 
              {:a 11 :b 12 :c 13} 
              {:a 21 :b 22 :c 23} 
              {:a 31 :b 32 :c 33}] 
             :c :b))
;; => #<HashSet [[2], [22], [12], [32]]>
{% endhighlight %}

This approach would force us to specify the used keys in the function call, and the attributes are no longer available in the `:where` clauses. Furthermore, we cannot query for the original entities (the maps) anymore. Instead, if we create an Entity-Attribute-Value (EAV) relation for each key/value pair in the input maps, we can retain the syntax normally used on Datomic databases.

{% highlight clj %}
(defn maps-eav [maps]
  (apply concat
         (for [map maps]
           (apply concat
                  (for [[a v] map]
                    (if (coll? v)
                      (for [sub-v v]
                        [map a sub-v])
                      [[map a v]]))))))
(d/q '[:find ?b
       :where
       [?m :b ?b]
       [?m :c ?c]]
     (maps-eav
      [{:a 1 :b 2 :c 3} 
       {:a 11 :b 12 :c 13} 
       {:a 21 :b 22 :c 23} 
       {:a 31 :b 32 :c 33}]))
;; => #<HashSet [[2], [22], [12], [32]]>
{% endhighlight %}

This approach allows us to perform more complicated joins in a Datomic database idiomatic way. For example, if we have a collection of maps representing persons and their likes, and another collection representing their names and emails, we can extract the emails of all that like chocolate.

{% highlight clj %}
(d/q '[:find ?email
       :in $1 $2
       :where
       [$1 ?p1 :likes "Chocolate"]
       [$1 ?p1 :name ?name]
       [$2 ?p2 :person/name ?name]
       [$2 ?p2 :person/email ?email]]
     (maps-eav 
      [{:name "Charlie", :likes ["Chocolate"]}
       {:name "Willy Wonka", :likes ["Carrots"]}])
     (maps-eav
      [{:person/name "Charlie", :person/email "charlie@factory.co.uk"}
       {:person/name "Roald Dahl", :person/email "roald@dahl.com"}]))
;; => #<HashSet [["charlie@factory.co.uk"]]>
{% endhighlight %}

Joining data across Datomic databases and in-memory collections of maps becomes much easier to write and read. Furthermore, data with the same syntax can be read from either type of source, without the query itself needing to be written in a way that reflects this.

This post has illustrated ways in which existing data can be converted to EAV relations which can be combined with other Datomic data sources to perform advanced queries. Even without a Datomic transactor, it is possible to do quite advanced experiments. It also makes it possible to try out Datomic queries and concepts without having to define a schema for the data.