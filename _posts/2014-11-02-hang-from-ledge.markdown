---
layout: post
title:  "Hanging From Ledges"
description: "Hanging from ledges, a distinctive feature of Commander Keen 5."
---
I thought this would be one of the hardest features to implement and it didn't
end up being nearly that hard. It took a day to fully implement with a few left
over bugs. I made the mistake of not writing this right afterward, but instead
several days later, so it's not as fresh in my mind but I think I can still
explain the process.

### The Problem

This isn't a common platformer feature. Hanging from a ledge refers to one of
the player's top corners colliding with an edge tile's opposite top corner
while the player is holding the arrow key in the corresponding direction. When
that specific collision happens, the player is supposed to hang from the ledge,
then also have the ability to either drop down or roll onto the ledge
above. These are two distinct pieces of functionality for the feature.

### The Specifics

The first step was the collision detection. I broke this problem down into a few requirements:

- Must be falling to collide `yVelocity > 0`
- Must be holding corresponding key

[The
algorithm](https://github.com/PlanetLotus/keen5-linux/blob/master/src/Player.cpp#L504-L533)
was pretty straightforward because it's similar to what I've done
with other collision detection problems. This was actually even easier than
most because, based on the player's position, I could look for a tile at a
specific spot rather than a range of spots. This avoids looping through a bunch
of tiles.

{% highlight cpp %}
int tileRow = nextKeenTop / TILE_HEIGHT;
int tileCol = nextKeenRight / TILE_WIDTH;

Tile* tile = tilesRef[tileCol][tileRow];
if (tile == NULL || !tile->getIsEdge())
    return;
{% endhighlight %}

Later, though, I realized I did have to do some looping because checking Keen's
current position against the tile's position makes it easy for Keen to skip the
pixel-perfect collision since falling has Keen zooming by at multiple pixels
per frame. With that in mind, just like I've done with other collision
algorithms, I check for the top of a tile between where Keen currently is and
where he's about to fall to this frame. That covers all of the ground traveled.

{% highlight cpp %}
int yCollide = -1;

for (int i = keenTop; i <= nextKeenTop; i++) {
    if (i % TILE_HEIGHT == 0) {
        yCollide = i;
        break;
    }
}
{% endhighlight %}

In retrospect, I could probably optimize this by taking advantage of the fact
that I'm using mod, but for now this seems more intuitive.

After that was done I had to add some logic that allows Keen to fall from the
ledge as well as disable the ability to jump, pogo, or shoot.

Next up was rolling onto the ledge. This took awhile because I don't have a
good framework for timed animations, and that's what this is. Player input has
to be essentially disabled while the animation runs and then re-enabled after
it's done. Right now my logic for this is pretty messy because it's scattered
all over the place. This actually took more work determining whether my logic
was messed up or whether the frames on the sprite sheet weren't where they
should be, though. It ended up mostly being the frames (after all, my logic is
*never* incorrect!). Unfortunately there's not much interesting to show
regarding the [animation
algorithm](https://github.com/PlanetLotus/keen5-linux/blob/master/src/Player.cpp#L567-L597),
just that it ended up being harder than the rest of this feature.

<iframe width="560" height="315" src="//www.youtube.com/embed/l3bUzzVmzAU" frameborder="0" allowfullscreen></iframe>
