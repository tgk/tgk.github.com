---
layout: post
title: A student or topic roulette in ClojureScript
---

# {{page.title}}

Students hate being pointed out and asked to answer a question. The fact that another person, the teacher, has pointed them out makes it seem unfair, and they often dislike the situation. Faced with this dilemma, my girl friend came up with a roulette system based on powerpoint: each slide has a students name (or a topic) and the slides are shifted automatically. The class is asked to say 'stop', and when they do so the presentation is halted and the lucky student is asked. The same system can be used for selecting topics or other similar exercises.

Now, setting up these slides can be quite time consuming as they have to be linked from slide to slide. Also, sharing roulettes can be cumbersome in that a mail has to be sent to all interested parties, containing an attachment which may or may not get picked up by the school mail filter.

To help out (and to learn ClojureScript) I voluntered to write a webpage with the same functionality as the slideshow. In this post I will go through and describe the single ClojureScript file which constitutes the entire program logic. My hope is that this will help other eager developers in getting started with ClojureScript.

The roulette consists of an `index.html` file, some bootstrap css and a single ClojureScript file.

The `index.html` file consists of a text area for entering names or topics, a div containing a mirrored bullet list of the names, a button for starting the slide show, a button for stopping it, a div with the current value of the roulette, and a button for getting back to the text area where names can be changed.

The ClojureScript part uses hiccups for formatting the list of names and jayq (based on JQuery) for dom manipulation. The first part of the script defines the dom elements that are to be manipulated.

{% highlight clj %}
(def input-area (jq/$ :#input-area))
(def mirrored-text (jq/$ :#mirrored-text))
(def configuration-view (jq/$ :#configuration-view))
(def current-item (jq/$ :#current-item))
(def roulette-view (jq/$ :#roulette-view))
(def stop-button-view (jq/$ :#stop-button-view))
(def redo-button-view (jq/$ :#redo-button-view))
{% endhighlight %}

Next follows convenience functions for hiding and showing groups of dom elements, corresponding to different views. If you know JQuery this should look familiar.

{% highlight clj %}
(defn hide-all []
  (jq/hide configuration-view)
  (jq/hide roulette-view)
  (jq/hide stop-button-view)
  (jq/hide redo-button-view))
  
(defn show-configuration []
  (hide-all)
  (jq/show configuration-view))

(defn show-roulette []
  (hide-all)
  (jq/show roulette-view)
  (jq/show stop-button-view))

(defn show-redo []
  (hide-all)
  (jq/show roulette-view)
  (jq/show redo-button-view))
{% endhighlight %}

The roulette supports getting the list of names from the URL. This is achieved by parsing the hash tag part of the URL. This part of the script uses `.-` prefixed functions (`.-length` and `.-hash`), which correspond directly to JavaScript functions. The function `update-according-to-url` parses and sets content of the input area, the `update-url-according-to-input` updates the hash part of the url according to the content of the input area.

{% highlight clj %}
(defn parse-url [hash-part]
  (when (> (.-length hash-part) 0)
    (js/decodeURIComponent (.substring hash-part 1))))

(defn update-according-to-url []
  (when-let [new-content (parse-url (.-hash js/location))]
    (jq/inner input-area new-content)))

(defn update-url-according-to-input []
  (set! (.-hash js/location) (js/encodeURIComponent (jq/val input-area))))
{% endhighlight %}

Next comes functions for converting the input area content to a list of items. The content of the input area is interpreted such that each line contain an item. Each item is a link if the last element is a URL. `extract-items` splits the string up based on new-lines and maps `decorate-item` on the elements. `decorate-items` generates a hiccup element for each item, either a link or a normal text element. The helper function `extract-href` parses a line; if the lines last element begins with `http` a map with `:text` and `:href` is returned, if it does not, only a `:text` part is returned.

{% highlight clj %}
(defn extract-href [s]
  (let [tokens (string/split s " ")]
    (if (= 0 (.indexOf (last tokens) "http"))
      {:text (.substring s 0 (- (.-length s) (.-length (last tokens)))) :href (last tokens)}
      {:text s})))

(defn decorate-item [item]
  (let [{text :text href :href} (extract-href item)]
    (if href [:a {:href href} text] text)))

(defn extract-items [s]
  (map decorate-item (string/split s "\n")))
{% endhighlight %}

The functions described above are then used for extracting the current items from the input area and mirroring them. `mirror-text` both updates the URL and the bullet list. The bullet list is created using the hiccups macro `hiccups/defhtml`.

{% highlight clj %}
(defn current-items []
  (extract-items (jq/val input-area)))

(hiccups/defhtml list-items [items]
  [:ul
    (for [item items]
      [:li item])])

(defn mirror-text []
  (update-url-according-to-input)
  (jq/inner mirrored-text (list-items (current-items))))
{% endhighlight %}

Initialization of the page is handled by the init function which makes sure the configuration screen is shown initially, it loads the input area according to the URL, mirrors the text and binds `mirror-text` such that future input in the input area is mirrored imideatly. To be able to call `init`, the function has to be marked with `^:export` meta-data.

{% highlight clj %}
(defn ^:export init []
  (show-configuration)
  (update-according-to-url)
  (mirror-text)
  (jq/bind input-area :input mirror-text))
{% endhighlight %}

The items are held in the atom `items`. When the roulette is reset, `items` is set to an infinit list of the current items, extracted from the input area. Running the roulette a step corresponds to removing the first item of this infinit list.

{% highlight clj %}
(def items (atom nil))

(defn reset-roulette! []
  (reset! items (cycle (current-items))))

(defn change-to-next! []
  (swap! items rest))

(defn ^:export show-next! []
  (jq/inner current-item (hiccups/html (first (change-to-next!)))))
{% endhighlight %}

Running through the items when the roulette is running is handled by a JavaScript timer. When `setInterval` from JavaScript is called, a reference to the timer is returned. This value is stored in the `worker` atom. The `start` function has a `swap!` call which ensures that the current timer, if any, is stopped, and a new started. The `swap!` section of `stop` makes sure that the current timer, if any, is stopped. Both `start` and `stop` are exported so they can be called from the `index.html` file.

{% highlight clj %}
(def worker (atom nil))

(defn ^:export start []
  (show-roulette)
  (reset-roulette!)
  (swap! worker
   (fn [worker-val]
      (when worker-val
           (.clearTimeout js/window worker-val))
	      (.setInterval js/window show-next! 100))))

(defn ^:export stop []
  (show-redo)
  (swap! worker
   (fn [worker-val]
      (when worker-val
           (.clearTimeout js/window worker-val)
	        nil))))
{% endhighlight %}

The final piece is a function for the "Change" button which takes the user back to the configuration screen.

{% highlight clj %}
(defn ^:export change []
  (show-configuration))
{% endhighlight %}

That's all there is to it. The project also contains a basic `project.clj` file and the `index.html`. The entire project is available at github.com/tgk/roulette. A running version of the roulette can be accessed from tgk.github.com/roulette.
