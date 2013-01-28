---
layout: post
title: edn service spike
---

# {{page.title}}

Rich Hickey has been advocating that services should be highly decoupled with a common, extensible format, such as edn. Fogus has a [great library](https://github.com/fogus/ring-edn) for exposing a REST service with the `application/edn` content type, and Edmund Jackson has a [blog post](http://boss-level.com/?p=119) about transmitting edn data through URLs, but neither example demonstrates a full stack service using edn. There are other full stack alternatives, such as [shoreleave](https://github.com/shoreleave/shoreleave-remote), but these abstract away the communication protocol, creating a tighter coupling between the frontend and backend of an application.

This blog post presents a REST service which expects an edn map with a `:name` key. The service reverses the name and returns it as an edn map containing a `:message` key.

The full spike with a `project.clj` file and instructions for testing the spike can be found [at github](https://github.com/tgk/edn-spike).

## Clojure Ring app

The Ring app contains both the REST service and the static files for the frontend.

{% highlight clj %}
(ns spike.backend
  (:use compojure.core
        compojure.route
        ring.middleware.edn))

(defn generate-response [data & [status]]
  {:status (or status 200)
   :headers {"Content-Type" "application/edn"}
   :body (pr-str data)})

(defroutes main-routes
  (GET "/" req (slurp "resources/public/index.html"))
  
  (GET "/rest" req
       (generate-response {:message "Hello stranger"}))

  (POST "/rest" {edn-params :edn-params}
        (generate-response
         {:message (str "Hello " (->> edn-params :name reverse (apply str)))}))

  (resources "/"))

(def handler
  (-> main-routes
      wrap-edn-params))
{% endhighlight %}

The `:edn-params` in the request map is supplied by Fogus' `ring-edn` [project](https://github.com/fogus/ring-edn), from which the `generate-response` function is also taken.

## ClojureScript frontend

The ClojureScript part of the spike is highly inspired by Edmund Jackson's [Minimal Ajaxy ClojureScript](https://github.com/ejackson/Minimal-Ajaxy-Closurescript) project with [accompanying blog post](http://boss-level.com/?p=119).

{% highlight clj %}
(ns spike.frontend
  (:require
   [cljs.reader :as reader]
   [goog.dom :as dom]
   [goog.events :as ev]
   [goog.net.XhrIo :as xhr]))

(defn- extract-response [message]
  (reader/read-string
   (. message/target (getResponseText))))

(defn- insert-into-dom
  [id val]
  (let [target (dom/getElement id)]
    (dom/setTextContent target val)))

(defn callback
  [message]
  (let [result (extract-response message)]
    (insert-into-dom "result" (:message result))))

(defn edn-call
  [path callback method data]
  (xhr/send path
            callback
            method
            (pr-str data)
            (clj->js {"Content-Type" "application/edn"})))

(defn send-request [e]
  (let [body {:name (.-value (dom/getElement "name"))}]
    (edn-call "/rest" callback "POST" body)))

(defn ^:export init []
  (ev/listen	
   (dom/getElement "button")
   "click"
   send-request))
{% endhighlight %}

One of the most important parts of the above code is setting the `Content-Type`. Without that, the `:edn-params` would not catch the data.

## The HTML

The HTML itself is pretty straightforward.

{% highlight html %}
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML//EN">
<html>
<head>
<title>edn spike</title>
<link href="//netdna.bootstrapcdn.com/twitter-bootstrap/2.2.2/css/bootstrap-combined.min.css" rel="stylesheet">
</head>

<body>

<input class="input-small search-query" id="name" value="thomas" type="text" placeholder="Value to send" autofocus />
<button class="btn" id="button" type="submit">Call service</button>
<span id="result"></span>

<script src="/js/compiled.js"></script>

<script type="text/javascript">
spike.frontend.init();
</script>

</body>

</html>
{% endhighlight %}

## Conclusion

This post has demonstrated a minimal example of a REST stack utilising edn with a Clojure backend and a ClojureScript frontend. While it might initially seem like a big task, it is acutally quite simple to get up and running, even without frameworks such as [shoreleave](https://github.com/shoreleave/shoreleave-remote).

As mentioned previously, the full example can be found [here at github](https://github.com/tgk/edn-spike).