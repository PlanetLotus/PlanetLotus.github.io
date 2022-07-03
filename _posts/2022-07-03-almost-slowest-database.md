---
layout: post
title: "The second slowest database in the world"
description: "How fast does this simple database get with only minor modifications?"
tags: [slowdb]
---

_This is part 2 in the Slowest Database series. Be sure to check out [part 1](https://planetlotus.github.io/2022/05/07/the-slowest-database.html) first._

This is nearly the same as the last implementation, except for one optimization. There's one more optimization coming before adding new features to this db which will be in a separate post.

## Store total record count

One of the operations this db supports is getting the total number of records in the database. Previously this involved a full iteration of the file. Next, I'm adding a simple cache of this information by storing the total record count in another file. My assumption is that this would slightly slow down inserts and deletes, but hopefully not noticeably.

The core of this feature involves three new methods:

{% highlight csharp %}
public int GetCount()
{
    if (!File.Exists(countFilePath))
    {
        return 0;
    }

    var line = File.ReadLines(countFilePath).First();
    var count = int.Parse(line);

    return count;
}

private async Task SetCount(int newValue)
{
    await File.WriteAllTextAsync(countFilePath, newValue.ToString());
}

private async Task IncrementCount(int amount)
{
    var currentCount = GetCount();
    var newCount = currentCount + amount;
    await SetCount(newCount);
}
{% endhighlight %}

Quick summary of what's changing here:

* `GetCount()` now reads an integer from a file instead of iterating the data line by line, keeping track of the total count
* `SetCount()` is used directly by the `Delete()` operation, incurring an additional loop to count the number of lines being saved to the data file
* `IncrementCount()` is used directly by the `Insert()` operation

I made other changes to the code to make it easier to extend, mostly just refactoring. I find it boring to write about refactoring, but let me know if you want more details.

## Benchmark results

I made a few changes to the benchmark since last time. I added timings for seeding the database (currently just a 1 MB file), inserts, and deletes, as well as the ability to benchmark all implementations at once without making code changes. The main thing left here (besides adding more tests) is improved formatting so it's easier to compare implementations, but I will put time into this in the future.

| | Seed 1 MB | Insert 10 words | Get total count | Search 10 common words | Search 10 less common words | Search 10 made up words | Delete 10 words |
| - | --------- | --------------- | --------------- | ---------------------- | --------------------------- | ----------------------- | --------------- |
| StringDb (original) | 692 ms | 8 ms | 57 ms | 182 ms | 177 ms | 171 ms | 1823 ms |
| StringDb (new) | 730 ms | 17 ms | 0 ms | 188 ms | 179 ms | 171 ms | 2161 ms |

## Thoughts

These results are mostly in line with expectations. Seeding should be only slightly slower because there's one additional file write at the end. Inserts are ~2x slower which makes sense because the time is dominated by the I/O, and there's now double the I/O happening. Searches are the same, which makes sense because nothing changed here. Deletes are only marginally slower. This was the biggest surprise. I think this is because my delete algorithm spends most of its time iterating the file looking for deletions, and the I/O at the end is minor in comparison.

The big question is whether this performance tradeoff is worth it. Obviously having to iterate the db to get a total record count isn't ideal and storing it is simple, but slowing down writes by a factor of 2 is probably not okay. I'll have to think about this more to see if there are any simple tweaks that could make this faster. While I have plenty of more advanced ideas, I'm intentionally staying out of complicated changes as long as I can; the point of this project is observing algorithmic complexity in the real world with only minimal code.

## Next steps

Part 3 will be about taking this same database and storing it sorted. I expect this will improve search speed and significantly reduce insert and delete speed. The question is whether I can make the tradeoff reasonable without getting too into the weeds.
