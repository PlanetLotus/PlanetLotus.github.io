---
layout: post
title: "The slowest database in the world"
description: "How slow can we make a database if we try? Is it still faster than you thought?"
tags: [slowdb]
---
Think of a file with one word on each line and the file is 1 MB. How long would it take to find a specific word in that file?

Let's make this more concrete with some design details for this "database":
* A text file
* One record == one word (space-delimited)
* Duplicates are allowed
* Supports insert, find, count, and sizeInBytes operations

This isn't a very useful database, but it may inform us how far we can get without the advanced database magic of today. I want to study this in iterations. This is the first iteration.

I created two programs to assist with this:

* A simple benchmarking program
* A word generator that pulls articles from Wikipedia (to get "real" data and not made up data)

I'm not going to get into the details of these programs but may later.

## Implementation

To give a concrete idea of how this database works, here's the implementation for the read operations:

{% highlight csharp %}
public async Task<string> Find(string data)
{
    var lines = File.ReadLines(filePath);
    foreach (var line in lines)
    {
        if (line == data)
        {
            return line;
        }
    }

    return await Task.FromResult<string>(default);
}

public int GetCount()
{
    if (!File.Exists(filePath))
    {
        return 0;
    }

    var lines = File.ReadLines(filePath);
    var count = 0;

    foreach (var line in lines)
    {
        count++;
    }

    return count;
}

public long GetSizeInBytes()
{
    if (!File.Exists(filePath))
    {
        return 0;
    }

    var info = new FileInfo(filePath);
    return info.Length;
}
{% endhighlight %}

Before reading the next section, imagine how fast these functions are on a 1 MB file and a 1 GB file. See whether your intuition is within an order of magnitude of the results.

## Benchmark results

I generated a 1 MB and a 1 GB database. For this iteration, I'm focused on read operations, not writes.

This is running on the following hardware:
    
```
i5-4670 3.4 GHz
16 GB RAM
Samsung 850 EVO SSD
```
    
1 MB:

```    
Found 387,518 records in 33 ms
Found 10/10 CommonWords in 1 ms
Found 10/10 LessCommonWords in 27 ms
Found 0/10 NotWords in 344 ms
```  
    
1 GB:

```
Found 136,812,515 records in 7062 ms
Found 10/10 CommonWords in 1 ms
Found 10/10 LessCommonWords in 27 ms
Found 0/10 NotWords in 70167 ms
```

## Thoughts

I knew it would be faster than my intuition, but I'm still surprised we can loop over 387K lines in a file (not read into memory) in just 33ms. That's incredible. Then in the 1 GB file, there are 353x more lines and it takes 214x as long. Not quite a linear increase but seems reasonable.

Timings between common words, less common words, and "not" words makes sense. Common words are likely to be toward the top of the file so it can find & return those quickly. Less common words, it has to go further through the file. For "not" words, it has to iterate the entire file only to return nothing. It makes sense that trying to find a non-word 10 times takes 10x as long as counting the records. This all adds up.

Still, in absolute terms, this is fast to me. Milliseconds are a long time for a computer but if we're thinking about human-perceived latency on operations, a lot of these timings are really good, even when operating on 1 GB of data.

## Limitations

It's difficult to generate several GBs of realistic data. I may look into downloading some public databases to help seed the db. I'm adamant about using realistic data, not a database full of lorem ipsum text.

While working on the implementations, I noticed a big limitation is the high level operations in C#. I spent a little time optimizing writes just so I could seed a 1 GB database in a reasonable amount of time (in this case, it was to use a StreamWriter to write in bursts instead of opening/closing the file for each append). But, generally, I want to stay away from low level stuff in this experiment. We'll see if this ends up becoming a limiting factor in what we can achieve.

## Next steps

Since we have some timings in seconds and others in the tens of milliseconds, I feel this is a good point to stop and do a new implementation to see what optimizing this will get us. I want to get into sub-second latencies for all of these read operations and have that scale much further. This is all an experiment, and I'm winging it. We'll see where it goes!
