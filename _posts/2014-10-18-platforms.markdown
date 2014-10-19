---
layout: post
title:  "Moving Platforms"
---
Moving platforms have been the hardest single feature to implement so far. It
was also much harder than I expected it to be. I think some other things were
harder, but not single features. Getting several features to not mess with each
other has been difficult in the past, but this was just insane. There's some
information about moving platforms for 2D games online, but I don't feel like
they capture what's actually difficult about them. Maybe their engine is
different than mine is and that's why this was so hard. At any rate, I'll
detail as best I can what this had in store for me.

It's worth noting I did some reading on the issue before attempting this.
There's a [great
article](http://higherorderfun.com/blog/2012/05/20/the-guide-to-implementing-2d-platformers/)
about 2D platformer engines that described this particular problem very well,
but again, it didn't capture what I struggled with.

### The Problem

"Moving platforms" refers to a statically sized object that moves, and that the
player can stand on. It's similar to a static tile that has top collision (e.g.
the "floor" of a level). The difference is it moves, and it carries the
player with it.

### The Initial Approach

This doesn't sound too hard. As usual, I expected things to be harder than it
sounded, but only so much. I broke this problem into a few pieces:

1) Treat the platform like the floor. No movement, and get collision working
2) Add movement to the platform
3) Move the player with the platform

In these distinct pieces, this sounds simple enough. After all, the only thing
I hadn't done before was the third step, and that should be as simple as
incrementing the player's position by the same value as the colliding
platform's velocity in each direction. That's trivial!

### It Wasn't Trivial

Actually, all of that was trivial. The hard part was the collision detection. I
quickly found out there are two main problems here, and their solutions like to
combat each other:

- The "initial" collision detection. For example, the player is falling and the platform is moving up. Detect when they collide.
- The continuous position updates based on the colliding platform.

The initial collision is, in theory, exactly the same as any other collision
detection. The difference here is I need to store a handle to the platform the
player is colliding with. This was easy enough.

The continuous position updates refer to checking if there's currently a
colliding platform. If so, increment position by the velocities of the
platform. This is also easy.

### The Actual Problems

Communication across classes. You could easily chalk this up to poor design and
I'd agree with you; there are probably better class designs for my engine that
would make this easy. At any rate, the way I have it, passing around
information between the platform and the player at the right time was very
challenging.

My first implementation had a problem where the player would fall through the
platform because the platform updated its position before the player's position
is updated, resulting in the player being "below" the platform by the time its
update method was run. I "fixed" this by checking against the platform's old
position instead of the new one. That's logical enough, though a little sloppy.
This created a new problem! The player could get himself stuck on the platform
if he landed on it at the right height. I don't think I ever found the exact
cause of this before scrapping it and trying something else.

I knew I had a catch 22 on my hands. The platform needs to update before the
player, but the player needs a chance to update too so that he's not under the
platform during his update. What's worse, I kept confusing myself with whether
I should be checking the platform's old position vs. new position, and the
player's current position vs. his desired position based on user input
(jumping, falling, etc.).

With this in mind, I rewrote my approach. I tried to only interact with the
platform's new position, and the player's current position. This helped a lot
because instead of checking potentially 4 states, I was only checking 2. With
the same implementation though, it didn't solve my problem. I messed around
with debugging this for hours before taking a few days off.

### What Actually Worked?

I started over. This was my new thought process:

1) Check if the player is standing on the platform (that is, top of platform ==
bottom of player) *before* the platform's position is updated. If so, store
the platform's handle.
2) Update the platform.
3) As for the player, check if there's a platform handle. If so, shift the
character by the platform's velocities as before. If not, check for an initial
collision.

This was actually very similar to my old implementation except I put more logic
in the platform code. I didn't like this because I feel like it should all be
handled in one spot, and in theory it could be, but this was the easiest way to
wrap my head around it this time.

An hour or so later and I was exactly where I was with the last implementation.
Same problem of getting "stuck" on the platform.

I finally figured out that this was due to the handle never getting set.
Whether this was the problem in the first implementation, I have no clue. At
any rate, a small bug fix later and it worked a lot better. I think the new
implementation made debugging a lot easier.

I faced a few smaller bugs after that, such as:

- Player can't jump while on platform
- Player jumps for one frame then gets pulled down to the platform
- Uneven, bouncy ride on the platform
- Jarring first collision

With a new understanding of what's going on, and shorter, simpler code, these
were much easier to figure out and I figured them all out within a couple
hours.

### Conclusion

The hardest part about this problem was knowing what logic to put in which
class, the Player or the Platform, and whether to look at old positions or
new/desired positions. There are many possibilities...per update, you
potentially have the old platform's location, the new location, the player's
current location, and the player's desired location. Combining all of these
makes for a very confusing experience and this is not obvious before you start
working on it.

<iframe width="420" height="315" src="//www.youtube.com/embed/0dpIANDihlI" frameborder="0" allowfullscreen></iframe>
