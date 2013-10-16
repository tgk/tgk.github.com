---
layout: post
title: Inspecting the content of channels in core.async
---

# {{page.title}}

core.async works great for building large systems and decoupling
dependencies in a program. At [uSwitch](http://www.uswitch.com/), we're
using the core.async in several systems to balance workload and decouple
processing steps, and in the ClojureCup competition,
[our architecture](http://ragnard.github.io/2013/10/01/clojurecup-pong-async.html)
relied heavily on core.async for separating concerns, allowing us to
develop parts of the project in parallel.

In those of Rich Hickeys talks that focus on building your system like a
factory, one of the main points is, that it provides transparency to the
processes in the system. Putting your data on a virtual conveyor belt
allows you to take it off or put it back on another belt at any
time. Having worked with core.async for a while, we can testify to the
fact that putting stuff on queues and channels truely makes a
difference, but we are yet to gain transparency of our data as it
travels through our system.

This post will demonstrate methods to inspect what objects are on
core.async channels at any given time, by implementing custom buffers
and by extending the buffers supplied with core.async. The post is split
in a strategy for Clojure, and a strategy for ClojureScript, as the two
platforms have different buffer implementations.

## Clojure core.async

In core.async, when creating a new channel with `chan`, it is possible
to specify a buffer to use for the channel. There are currently three
types of buffers supplied with core.async:

- `FixedBuffer` - a buffer of a specific size. When you try to put
  things in the buffer and there is no room for them, your operation
  will block.
- `DroppingBuffer` - a buffer of a specific size. When you try to put
  things in the buffer and there is no room for them, they will be
  silently dropped.
- `SlidingBuffer` - a buffer of a specific size. When you try to put
  things in the buffer and there is no room for them, items at the head
  of the buffer will be dropped to make room for new elements.

There are no infinite buffers - if you build one, you will probably
regret it.

None of the built in buffers allows inspection of what is currently on
in buffer. They implement `ICountable`, so it is possible to see how much
is on it. By implementing our own buffer, we can build in methods for
inspecting the contents.

The `Buffer` protocol is defined in `clojure.core.async.impl.protocols`
and looks like this:

{% highlight clj %}
(defprotocol Buffer
  (full? [b])
  (remove! [b])
  (add! [b itm]))
{% endhighlight %}

Let's take the source for `FixedBuffer` in
`clojure.core.async.impl.buffers` and make our own
`TransparentFixedBuffer`:

{% highlight clj %}
(defprotocol TransparentBuffer
  (inspect [this]))

(deftype TransparentFixedBuffer [^LinkedList buf ^long n]
  impl/Buffer
  (full? [this]
    (= (.size buf) n))
  (remove! [this]
    (.removeLast buf))
  (add! [this itm]
    (assert (not (impl/full? this)) "Can't add to a full buffer")
    (.addFirst buf itm))
  clojure.lang.Counted
  (count [this]
    (.size buf))
  TransparentBuffer
  (inspect [this]
    (seq buf))
  clojure.lang.IDeref
  (deref [this]
    (seq buf)))
{% endhighlight %}

As a bonus, our buffer is `deref`-able. We can easily use our buffer
with core.async
([look here](https://github.com/tgk/observable-buffer-spike/blob/master/src/clj/observable_buffer/core.clj)
for the the complete code).

{% highlight clj %}
(def buf (transparent-fixed-buffer 2))

(def example-chan (chan buf))

(>!! example-chan :foo)
(>!! example-chan :bar)

(count buf)
;; 2
(inspect buf)
;; (:bar :foo)
@buf
;; (:bar :foo)

(<!! example-chan)

(count buf)
;; 1
(inspect buf)
;; (:bar)
@buf
;; (:bar)
{% endhighlight %}

## ClojureScript core.async

The ClojureScript implementation of core.async also contains
implementations of fixed, dropping and sliding buffers. The buffer
implementations are based on an implementation of ring buffers. Rather
than copying the entire implementation of ring buffers, we can write a
function for extracting the content of a ring buffer and extend the
available buffers.

{% highlight clj %}
;; helpers for creating a seq from a ring buffer
(defn empty-ring-buffer
  [ring-buffer]
  (loop [elms nil]
    (if (> (.-length ring-buffer) 0)
      (recur (cons (.pop ring-buffer) elms))
      elms)))

(defn fill-ring-buffer
  [ring-buffer elms]
  (doseq [e elms]
    (.unshift ring-buffer e)))

(defn ring-buffer-seq
  [ring-buffer]
  (let [elms (empty-ring-buffer ring-buffer)]
    (fill-ring-buffer ring-buffer (reverse elms))
    elms))

(defprotocol TransparentBuffer
  (inspect [this]))

(extend-type bufs/FixedBuffer
  TransparentBuffer
  (inspect [this]
    (ring-buffer-seq (.-buf this))))

(extend-type bufs/DroppingBuffer
  TransparentBuffer
  (inspect [this]
    (ring-buffer-seq (.-buf this))))

(extend-type bufs/SlidingBuffer
  TransparentBuffer
  (inspect [this]
    (ring-buffer-seq (.-buf this))))
{% endhighlight %}

Please note that these implementations make hard assumptions on the
underlying ring buffer. A more stabile implementation would re-implement
the ring buffer to be certain to have it available.

The buffers above can be used directly in the browser. This allows us to
more clearly demonstrate how the three different buffer strategies work
directly in the browser. The demo below is a full implementation using
core.async and the inspect function from above. The code is
[available at cljsfiddle](http://cljsfiddle.net/fiddle/tgk.observable-buffer).

<iframe src="http://cljsfiddle.net/view/tgk.observable-buffer"
        seamless="true"
        height="250">
</iframe>

Try the different buffer strategies to see the differences when more
data is pushed through than will fit.

## Conclusion

This post has illustrated how to implement custom buffers as well as
extending existing buffers to gain insight into what is on the channels
they represent. It should be noted that the number of items can be
inspected on the existing channels with no alternations.

An open problem is to monitor the throughput of a channel. Using the
inspection tricks presented here only gives us snapshots of the state,
but we do not gain any insight into statistics such as average time an
item spends on a channel.

Thanks to Jonas Enlund for having created
[cljsfiddle](http://cljsfiddle.net/). It's a great way to play around
with ClojureScript and to share example code.
