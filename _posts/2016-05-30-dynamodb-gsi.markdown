---
layout: post
title: "DynamoDB Global Secondary Index"
description: "Using DynamoDB to get a quick lookup of all of the partition keys."
---

### The Problem

We regularly need a reasonably accurate list of all of our partition keys for a
couple purposes. One, we compare them against another database that we don't
control and want to make sure we remove any that they remove. Two, we like to
run "integrity checks" to make sure we have everything we think we have, and
for reporting purposes.

In our case, we want to fetch items by `FooGroupId1`. Right now we have roughly
1.5 million items and 8 unique values for `FooGroupId1`. That's an average of
187,500 items per value. In a SQL database this really isn't a problem, but we
recognize that fetching large sets of rows (especially sequentially) isn't one
of DynamoDB's strengths. So, we looked into GSIs...

### Global Secondary Indexes

Indices? Ugh, grammar and technology don't get along. AWS's documentation says
"Indexes" so I'll go with that!

There's a reason this is my second post about DynamoDB. A GSI's partition key
can only be on a root-level property, so that's something to watch out for as
needs change. After [migrating]({% post_url 2016-05-28-dynamodb-migration %}) our
document to put `FooGroupId1` on the root level, we were ready to create a GSI.

We care about three properties when doing this kind of query:

1. `FooPartitionKey`
2. `FooGroupId1`
3. `FooGroupId2`

What we're really after is an exhaustive list of `FooPartitionKey`, but having
the other properties is helpful too. We could just scan the table, but scanning
observes the entire document (in our case, an average of 3.5 KB or so) which is
expensive. Further, in our case, it's helpful to additionally partition by
`FooGroupId1` so we're not grabbing the whole table every time.

Creating the GSI is easy enough through AWS's interface.

### Gotcha

Yeah, so...it turns out, `FooPartitionKey` is automatically included in the
index. So when I first created the GSI, it had the following properties:
`FooPartitionKey`, `FooGroupId1`, `FooPartitionKey`, `FooGroupId2`. Seriously?!
I suppose this is the downside of a schema-less document.

It gets better. You can't cancel the creation of an index! I had the pleasure
of waiting for this to finish at 400 Write Capacity Units. It took about 90
minutes. Then, I got to delete the index and do the same thing over again
without manually including `FooPartitionKey`. Jeez, that was unintuitive.
Hopefully someone can avoid that mistake because of this post.

### Finally!

After nearly 3 hours of twiddling my thumbs, the correct index was finally
created. Now the fun begins: Measuring performance!

I'll spoil this early. I am pleasantly surprised with the performance of our
GSI. We're not at scale yet, but 1.5 million items is enough to notice
performance degradation without a GSI. Hopefully it'll continue to scale under
this model.

In production right now I have our GSI set to only 10 RCU and 7 WCU (this is
fewer WCU than our table, and the same RCU) and no throttling has
occurred. The reads from our GSI happen about 8 times per day at roughly the
same time, so I would qualify this as "burst read activity" (we will change
this so it's spread more evenly later). The writes happen in small
bursts every hour. Needless to say, I'm both surprised and impressed. I believe
and hope that it will scale. I expect the average number of items per
`FooGroupId1` to stay roughly the same, but our item count is expect to grow at
least into the tens of millions in the near future.
