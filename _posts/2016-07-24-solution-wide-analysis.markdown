---
layout: post
title: "How ReSharper's Solution-Wide Analysis Exposes Bugs"
description: "My favorite 9-5 discovery of the last few months."
---

### ReSharper's Solution-Wide Analysis

I've used ReSharper at work for a few years now and I only just recently
discovered the [Solution-Wide
Analysis](https://www.jetbrains.com/help/resharper/2016.1/Code_Analysis__Solution-Wide_Analysis.html)
tool.

Usually you can find me turning off as many features as possible so that I'm
not slowing down the tooling, but I've found it very helpful to at least
temporarily turn this tool on. Thankfully it's easy to toggle.

Since you can see its purpose at the link above, I'll just quickly go over my
main use case.

### Identifying Unused Public Properties

At work, I have classes with hundreds of public properties that are all mapped
to another class's properties. In order to verify that I haven't forgotten to
map any, I can simply enable Solution-Wide Analysis and it'll tell me if any of
the public properties are unused. This helped me identify some properties I
added and didn't need, others that should've been mapped and weren't, and
others that were incorrectly mapped! Three types of bugs solved immediately.

Without this tool, I would've had to Find Usages on each property manually.
This might be fine once, but as code changes and I need to check again, it
doesn't really work. At first, it doesn't sound too amazing to be able to see
unused public properties (it just sounds like cleanup), but in my case it
actually identifies bugs.
