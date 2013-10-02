---
layout: post
title: Torus pong - a massive multiplayer online game built in a weekend using Clojure and ClojureScript.
---

# {{page.title}}

*Update* Ragnar has written up a blog post describing the architecture
 of our solution. If you're interested in that part of the project,
 please find it
 [here](http://ragnard.github.io/2013/10/01/clojurecup-pong-async.html).
 It goes into detail on how core.async and WebSockets and how we used
 them to coordinate state between clients. It's a great read, and very
 much worth your time.

Last weekend saw the first ClojureCup. Here at uSwitch, we teamed up to
build a massive multiplayer version of the arcade classic pong using
Clojure and ClojureScript. This blog post outlines how we went about it.

If you want to try out the game, you can find it at
[pong.clojurecup.com](http://pong.clojurecup.com). Until Friday October
4 2013, you can vote for our entry
[here](http://clojurecup.com/app.html?app=pong).  If you'd like to read
the code or try running the project on your own machine,
[the code is available at github](http://github.com/uswitch/torus-pong).

## Journal

We did quite a lot of planning ahead of the competition. Having built
and maintaining quite a lot of Clojure applications at our day-job gave
us a pretty good idea what we wanted the server architecture to be like,
and how we wanted to deploy our app. Apart from that we spent a couple
of sessions discussing how the game engine should work, the general
objective of the game, and how communication should be handled across
the server and client.

The work we did over the two days was distributed like this:

- Saturday

  We didn't start Friday night, but started in the morning on Saturday,
  giving us plenty of rest before beginning.

  - Setting up the server with
    [DigitalOcean](https://www.digitalocean.com), installing nginx,
    java, lein and ruby for deployment. Creating a deploy user and
    copying in public keys for all participants, such that everyone
    could deploy. I did this before leaving my house - we already agreed
    the setup, so this was mainly grunt work.

  - Setting up a basic skeleton for the project. This included building
    a basic jetty app, extending it with WebSocket support, setting up a
    ClojureScript code base with compilation, and building a project
    that it was easy to restart locally on our development
    machines. Ragnar did all of this before leaving his house in
    Cambridge to meet up with us down in London.

  - Doing basic setup. After Jon, Ragnar and I met (and after having
    created our war room at work), we did some more basic setup:
    configuring nginx to support WebSockets, configure Capistrano to do
    automatic deployment. Set up upstart scripts, etc.... All the small
    details.

  - Build the first game engine and hook it up to jetty. We'd already
    designed these parts, but they still had to be implemented. We had a
    pretty good idea about what the communication protocol was going to
    be between frontend and backend, so this was pretty much being done
    with very few discussions.

  - Implement the first drawing methods on the frontend. We originally
    had a different plan for how the game board should look, and a lot
    of time went into implementing the first proposal. Again, having the
    communication protocol agreed upon beforehand gave us the
    opportunity to do this work in parallel.

  That was basically it for the first day. We had basic communication
  set up, we were able to move the paddles around on the screen (but
  balls were a mess) and we had a basic physics engine. After that,
  Ragnar went back up to Cambridge, and Jon and I went to the pub. No
  pulling all-nighters for us :)

- Sunday

  On the Sunday, Ragnar stayed up north, remoting on the project. Jon
  and I met up at the office around mid-day to finish up our project.

  - Before meeting up with Jon, I had a go at implementing a different
    view of the torus game board. In the original implementation, you
    could only see yourself, and the pads immidiatly next to you. In the
    second implementation, you can see all players at once, aligned in a
    donut-shaped gameboard. This also fixed the ball drawing, so we
    scrapped the previous visualisation and went for this one.

  - Jon added some helpful text and styled it with a retro font. It
    really made the thing look like a variation of one of the classic
    games.

  - Ragnar implemented multiple games. Up till now we only had one
    massive game going on. With the change, new games are spawned when
    the number of participants in a game reaches eight.

  - We had quite a lot of problems with the original physics engine. I
    had several goes at fixing it, but didn't attain insigth until later
    in the evening. A lot of time went into trying different approaches.

  - Ragnar added monitoring, both using google analytics and
    collectd. Knowing the health of your instance is extremely nice when
    deploying or performing experiments. collectd data was sent to
    [librato metrics](https://metrics.librato.com/metrics).

  - Ball collisions. Adds a bit of chaos to the game.

  - Scoring and rules. Up till now there was no objective to the
    game. We added a score that would increment every time your pad was
    hit. Whoever reaches 20 points first wins, and the game is reset
    (with the same players).

  - Sounds! Jon added sounds for collisions and for when players
    won. Again, much more retro!

  - We added player names, taken from a list of gamer handles Jon dug
    out. He also pruned away the most offensive ones from the list. When
    you connect to the game, you are automatically assigned a random
    name which is shown above your pad.

  - We added two buttons on the page, for mobile support, when people
    can't use up and down keys (or `w` and `s`). The buttons also work
    when you hover over them with your mouse.

  - Quite a lot of maintaineance is needed, even on such a small
    project. Having a clean codebase allowed us to work incredibly
    fast. It did mean being slightly frustrated by not making any
    progress from time, but that's how it is.

  Jon and I left uSwitch around six or seven, Ragnar cracked on a bit
  after that. I finally got around to fixing the physics engine in the
  evening, and Ragnar fixed some font issues. Again, we didn't work
  until late in the evening, but terminated development a couple of
  hours before the deadline. You really don't want to introduce a
  feature with a bug in it 15 minutes before a deadline.

## Status

Doing the project was great fun! Being forced to do all the planning up
front is a always a good exercise. It forces you to go into hammock
mode and think before typing.

Working with people on something completely different form what we
normally do at work was also incredibly gratifying. You get to see some
completely different technologies and techniques being used.

## Advice

It's the first time I've participated in a hackathon. The following is a
mixture of advice we heard before the competition and our own
experiences. It all seems to be valid.

- Be open and honest about your ambitions for the project up front.
- Get plenty of rest and go for walks. You need a clear mind to not fall
  down into a bug-hunt hole.
- Discuss and design up front. Get those whiteboards out. Start with the
  big picture, and dig down into components where you aren't entirely
  sure everyone has the same understanding. This makes it possible to
  divide work on the day.
- Ask questions - accept questions. There's no time for
  misunderstandings.
- Deploy often. You want to know if your commit broke the
  project. No-one depends on our system, it isn't critical. Do not be
  afraid of breaking it.
- Accept change. Every plan fails once it hits the battlefield, but with
  no plan at all, you are going to spend a lot of time discussing on the
  day.
- Don't start major work in the last six hours. Gold-plate instead.

## Backlog

There were (of course) some things we didn't get around to get
implemented. Here are some of the things we had on our backlog by the
end.

- Running the game engine on the clients as well as one the server. This
  way, the server would only have to transmit data about other player
  movements and when they hit balls, not ball positions at all
  time. Doing predictive rendering is hard. Ragnar found a good article
  called
  [Source Multiplayer Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking),
  but we  didn't fint the time to implement it.
- Share games between your friends, instead of just being thrown into a
  game.
- Selecting your own handle.
- Different game rules.
- Special balls that gives you a penalty or alters the game in other ways.
- A more realistic physics engine that doesn't rely on ticks.
- AIs in games with few players.
- Kick idle players.

Even without the things from the above list, the game is quite
playable. If you want to give it a go, you can find it at
[pong.clojurecup.com](http://pong.clojurecup.com). Remember to vote for
us [here](http://clojurecup.com/app.html?app=pong). The source is
available [at github](http://github.com/uswitch/torus-pong).
