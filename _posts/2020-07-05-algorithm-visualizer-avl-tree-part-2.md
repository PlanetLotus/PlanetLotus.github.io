---
layout: post
title: "Algorithm Visualizer - AVL Tree - Part 2"
description: "A demo showing basic insertion & rotation in an AVL Tree."
tags: [algorithm visualizer]
---

# A basic demo of AVL Trees

This is a follow-up to my [last post](https://planetlotus.github.io/2020/06/14/algorithm-visualizer-avl-tree-part-1.html). I finally got the basic drawing logic working. As noted in my last post, the main challenge is dynamically drawing an AVL tree as it's built, specifically how information is passed between the data structure and the drawing logic.

## Challenge 1 - Broken references

This is detailed in the last post too, but to reiterate, the first big problem I had was that the references between the nodes change after a rotation. Originally, the drawing happened after the data structure finished all its operations (at minimum, an insertion and the rotations it causes), so when I passed back references to the nodes to the drawing logic, some information was lost.

I solved this by changing the data structure implementation to use coroutines. This allowed me to interrupt the operations whenever I wanted. It worked really well.

## Challenge 2 - Wrong root node

Unfortunately, the new coroutine was also really complicated. My AVL tree algorithm no longer looked like an AVL tree algorithm. This led to bugs, the biggest of which was the root node getting incorrectly assigned. I never figured this one out. I decided to get rid of the coroutine.

## Challenge 3 - Broken references, again

So I nearly ended up back where I started, except with one less solution to try. I ended up trying another idea I had had originally. It sounded complicated in my head, but it ended up being much simpler than the coroutine. I have a model called `TreeStep` that the drawing logic reads from the data structure after each operation. In that model, I was originally keeping track of references to nodes (which led to the broken reference problem). Instead, I started just keeping track of an exhaustive list of values that needed updating. Since AVL trees don't have duplicate nodes, I could take advantage of that fact. My AVL tree and my drawing layer could use the value of the node to key off of, without leaking any other information. For each new `TreeStep`, I built a list of affected nodes and passed that back to the drawing logic. A snippet of this in action is below.

{% highlight csharp %}
public TreeStep(TreeNode rotated, TreeNode parent, TreeStepType type)
{
    this.Node = rotated;
    this.ParentValue = parent?.Value;
    this.Type = type;

    ValuesToMoveDown = new List<int>();
    ValuesToMoveUp = new List<int>();

    // Build values here instead of passing references to Nodes
    // The references get modified before they are read from, which breaks the drawing logic
    if (type == TreeStepType.RotateRight || type == TreeStepType.IntermediateRotateRight)
    {
        BuildValueList(rotated.Right, ValuesToMoveDown);
        BuildValueList(parent.Left, ValuesToMoveUp);
    }
    else
    {
        BuildValueList(rotated.Left, ValuesToMoveDown);
        BuildValueList(parent.Right, ValuesToMoveUp);
    }
}

...

private void BuildValueList(TreeNode node, List<int> values)
{
    if (node != null)
    {
        values.Add(node.Value);
        BuildValueList(node.Left, values);
        BuildValueList(node.Right, values);
    }
}
{% endhighlight %}

This ended up working perfectly, and along the way, I cleaned up a lot of awkward code.

## Next steps

Here are my next priorities:

* Fix the spacing adjustments so that it doesn't push everything to the right at the end (you'll see what I mean in the video)
* Make it easier to follow what's happening in the rotations (for example, move multiple nodes simultaneously in the same rotation)
* Figure out what to do with the camera so it can fit growing trees (for example, auto-zoom and based on the size of the tree)

Beyond that, I hope to remove all references between nodes in the drawing logic, so that it only depends on the value being drawn, not any relationships between the nodes. I think this will simplify things since the reference updates are the trickiest part to keep track of.

## Demo

Here's a demo showing two trees, one with 3 nodes and one with 11.

<iframe width="720" height="480" src="https://www.youtube.com/embed/AXq2ZJCxIus" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
