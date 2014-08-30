---
layout: post
title:  "Next Steps: \"Little Ampton\""
date:   2014-08-30 16:35:48
---
I recently finished the AI for Sparky, the most basic enemy unit in the game. AI is of particular interest to me currently so I'm choosing the next most basic unit as my next step. This guy's name is "Little Ampton" (apparently there's some background to that name, but I'm not going into it). As with Sparky, I'm going to approach this problem with a state machine with the following states:

- Patrol
- Climb up pole
- Climb down pole
- Fix machine
- Stunned

Here are some notes of mine regarding each state:

### Patrol

- Move left/right on ground
- If colliding with Keen, push him
- If colliding with a pole, climb it (state = `CLIMB_UP`)
- If colliding with a machine, fix it (state = `FIX_MACHINE`)

### Climb up

- Kill Keen on contact
- At top of pole, exit to ground (state = `PATROL`)

### Climb down

- Kill Keen on contact
- Exit to ground when colliding with a tile with top-collision (state = `PATROL`)

### Fix machine

- Play animation for a few frames
- When done, patrol (state = `PATROL`)

### Stunned

- Occurs when shot once by Keen
- Play looping animation
