---
layout: post
title: "Algorithm Visualizer - AVL Tree - Part 1"
description: "The challenges of programmatically drawing a self-balancing binary search tree."
tags: [projects]
---

# AVL Tree in Unity

This is the third post for this project and it's finally getting where I had envisioned it. The original idea for this project was to animate binary search tree operations, but I started simpler, and I'm glad I did. This is a significant step up in difficulty, not in terms of implementing the AVL tree algorithm, but in drawing/animating it.

I've tried (maybe foolishly) to keep things loosely coupled, keeping the AVL tree implementation logic separate from the drawing logic. This has proven to be the primary source of frustration and difficulty. In [previous algorithms](https://planetlotus.github.io/2020/05/17/algorithm-visualizer-quicksort.html), these two layers operated separately:

1. Run the sorting algorithm on a list of numbers
2. The sorting algorithm returns a list of "steps" taken to get to the final state
3. Re-play those steps in another layer to draw out what happened

I started by doing this same thing for the AVL tree, but I've discovered this is more trouble than it's worth. The AVL tree's data structure consists of a root node which then has pointers to child nodes. As nodes are added, these references change due to the self-balancing nature of the tree. Therefore, a "final snapshot" of steps returned to the drawing layer actually misses information, because the references have often changed multiple times.

I managed to get this working for inserts and simple (single) rotations, but it completely breaks for double rotation scenarios. At this point, I realized I needed to try something else. So for now, I have a half-working implementation that I'm going to partially rework. The next idea is to use coroutines in Unity to have the algorithm layer return one step at a time, "interrupting" it in the middle of its operations in order to draw the current step, before handing back control to the algorithm.

No demo this time, since it's in a half-working state. This post is just to keep myself accountable to finish the project!
