---
layout: post
title: "How to write custom data migrations"
description: "Data migrations are scary. Here's how to gain a better understanding of your data and how to migrate it safely."
tags: [work]
---

Migrating data is hard. Data migrations vary greatly, such as reformatting data within a database, moving data to a new database, or just changing how it's accessed in a non-breaking way.

Depending on the size of the data, you may decide it's worth it to take the data offline for the duration, but often this isn't an option. Your data has to migrate in a non-breaking way while remaining online.

The easiest approach to a database migration is something like a script in your ORM that runs on deploy. However, I've found that this straightforward approach is often lacking, and with just a little more time investment you can get more confidence in your migration and spend less time troubleshooting.

These are the generic goals I'd set for any online data migration:

* Self-healing, resilient to failure
* Resumable, doesn't have to start over after failure
* Transparent, easy to tell what's going on at all times
* Backward compatible, does not break production traffic, and its operations are idempotent

Your default migration script via your ORM might handle backward compatibility and idempotence but basically none of the others. By default, if your script fails in the middle of the migration, it'll have to start over, you may not know where it failed, you can't tell it to just do a subset of the migration while trying again, and it won't be able to restart itself.

## My recent data migration

Real-life examples are the most useful, so I'll describe the migration I just recently did, but keep in mind this could be used for other databases and in plenty of other scenarios. We moved one of our datastores from Azure BLOB into AWS S3. The most practical option was for us to write our own custom migration. The total data size is about 3 TB, not very big, but has roughly 200 million files, which I knew would take multiple days to migrate.

The most straightforward and obvious approach would be something like:

1. Loop over each record
2. Check if the record exists in the new datastore already
3. If not, copy it to the new datastore, otherwise skip it

but with 200M files, if the process fails, you really don't want to have to start over again; you want to be able to pick up where you left off. With a little more work up front, we can make this process robust. Here's the approach:

1. Split the migration into batches
2. Use the batch size to save progress mid-migration
3. Log what's happening per batch
4. Add error handling in case of failure
5. Test the migration by running a few batches individually
6. Spin up an internal web app with a simple UI to manage the migration

Let's go into detail on each of these.

### Migrate in small batches

This is the most important idea and all other benefits flow from here. When thinking about a batch size, I first think about the logging: how often do I want to be informed about its progress? I usually want a log every minute or two, otherwise I don't have quick feedback on what's going on. Anything more frequent than that and it's just noise.

Then, I need to know something about my data to pick a batch size from that, but honestly, I just guessed and it happened to be pretty close (experimenting with this is okay!). I ran this migration in batches of 20,000 files.

### Save progress mid-migration

This is related to the batch size. My approach here was somewhat custom; I loaded a list of primary keys from another datastore (this was faster to access sequentially) and grouped these into files. Each file had a list of primary keys and in my case, there were about 400 files (the grouping here is specific to how the data is organized but it's not an important detail for the approach in general). I saved these files to Azure BLOB so they could be accessed from the production machine the migration runs on.

The migration loops over these files one at a time. When it completes a batch, it removes those primary keys from the file and saves it, effectively marking them as done, then continues on to the next batch. That way, if the migration fails at any step, we don't lose progress; we just load up the files again and keep going.

### Log every batch

For a long-running migration, I want to know even when things are going well. Every time the migration finishes a batch, I log it with something like:
"{batchNumber}/{totalBatches} completed in {secondsElapsed} seconds"

### Error handling

The main thing here, again, is logging. Every exception gets logged. Exceptions I can't handle will cancel the migration, but transient network failures get handled with a simple retry loop.

### Test the migration

Another benefit to the migration being split into files/batches is that I can easily run just a single file or batch to test it. By testing with various file numbers (slightly different types of data in my case) I can validate my assumptions without having to run the whole thing. This also helps me catch common errors quickly.

### Internal web app

I use internal tooling a ton. I have an administrative site for this service, so it was trivial to add another controller to the site which calls my migration script. It took another 30min or so of copy/pasting UI patterns I use on other pages and suddenly I had a useful UI to manage the migration. Starting the migration takes a single click from this page.

## Wrapping it up

I recommend an approach like this whenever the migration could otherwise negatively impact customers, or even if it would just take multiple hours (or longer) to run. I've done this a handful of times with various datastores and I'm always surprised at how little extra time it takes to add these benefits in. Thankfully, data migrations like this aren't very common, but the downside is that they're hard to get practice with, so I hope this guide will give others some ideas about how to approach their next migration.
