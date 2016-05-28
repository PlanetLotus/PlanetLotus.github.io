---
layout: post
title:  "Making the Physics Framerate-Independent"
description: "Refactoring the physics engine to work the same regardless of the preferred framerate."
---
Since my last post I've added the concept of an item to the game and so now
collision with items works, but I admit a bit of laziness on my part has
resulted in no new video. I also want to hold off on a video for now because
there are lots of bugs I still need to work out...many parts of the gameplay
look a bit ridiculous even though it does actually look like a game now.

For the last couple weeks I decided to do a fairly significant refactor of my
physics logic. I made some really bad design decisions when I was writing that
logic and I wanted to fix that permanently. At the time, I was mostly concerned
with getting it working (that was hard enough!), and didn't put too much
thought into doing it the right way, especially since C++ was pretty new to me
at the time too. I now have a somewhat better handle on things and could put
more thought into the design.

The main feature of this
[refactor](https://github.com/PlanetLotus/keen5-linux/pull/11/files) is that I
made all velocities framerate-independent. I also made these velocity/speed
values easier to modify by stuffing them all into header files rather than
hardcoded numbers scattered throughout the various files.

### Framerate Independence

I've wanted to do this for quite awhile but my recent exposure to Unity3D made
me think about it more. I wanted to do something like what Unity does in its
update methods. There are several intricacies of Unity I don't understand yet,
but the main idea is there are a few methods that Unity calls every so
often, some more than others. When you want to do programmatic movement
of something, you have some value representing the magnitude of
movement, based on some unit like meters or pixels. You then multiply by
the amount of time that has elapsed since the last frame. You get this
elapsed time via Unity's built-in
[Time.deltaTime](http://docs.unity3d.com/ScriptReference/Time-deltaTime.html)
value.

Up until now my movement was based in pixels per frame. For example, for every
game loop, my character could move 5 pixels in the left or right direction when
walking. This is really brittle because if you change the framerate of the game
(which I'm likely to do later), then you also need to adjust all of your speed
values. Making this framerate-independent meant making a speed value based on
pixels per second, rather than pixels per frame.

To do this, I attempted to do what Unity does by keeping track of a `float`
that would tell me how much time has elapsed since the last frame. I then
multiply this by the velocity of each unit and that gives me essentially what I
had before, but this time it's not tied to the framerate.

### Problems

I ran into some pretty annoying problems that took me awhile to pin down. At
first I just took the final calculated velocity value for each unit and
multiplied by the time delta to get the final movement. In many cases, though,
I don't want this to happen. For example, when the player character
is about to collide with the ground, my game logic sets the
character's velocity (the distance he's about to move) to the
distance between him and the ground. This value constantly changes
because it depends on how close to the ground the character is when
a collision is detected. I don't want this to be multiplied by the
time delta because then it'll take multiple frames to reach the
ground, creating some very jittery movement.

The solution to this was to simply negate the time delta calculation (by
dividing by the time delta) in several spots. This sort of left a bad
taste in my mouth because I originally wanted to multiply by the timedelta in
one spot and have that just work. Now, though, I'm dividing by the time delta
in several spots, which not only adds a lot of messiness to the code, but it
also defeats the point of making velocities dependent on the passage of time.

### Next Steps

I need to figure out how best to handle these edge cases and not negate the
calculation all over the place. I may accomplish this by only multiplying by
the time delta in certain spots, rather than at the end of the velocity
calculations. That will only help if the exception is more common than the rule
though, which I have not determined yet.
