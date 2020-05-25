---
layout: post
title: "MongoDB - When to shard"
description: "A quick rule of thumb to determine whether your database is large enough that it should be sharded."
tags: [mongodb]
---

# 2 TB

2 TB. If your database is smaller than this, really consider whether sharding is necessary. Of course, size isn't the only factor, but this is a good rule of thumb.

Disclaimer: I run a ~200 GB MongoDB cluster and *have not* sharded, so I haven't seen both sides of this.

I didn't come up with the number. This number was provided by a consultant from MongoDB. Internally at my company, we were debating whether sharding was necessary at our size. I was really hoping it wasn't, because I knew it would make things a lot more complicated. Over the last several years, there's been a recent trend of developers thinking their data is bigger than it is, and it seems like this is commonly leading to premature large-scale design patterns (I'm not immune to this either).

The database I run will never hit 2 TB. We trim it so that it only keeps the last 2 years of data, and that makes its size fairly predictable. With that in mind, I'm hoping to never need sharding!

Worth noting: this doesn't apply to all databases, just MongoDB. For Elasticsearch, for example, [recommendations](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster) I've seen say that you should have a shard every 10-50 GB or so.

I have a feeling that hearing this advice saved me a lot of pain. I'm hoping that sharing it more broadly will save others pain as well.

Have you sharded your MongoDB cluster? How big is the cluster? What was your experience? Let me know in the comments below.
