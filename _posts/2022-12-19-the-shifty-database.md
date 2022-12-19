---
layout: post
title: "The shifty database"
description: "Time to move some bytes around."
tags: [slowdb]
---

_This is part 4 in the Slowest Database series. Be sure to check out [part 1](https://planetlotus.github.io/2022/05/07/the-slowest-database.html) first._

## Background

[Last time](https://planetlotus.github.io/2022/12/17/the-database-that-never-lets-go.html), we greatly improved delete performance by not actually deleting data, just making it unsearchable and keeping track of what was removed. This isn't much of a database though if it can't free space, and it also made searches much slower as the list of deleted records grew.

This time, our goal will be to actually free the disk space and not slow down searching as the freelist grows.

## Implementation

This is a big step up in complexity. I'm recalling a quote from the first post of the series:

> But, generally, I want to stay away from low level stuff in this experiment.

Unfortunately, that's out. Fundamentally, we can't afford to rewrite the file for every delete operation. In order to do this in the middle of the file, any data that comes after the data being deleted has to be moved in order to replace what was there. As far as I know, no matter what, this involves dropping down to the byte-level.

That said, we're still limiting ourselves to this database design, which means every newline represents the start of a new record. Therefore, it's easy to distinguish between records and it's also easy to debug just by looking at the file.

Algorithm details:

1. As before, loop over each line in the file, looking for records we want to delete (the file is still not sorted).
2. For each matched line, calculate its byte offset in the file and keep track of this in a list.
3. Now, loop over the list of byte offsets. For each, calculate the number of bytes we need to delete and the number of bytes between this record and the next record we're deleting (or the end of the file, if there is none).
4. Read the "in between" bytes into a buffer and write them starting at the current byte offset.
5. After doing this for all replacements, overwrite the length of the file to the last byte offset written.

This does take two full loops over the file in the worst case. It could be made to take one but it would complicate things.

{% highlight csharp %}
public async Task Delete(string data)
{
    var byteIndexesToDelete = new List<long>();

    var numBytesPerDelete = Encoding.UTF8.GetByteCount(data) + Environment.NewLine.Length;

    var currentIndex = 0;
    var numDeletions = 0;

    var lines = File.ReadLines(filePath);
    foreach (var line in lines)
    {
        if (line == data)
        {
            byteIndexesToDelete.Add(currentIndex);
            numDeletions++;
        }

        currentIndex += Encoding.UTF8.GetByteCount(line) + Environment.NewLine.Length;
    }

    if (byteIndexesToDelete.Any())
    {
        FreeSpace(byteIndexesToDelete, numBytesPerDelete);
    }

    await SetCount(rowCount - numDeletions);
}

private void FreeSpace(List<long> byteIndexes, int numBytesPerIndex)
{
    using var stream = File.Open(this.filePath, FileMode.Open);

    var writeIndex = byteIndexes[0];

    for (var i = 0; i < byteIndexes.Count; i++)
    {
        var lastIndex = i + 1 == byteIndexes.Count;
        var readIndex = byteIndexes[i] + numBytesPerIndex;

        var numBytesToCopy = lastIndex
            ? stream.Length - readIndex
            : byteIndexes[i + 1] - readIndex;

        if (numBytesToCopy > 0)
        {
            // Copy bytes in batches to limit max RAM consumed
            var lastBatchSize = numBytesToCopy % MaxBufferSizeInBytes;
            var numBatches = numBytesToCopy / MaxBufferSizeInBytes + 1;

            for (var batchNum = 0; batchNum < numBatches; batchNum++)
            {
                var lastBatch = batchNum == numBatches - 1;
                var bufferSize = lastBatch ? lastBatchSize : MaxBufferSizeInBytes;

                CopyBytes(stream, bufferSize, readIndex, writeIndex);
                readIndex += bufferSize;
                writeIndex += bufferSize;
            }
        }
    }

    stream.SetLength(writeIndex);
    stream.Flush();
}

private static void CopyBytes(FileStream stream, long bufferSize, long readIndex, long writeIndex)
{
    var buffer = new byte[bufferSize];

    stream.Position = readIndex;
    stream.Read(buffer, 0, buffer.Length);

    stream.Position = writeIndex;
    stream.Write(buffer, 0, buffer.Length);
}
{% endhighlight %}

I went ahead and included a couple not-strictly-necessary details to make this work more generically:
                                                         
1. It handles UTF8 characters and adjusts sizing accordingly
2. There's a max buffer size so we don't consume all RAM in the worst-case

## Results

| Operation             | [Part 1](https://planetlotus.github.io/2022/05/07/the-slowest-database.html) | [Part 2](https://planetlotus.github.io/2022/07/03/almost-slowest-database.html) | [Part 3](https://planetlotus.github.io/2022/12/17/the-database-that-never-lets-go.html) | Part 4 |
| --------------------- | -------- | -------------- | ------------------ | ------------------ |
| Seed 1 MB             | 406      | 374            | 385                | 371                |
| Insert 10             | 4        | 24             | 18                 | 18                 |
| Insert 10 less common | 2        | 19             | 17                 | 15                 |
| Get total count       | 72       | 1              | 1                  | 0                  |
| Search 10             | 2        | 1              | 4                  | 1                  |
| Search 10 less common | 35       | 48             | 32                 | 50                 |
| Search 10 non\-words  | 145      | 158            | 148                | 144                |
| Delete 10             | 1434     | 1597           | 177                | 763                |
| Delete 10 less common | 1371     | 1531           | 156                | 511                |
| Delete 10 non\-words  | 1390     | 1505           | 155                | 194                |
| Insert 10 less common | 5        | 18             | 17                 | 21                 |
| Search 10 less common | 156      | 146            | 633                | 136                |

I'm very pleased with the results. Focusing on the delete operation, it's obviously slower than the previous iteration where we didn't actually free any disk space, but it's worst-case twice as fast as the original delete, and as the number of deletes per test decreases, the bigger the performance difference. This is because the original implementation requires rewriting the entire file regardless, which has more overhead than iterating the file looking for data to delete.

While writing this I noticed a performance bug in the original StringDb implementation that makes the last delete operation much slower than it should be. With just a couple lines, we could ensure no files are rewritten if there's nothing to delete, but this was an oversight originally.

The byte manipulation could be faster, but I'm hoping to focus more on the data structures used which I think will make the bigger difference in most cases.

## Next steps

The next most obvious thing to improve is search speed, and I don't know how much better it can get without keeping the data sorted on disk. I know this is going to be complicated, so I'm going to attempt to create some intermediate data structures so that the next step isn't B-trees with slotted pages. One step at a time!
