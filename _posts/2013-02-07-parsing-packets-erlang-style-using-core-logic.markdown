---
layout: post
title: Parsing packets Erlang style using core.logic
---

# {{page.title}}

Attending the [Zurich Erlang User Group](http://www.meetup.com/Zurich-Erlang-User-Group/) meetup yesterday, I was struck by how easy it is to parse packets of data in Erlang using the `<<...>>` construct. Here is a nice example which parses a size, and then uses that size to parse a chunk of data:

{% highlight erl %}
<<Size:4/binary,C:Size,_Rest>> = Data.
{% endhighlight %}

I have previously tried to tackle the same problem in Clojure using core.match with little luck. Seeing Lars pulling data out from his watch yesterday, and parsing it in Erlang inspired me to give it another go, this time using core.logic.

If you want to hack along, the full source is at [github](https://github.com/tgk/parsing-packet-with-logic).

## Introducing `packeto`

The result of my attempt is a core.logic goal called `packeto` which matches a packet with a pattern. The following example will extract a binary of four bits, a chunk of data of 6 bits and bind the rest to an unused lvar.

{% highlight clj %}
(run 1 [q]
     (fresh [b c
             rest-size rest-chunk]
            (packeto [1 0 0 1 0 1 0 1 1 1 1 1]
                     [:binary 4 b
                      :chunk 6 c
                      :chunk rest-size rest-chunk])
            (== q [b c])))
;; => ([9 (0 1 0 1 1 1)])
{% endhighlight %}

Since this is core.logic, we can do more sophisticated things, such as using vars that have been bound other places in our expression. For example, we can let the size of a chunk be defined by a previous matched part of the packet.

{% highlight clj %}
(run 1 [q]
     (fresh [b c
             rest-size rest-chunk]
            (packeto [1 0 0 1 0 1 0 0 1 0 1 1 1 1 1 1]
                     [:binary 4 b
                      :chunk b c
                      :chunk rest-size rest-chunk])
            (== q [b c])))
;; => ([9 (0 1 0 0 1 0 1 1 1)])
{% endhighlight %}

And since this is logic programming, we can run our program backwards and generate packets as well.

{% highlight clj %}
(run 1 [q]
     (fresh [b c]
            (packeto q
                     [:binary 4 b
                      :chunk b [0 1 0 1 0 1 0 1 1]])))
;; => ((1 0 0 1 0 1 0 1 0 1 0 1 1))
{% endhighlight %}

## The algorithm

There are two minor helping goals. `counto` for unifying a sequence and its length, and binaryo for unifying a binary representation of a number and the value of that number.

{% highlight clj %}
(defne counto [l n]
  ([() 0])
  ([[h . t] _]
     (fresh [m]
            (fd/in m (fd/interval 0 Integer/MAX_VALUE))
            (fd/+ m 1 n)
            (counto t m))))

(defne binaryo [p n]
  ([() 0])
  ([[h . t] _]
     (fresh [m m-times-two]
            (fd/in m (fd/interval 0 Integer/MAX_VALUE))
            (fd/in h (fd/interval 0 1))
            (binaryo t m)
            (fd/* m 2 m-times-two)
            (fd/+ m-times-two h n))))
{% endhighlight %}

My knowledge of finite domains is pretty limited. Any feedback on making these two functions smarter is highly appreciated.

The `packeto` relation itself is actually quite simple given these tools. Chunks of bits are easy enough to extract, and binary values can be extracted by first extracting chunks and then converting them.

{% highlight clj %}
(defne packeto [p pattern]
  ([() ()])
  ([_ [:chunk n c . t]]
     (fresh [pt]
            (appendo c pt p)
            (counto c n)
            (packeto pt t)))
  ([_ [:binary n v . t]]
     (fresh [c np]
            (appendo [:chunk n c] t np)
            (packeto p np)
            (binaryo c v))))
{% endhighlight %}

And that's all that is needed for performing packet matchings as described in the introduction.

## Conclusion

It has been very interesting to show how flexible Clojure and core.logic is, and how they can be used to easily create features highly praised in other languages. The entire code of this exercise, including examples, can be found [here at github](https://github.com/tgk/parsing-packet-with-logic). 

I know that there are some patterns that are missing for this tool to be as sophisticated as the Erlang packet matcher, one of them being the possibility to distinguish between big-endian and little-endian numbers. Adding these keywords and goals to `packeto` should be easy enough, but I have not done it here, to keep the example short and clear.

Another interesting extra goal that can be added is a checksum part of the packet. This is simply another goal that can be bound to the lvar representing the checksum in the packet.

