---
layout: post
title:  "Deadly Tiles!"
description: "How I implemented a stationary enemy in Commander Keen 5."
tags: [keen]
---
Next up is the FireSpinner. The FireSpinner is a stationary object with a
single repeating animation that kills Keen upon contact. That's it. Pretty
simple.

### Tile or Enemy?

How should this fit into my class hierarchy? I have two sort of class trees
this could fit into. One is the Sprite tree. Sprite is the base class, and from
there it goes to MovingSprite, then Enemy. Enemy has all of the properties of a
sprite that can move and collide with other objects, as well as the ability to
specifically detect collision with the player.

The other logical choice is the Tile base class. A Tile is basically a 32x32
block of pixels with an image and various properties such as collision.

I decided that a FireSpinner is more closely related to a Tile than an Enemy.
Tile has no movement and no agenda of its own. Neither does a FireSpinner. The
only things it lacks are animations and the ability to harm Keen, which is what
Enemy has.

This fit into my code pretty easily. In my level editor, I added an "IsDeadly"
boolean property to tiles. In my game code, for now I assume that if isDeadly
== true, then it's a FireSpinner. This can get more intelligent later without a
ton of modification, such as a "DeadlyTile" class. For now all I have is the
FireSpinner and I've learned not to overcomplicate class structure prematurely.

After that, it was a pretty routine process of adding animation frames. Giving
it the ability to harm Keen on contact was easier than expected too. Other
Enemy objects keep a reference to Keen and check when they come in contact with
him. This is a bit sloppy. The FireSpinner, on the other hand, is detected by
the Player class when Keen collides with it. I can then check if isDeadly ==
true, and if so, kill the player.

I later realized this isn't totally true to the real game because FireSpinner's
don't have collision, and the way the Player detects the FireSpinner in my code
is reliant upon the fact that he collides with the FireSpinner. If I decide I
care about this later, I can write it more like how Enemy classes detect and
kill Keen.

After that, I once again had the distinct pleasure of editing the black
background of the FireSpinner frames to instead be red so that they would
appear as transparent in my game. Thankfully with GIMP's clever color threshold
this was a lot easier than normal (Typically tiles have a black border which
makes this impossible. The FireSpinner does not).

<iframe width="560" height="315" src="https://www.youtube.com/embed/0mUlXPaq5cM" frameborder="0" allowfullscreen></iframe>
