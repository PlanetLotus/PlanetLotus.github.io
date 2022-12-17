---
layout: post
title: "The database that never lets go"
description: "Slowest Database part 3: Insufficient free disk space"
tags: [slowdb]
---

_This is part 3 in the Slowest Database series. Be sure to check out [part 1](https://planetlotus.github.io/2022/05/07/the-slowest-database.html) first._

## Background

In [part 2](https://planetlotus.github.io/2022/07/03/almost-slowest-database.html), we left off with deletes being the slowest operation, at around 2 seconds to delete 10 words in only a ~1 MB file. This seems like the place to focus.

The last delete implementation worked like this:

1. Loop over the file looking for all occurrences of the string to be deleted
2. Buffer the entire database, post-delete, in memory
3. Write this into a new file
4. Delete the old file

Aside from the performance, this has big problems with RAM consumption; our database can only do deletes if we can hold the entire thing in memory. This doesn't work.

## UnbufferedStringDb

I originally planned on part 3 being about storing the data in sorted order, but storing data sorted on disk is surprisingly complicated. We'll get there, but I want to do simpler intermediate steps first. The next step we'll take is figuring out how to not buffer and write the entire file post-delete. This will involve:

1. Identifying the line indexes of the file that need deleted
2. Marking them deleted (not actually removing data at this point) so that they are unsearchable
3. Updating our read logic so that it doesn't detect these "deleted" records

This doesn't free any disk space. This only makes data unsearchable. We'll continue to iterate on this.

Relevant changes in this iteration:

{% highlight csharp %}
public async Task Delete(string data)
{
    var deletedIndexes = new List<string>();
    var currentIndex = 0;

    var lines = File.ReadLines(filePath);
    foreach (var line in lines)
    {
        if (line == data)
        {
            deletedIndexes.Add(currentIndex.ToString());
        }

        currentIndex++;
    }

    await SetCount(rowCount - deletedIndexes.Count);

    File.AppendAllLines(freeListFilePath, deletedIndexes);
}

public async Task<string> Find(string data)
{
    var lines = File.ReadLines(filePath);
    var currentIndex = 0;

    foreach (var line in lines)
    {
        if (line == data && !IsDeleted(currentIndex))
        {
            return line;
        }

        currentIndex++;
    }

    return await Task.FromResult<string>(default);
}

private bool IsDeleted(int recordIndex)
{
    var lines = File.ReadLines(freeListFilePath);
    foreach (var line in lines)
    {
        if (int.Parse(line) == recordIndex)
        {
            return true;
        }
    }

    return false;
}
{% endhighlight %}

Our Find method now naively checks a file to see if the current index is marked deleted, and if so, skips it.

## Benchmark results

The big change in benchmarking here is adding an additional insert and search after deletions are run. This is relevant in this implementation because, as you might expect, searches get slower as the "freelist" grows, and the freelist never gets smaller.

All values are in milliseconds.

| Operation             | StringDb | StringDb2 | UnbufferedStringDb |
| --------------------- | -------- | -------------- | ------------------ |
| Seed 1 MB             | 893      | 935            | 921                |
| Insert 10             | 16       | 12             | 13                 |
| Insert 10 less common | 4        | 20             | 14                 |
| Get total count       | 52       | 1              | 0                  |
| Search 10             | 1        | 16             | 20                 |
| Search 10 less common | 24       | 29             | 34                 |
| Search 10 non\-words  | 196      | 198            | 206                |
| Delete 10             | 2014     | 2254           | 263                |
| Delete 10 less common | 2015     | 2051           | 226                |
| Delete 10 non\-words  | 2002     | 2206           | 223                |
| Insert 10 less common | 5        | 12             | 13                 |
| Search 10 less common | 237      | 204            | 1077               |

## Thoughts

This achieved the primary goal of speeding up deletes (~10x faster), but had the cost of slowing down searches as the freelist grows (~5x slower at this size). This was anticipated, though I will admit the slowdown was bigger than I figured even at a fairly small freelist (0.5 MB).

There were some optimizations to the freelist that I intentionally avoided at this point, such as keeping some or all of it in RAM. One of the goals of this project is focusing on how things are stored on disk. Buffering data in memory makes many performance problems trivial, but is then limited by system RAM. I'm making careful considerations here, only consuming RAM where the max it can take up is predictable and tolerable (such as the total record count fitting into a single int).

## Next steps

The big problem now is search performance after deletions, so let's focus there. Stay tuned!
