---
layout: post
title: "Prototyping an RTS idea"
description: "A prototyping process that finally worked for me."
tags: [games]
---

Optional Background
===================

Pipe dream: I would love to make an original game. Back in 2014 I worked on a [Commander Keen 5 clone](https://github.com/PlanetLotus/keen5-linux) on many of my weekends for a little over a year. I loved having something to work on on the weekends and see the project get better month by month. Since then, I haven't had much of a programming side project. The main blocker for me is making an "original" game, but having made a clone already, I don't really feel the need to make another clone. The problem with an original game, though, is that there's a lot of work up front for designing it. I'm comfortable with development. Not so comfortable at game design. I'm not even sure if I like it. But, slowly, I'm dipping my toes in...

Prototyping!
============

I've had an idea for an RTS for a couple years now, but never tried prototyping it. I didn't really know how. I've read a lot about game prototyping over the last couple years but never gave it a shot. I even [wrote a post](https://planetlotus.github.io/2017/05/28/prototyping.html) about the conceptual challenges I have with prototyping.

Then it occurred to me. This is an RTS. I can use Warcraft III's world editor to easily prototype this game. Coincidentally, this world editor was my first introduction to programming when I was 15 or so. Because of that, it's already a familiar tool; no learning curve required. It's also perfectly suited to prototyping RTSs because Warcraft III itself is an RTS.

I'm still figuring out the process, and this is my first try, but here's what I've done so far:

1. Make a copy of an existing project I had from, like, 10 years ago
2. Identify the core gameplay mechanics that need tested for viability
3. Identify any mechanics that won't work well in the world editor and ignore them for now (this turned out to be the UI)
4. Start modifying the project to test these mechanics
5. Write reusable scripts to iterate quickly

Before giving this a shot, I would get stuck on figuring out what questions the prototype needed to answer. Now I realize that simply starting a prototype with the idea has led me to come up with better questions. Starting from a vague, subjective question like, "Would this be fun?" I ended up running into real problems with the mechanics that made me rethink how it should work. It answered some of my questions about how things should work based on practical results, instead of just guessing what might be fun. I still have a long way to go and more to learn, but at the moment I'm really excited with where this is going and I'm happy to have a fun side project again, if only for a little while.

For anyone with my same mentality before trying this, I would now suggest:

1. Find a prototyping tool / game engine that will let you test an idea quickly. Don't worry about the fact that you'll have to rewrite it if you end up wanting to build the game
2. Figure out what the core mechanic is and build it in your prototyping tool. Don't worry about what questions it needs to answer yet. You'll come up with those while building it
