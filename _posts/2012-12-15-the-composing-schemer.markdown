---
layout: post
title: The Composing Schemer
---

# {{page.title}}

One of the great things about the Clojure libraries is, that they are highly orthogonal. It's easy to build systems for experimenting by using different, great libraries.

In this post I am going to demonstrate how I used Overtone, Quil and core.logic to explore tonal melodies. I have no formal musical education - I used to play the keyboard, but that was back when 2 Unlimited's No Limits was considered cool. I started in the evening and posted this blog post the next day, even sparing time for a trip to IKEA. I'm very impressed by how fast it was to build.

Before I start, I would just like to state that I am in no way a musician or composer, and I have little grasp of the musical concepts. Any errors in use of terminology is fully my fault, and I am very sorry if they confuse more than explain.

If you want to play along, the full code can be found [here at github](https://github.com/tgk/the-composing-schemer).

## The video

Writing about something musical can be a bit difficult without some example. In [this video](https://vimeo.com/55677313), I demonstrate what I came up with. I will go through selections of the code below.

## Overtone 

I must confess I've never played with Overtone before. I like to blame the fact that I don't feel very musical, even though Sam Aaron repeatedly state that that is not the point. Reading through the tutorial, I found the definition of a triangle-wave, and instructions for how to play notes.

{% highlight clj %}
(definst triangle-wave [freq 440 attack 0.01 sustain 0.1 release 0.4 vol 0.4] 
  (* (env-gen (lin-env attack sustain release) 1 1 0 1 FREE)
     (lf-tri freq)
     vol))

(triangle-wave)

(defn note->hz [music-note]
  (midi->hz (note music-note)))

(defn triangle2 [music-note]
  (triangle-wave (midi->hz (note music-note))))

(triangle2 :A4)
(triangle2 :C5)
(triangle2 :C4)
{% endhighlight %}

I figured out how to set up a metronome, and created an atom for holding my melody. The atom is our "sketch board", which makes it extremely easy to play around. To test a new melody, simply reset the atom.

{% highlight clj %}
(defonce metro (metronome 200))
(metro)

(def m-atom (atom [:C4]))

(defn chord-progression-atom [m beat-num melody-atom]
  (println @melody-atom)
  (doseq [[beat note] (zipmap (range) @melody-atom)]
    (at (m (+ beat beat-num)) (triangle2 note)))
  (apply-at
   (m (+ (count @melody-atom) beat-num))
   chord-progression-atom
   m
   (+ (count @melody-atom) beat-num) melody-atom []))

(defn play []
  (chord-progression-atom metro (metro) m-atom))

(play)
(stop)

(reset! m-atom [:D#4 :G#4 :D5 :F4 :B4 :F#4 :A#4 :D#4])
{% endhighlight %}

That's all the pieces we need from Overtone.

## Quil

My girlfriend helped me out with the musical theory (which I knew nothing about before starting), but she thought reading keyword sequences like `[:D#4 :G#4 :D5 :F4 :B4 :F#4 :A#4 :D#4])` was difficult. I created a stave in Quil that uses the melody atom. I'm not going in to details as to how it works - I've been told that the stave is drawn an octave above what is actually playing. Also, everything is shown as sharps with a `#`. Doing flats with require some more insight into sheet music.

I'm not going into details with the drawing routines themselves. They are pretty basic.

{% highlight clj %}
(defn setup []
  (q/smooth)
  (q/frame-rate 1)
  (q/background 255))

(def white-notes
  [:C3 :D3 :E3 :F3 :G3 :A3 :B3
   :C4 :D4 :E4 :F4 :G4 :A4 :B4
   :C5 :D5])

(def black-notes
  [:C#3 :D#3 :NO :F#3 :G#3 :A#3 :NO
   :C#4 :D#4 :NO :F#4 :G#4 :A#4 :NO])

(defn draw []
  (q/background 255)
  (q/stroke (q/color 0))
  (let [positions (merge
                   (zipmap white-notes (range))
                   (zipmap black-notes (range)))
        x (fn [pos] (+ 50 (* 30 pos)))
        y (fn [pos] (+ -20 (- (q/height) (* 7 pos))))
        line (fn [note] (q/line (x -1) (y (positions note))
                                (x  8) (y (positions note))))]
    (doseq [[pos note] (zipmap (range) @m-atom)]
      (q/fill (q/color 255))
      (q/ellipse
       (x pos)
       (y (positions note))
       14 11)
      (when (even? (positions note))
        (q/line (- (x pos) 10) (y (positions note))
                (+ (x pos) 10) (y (positions note))))
      (when (contains? (set black-notes) note)
        (q/fill (q/color 0))
        (q/text-align :right)
        (q/text-size 24)
        (q/text "#" (- (x pos) 7) (+ (y (positions note)) 9))))
    (doseq [line-note [:E3 :G3 :B3 :D4 :F4]]
      (line line-note))))

(q/defsketch sketch
  :title "Notes"
  :setup setup
  :draw draw
  :size [320 140])
{% endhighlight %}

## core.logic

Now, to examine what makes a tonal melody, we are first going to define the concept of _semitones_. A semitone is two notes adjacent on a piano, which can be defined in core.logic like this.

{% highlight clj %}
(l/defrel semitone note-1 note-2)
(l/facts
 semitone
 (partition
  2 1 [:C3 :C#3 :D3 :D#3 :E3 :F3 :F#3 :G3 :G#3 :A3 :A#3 :B3
       :C4 :C#4 :D4 :D#4 :E4 :F4 :F#4 :G4 :G#4 :A4 :A#4 :B4
       :C5]))
{% endhighlight %}

Tones are defined as two semitone, so `:C3` and `:D3` are a tone. We ca create a core.logic function for that. We also need tones and a half.

{% highlight clj %}
(defn tone [note-1 note-2]
  (l/fresh [middle-note]
           (semitone note-1 middle-note)
           (semitone middle-note note-2)))

(defn tone-and-a-half [note-1 note-2]
  (l/fresh [middle-tone]
           (tone note-1 middle-tone)
           (semitone middle-tone note-2)))
{% endhighlight %}

The next concept is scales. Scales are adjacent notes that have a pattern of semitone, tone and tone-and-a-half between them (I once again apologise for my poor grasp of music). Here are some frequently used scales in our model.

{% highlight clj %}
(def major-scale
  [tone tone semitone tone tone tone semitone])
(def harmonic-minor-scale
  [tone semitone tone tone semitone tone-and-a-half semitone])
(def natural-minor-scale
  [tone semitone tone tone semitone tone tone])
(def locrian-mode
  [semitone tone tone semitone tone tone tone])
(def mixolydian-mode
  [tone tone semitone tone tone semitone tone])
{% endhighlight %}

To check if a sequence of notes are a scale, we define a function for checking this. We only operate with scales of length eight in this demonstration. The function takes a scale and defines a core.logic function that takes a sequence of notes and dictate they conform to the scale.

{% highlight clj %}
(defn scaleo-fn [[s1 s2 s3 s4 s5 s6 s7]]
  (fn [m1 m2 m3 m4 m5 m6 m7 m8]
    (l/all
     (s1 m1 m2)
     (s2 m2 m3)
     (s3 m3 m4)
     (s4 m4 m5)
     (s5 m5 m6)
     (s6 m6 m7)
     (s7 m7 m8))))
{% endhighlight %}

We can now generate scale using core.logic and play them in Overtone

{% highlight clj %}
(reset!
 m-atom
 (rand-nth
  (let [scaleo (scaleo-fn major-scale)]
  (l/run 10 [scale]
         (l/fresh [s1 s2 s3 s4 s5 s6 s7 s8]
                  (l/== scale [s1 s2 s3 s4 s5 s6 s7 s8])
                  (scaleo s1 s2 s3 s4 s5 s6 s7 s8))))))
{% endhighlight %}

Now, a tonal melody can be generated by using our scales and some other rules. In the example below, I dictate that a melody must

* The scale must be in major
* The melody must be a permutation of the scale
* The melody must start and end on the same note as the scale
* The melody must have a perfect cadence

The example asks core.logic to generate 512 melodies that conform to these demands and plays a random one of those. If you run the code several times, you can probably find a melody you find pleasing. They are printed to standard out by Overtone every time they are played, so you can save them for later. And please play around with the different scales, cadences etc.. It's great fun!

{% highlight clj %}
(reset!
 m-atom
 (rand-nth
  (let [
        scaleo (scaleo-fn major-scale)
        ;;scaleo (scaleo-fn harmonic-minor-scale)
        ;;scaleo (scaleo-fn natural-minor-scale)
        ;;scaleo (scaleo-fn mixolydian-mode)
        ]
    (l/run 512 [melody2]
           (l/fresh [melody
                     m1 m2 m3 m4 m5 m6 m7 m8
                     scale
                     s1 s2 s3 s4 s5 s6 s7 s8]
                    ;;(l/== s1 :D#4)
                    (l/== melody [m1 m2 m3 m4 m5 m6 m7 m8])
                    (l/== scale [s1 s2 s3 s4 s5 s6 s7 s8])
                    (scaleo s1 s2 s3 s4 s5 s6 s7 s8)
                    (l/permuteo scale melody)
                    (l/== m1 s1)
                    (l/== m8 s8)
                    (l/== m7 s5) ;; perfect cadence
                    ;;(l/== m7 s4) ;; plagal cadence
                    ;;(l/== m7 s2) ;; just nice cadence
                    (l/== melody2 [m1 m2 m3 m4 m5 m6 m7 m1]))))))
{% endhighlight %}

I didn't know any of these rules before playing around with them, but they make for very pleasing melodies!

## Conclusion

I think Clojure really showed what a powerful platform it is. In under 200 lines of code, I was able to implement a program that composes and plays music using logic programming.

The main thing that annoys me about my experiments is, that I need a `scaleo` generator function. It would be mush nicer to define `scaleo` as taking a melody and a scale. That would also allow me to feed a melody into core.logic, and having it telling me which scale the melody was using. I was not able to do so, as my attempts to define `scaleo` ended up in something that I couldn't figure out why didn't work. 

{% highlight clj %}
(l/defne scaleo [scale melody]
  ([() ()])
  ([[sh . st] [[mh1 mh2] . mt]]
     (sh mh1 mh2)
     (scaleo st mt)))
{% endhighlight %}

Feedback in the comments and pull requests [on the project](https://github.com/tgk/the-composing-schemer) are more than welcome!