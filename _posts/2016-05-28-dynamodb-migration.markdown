---
layout: post
title:  "Performing a DynamoDB Migration"
description: "The why and how of changing the format of our DynamoDB document with a no-downtime approach."
tags: [work]
---

### First, Some Background

At the company I work for, my "area of ownership" is a chain of several backend
programs that are designed to scale horizontally (more machines, not better
hardware). A couple months ago I recommended we switch from using Azure
DocumentDB as our primary data store to Amazon DynamoDB. The "why" of that
could easily be another post, so for now we'll stick with the fact that we've
only recently started using DynamoDB and are still figuring out what its
strengths and weaknesses are, and how well our model fits with it.

I soon found myself wanting to use a GSI (Global Secondary Index) to perform a
quick lookup of a large set of data (the "how" and "why" of which should also
be another post) and instantly ran into a limitation of DynamoDB: You
can't use a nested property as the partition key of an index. It has to be a
root-level property. That means it's time to change the document format (or
what I'd call a "migration") and update existing documents along the
way.

### Now, the Challenges...

My company is still young enough that our database isn't huge yet. Our DynamoDB
table currently has just over 1.5 million items, which is big enough to make me
really think about the best way to migrate data, but not so large that it's
super expensive. Our data by its very nature will grow over time, and I expect
we will have over 10 million items, maybe 50 million, within the next year or
two. I would consider us "at scale" at 100 million, at which point it'll still
keep growing, but at a slower rate (I apologize for not being super clear about
what we do at this point. I want to respect our company's privacy until I get
the OK to make more details public).

Because we're planning for large growth, I wanted to do this the right way. At
this stage, it's not a big deal if this particular system has a day of downtime
(it's not user-facing) but at scale we wouldn't want any downtime. With that in
mind, I took the extra time to make this happen with no downtime and to devise
a reusable strategy for future migrations. Why would I want this to be easy?!

"No downtime" in our system means that the migration will be running while
another process is modifying our documents, so that means the code running in
production must be aware of both the old and new document formats. This process
runs once an hour and involves reading existing items, updating them, and
adding new items.

While I was at it, I reassessed our document format. There were many things
wrong with the format in addition to needing to move one property to the root
level. I ended up fixing every problem I could think of so that I only had to
migrate once (at least, until we discover more problems). This ended up being
much more complicated than moving a field because several data types changed,
requiring properties to be remapped.

### The Plan

With the help of a coworker, we broke this down into these simple-ish steps:

1. Create a new DynamoDB table for the new format
2. Make the production process do reads from the new table first, then if the item isn't there yet, read from the old table
3. Make the production process write only to the new table
4. Make the migrator scan the old table, writing to the new table but only if that document doesn't already exist in the new table

![]({{ site.github.url }}/images/dynamodb-migration/steps.png "Now you see why I didn't go into graphic design. Next time I'll at least use something other than Paint.")

The important thing to note here is that once the migration starts, the old
table will never be written to again. It'll only be read from if the document
in question doesn't already exist in the new table. Even if the migration fails
midway through and has to be restarted, it will only write each document once.

### Running the Migration

Fast-forward several hours of code changes and testing, and I was ready to run
the migration. I wrote a quick script to scan the old table, converting the
documents to the new format and uploading to the new table if it wasn't already
there.

I wasn't sure what Read Capacity Unit (RCU) and Write Capacity Unit (WCU)
values to use. Honestly, I guessed, knowing I could increase the value as many
times per day as I wanted. For our 1.5 million items totaling 5.1 GB, I picked
100 RCU on the old table and 50 RCU, 400 WCU on the new table. I added a good
amount of logging to the script, threw it on an EC2 instance, then let it rip.

I quickly confirmed that new documents being added looked the way I wanted to.
The hours of testing paid off. This post ignores how much time was spent in C#
creating a new format for an otherwise simple goal of moving a property to the
root of a document, so while that's where most of the effort went, it certainly
wasn't the interesting part of the migration.

I was impressed with the performance. With the resources I threw at it, I
figured it would take about 6 hours to finish, which would put it at around
10pm that day. Great idea, by the way, starting a lengthy process at 4pm...

The RCU and WCU values I chose seemed to be reasonable. I actually
over-provisioned pretty much everywhere. Reads from the old table never went
above 40, and I had it set at 100. Reads from the new table never went above 15
or so and I had it at 50 (this was likely to burst though, so it was good to
over-provision). Writes to the new table never went above 335 or so,
and I had that set to 400.

One thing I'm still unsure of is how I'd squeeze more performance out of
DynamoDB if I wanted it to go faster. Would I need to run my code on more
machines? Would my reads/writes need to be more distributed, rather than
sequential? This seems like a very low limit, so I'm sure there's plenty more I
could do to make it go faster. If you know, please leave a comment down below.

### Performance Degradation

I kept an eye on the logging throughout the night. I started to notice around
9pm that things had started slowing down significantly, to the point where I
was going to lose sleep making sure it finished. I spent about an hour looking
at AWS's wonderful metrics trying to figure out why throughput suddenly tanked
to about 40% of where it was.

![]({{ site.github.url }}/images/dynamodb-migration/old-table-reads.png "Reads on the old table over time")

![]({{ site.github.url }}/images/dynamodb-migration/writes.png "Writes on the new table over time")

I'm still not totally sure what went wrong, but I think it had something to do
with my scanning of the old table. At some point, the scanning wasn't returning
results for every page, so my script had to keep querying DynamoDB trying to
get more data, knowing that it wasn't at the end of the table yet. I'm not sure
what causes this. I tried bumping up the RCU and WCU values to no effect. Sadly
no "lesson learned" here; I'll just have to keep an eye on scanning behavior as
the table grows. If you know what I may have missed, please let me know with a
comment below.

![Table scan showing decreased result count over time]({{ site.github.url }}/images/dynamodb-migration/scan-results.png "Scan of the old table")

### Conclusion

To my surprise, it's been several days and I still haven't found any bugs in
the migration. The morning after the migration, I wrote a quick script to grab
all of the IDs out of both the old and new tables, making sure no IDs were in
the old table that weren't in the new table. It's nice to use a system that
just works.
