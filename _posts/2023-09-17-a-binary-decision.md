---
layout: post
title: "A binary decision"
description: "It finally gets fast-ish."
tags: [slowdb]
---

_This is part 5 in the Slowest Database series. Be sure to check out [part 1](https://planetlotus.github.io/2022/05/07/the-slowest-database.html) first._

Background
==========

Last time, we stepped down into the byte manipulation level to make deletes faster. Fundamentally, search operations were still slow though, and since every other operation depends on searching it first, this became the limiting factor. This next iteration rewrites the search to use binary search on the data, where the data is stored in sorted order on disk.

Implementation
==============

Aside: I should mention, this took two tries. The first approach would've worked but ended up way more complicated. I got about halfway through before getting a simpler idea and switching to that. In total, I think this iteration took about a dozen programming sessions...a big step up from previous iterations which took 2-4 sessions each.

I wanted to see how far I could get with a human-readable database file, so this iteration still uses a text file. The challenge is, how do you do binary search when you don't know where each record is, at the byte level? Before we can search on records, we first have to be able to identify records when all we know are bytes. Typically this might be done with a binary file format where you store byte offsets you can "jump" to (in other words, a B-tree on disk), and this may get there, but again, I'm seeing how far this human-readable file can go.

Algorithm:

1. Calculate the starting midpoint for binary search by taking the total byte length of the database and dividing by 2.
2. Scan bytes rightward from the midpoint until we reach a word boundary (newline). Save that as the "end".
3. Scan bytes leftward from the midpoint until we reach a word boundary (newline). Save that as the "start".
4. Now we know exactly which byte range comprises our record. Convert to string and check for a match. If match, return.
5. Based on the comparison, determine whether to search in the upper or lower half.
6. Repeat 1-5 until found or no more records.

Steps 2 & 3 are the magic here, picking an arbitrary byte in the file and figuring out what record it's in the middle of. This is easily the most complicated part of the logic. While it doesn't take many lines of code (the entire db implementation is still only ~500), I spent the most time debugging these two "scan bytes" methods.

Once this was working, I needed to update the Insert and Delete methods to take advantage of the new search. For Insert, this meant a slightly modified search to identify the byte index at which to insert. For Delete, something similar.

If I were to do this over again, I would try to combine steps 2 & 3 so that it's only one "rightward" search. This was less intuitive to me because it means in many cases you'll completely skip the midpoint you start on and I wasn't sure how the edge cases would play out. But, scanning for bytes leftward was the buggiest part of this without a doubt and I probably made things harder than it needed to be.

This is the basic Find loop with the binary search algorithm, hiding all of the difficult details about figuring out where a record begins and ends:

{% highlight csharp %}
...

while (true)
{
	var noMoreRecords = minPosition >= this.stream.Length - Environment.NewLine.Length;
	if (minPosition >= maxPosition || noMoreRecords)
	{
		return null;
	}

	var midpoint = (maxPosition + minPosition) / 2;

	var (left, _) = GetLeftPieceOfRecord(this.stream, midpoint);
	var (right, _) = GetRightPieceOfRecord(this.stream, midpoint);
	var currentRecord = left + right;

	var comparison = data.CompareTo(currentRecord);
	if (comparison == 0)
	{
		return currentRecord;
	}
	else if (comparison < 0)
	{
		maxPosition = midpoint - 1;
	}
	else
	{
		minPosition = midpoint + 1;
	}
}
{% endhighlight %}

Results
=======

| Operation             | [Part 1](https://planetlotus.github.io/2022/05/07/the-slowest-database.html) | [Part 2](https://planetlotus.github.io/2022/07/03/almost-slowest-database.html) | [Part 3](https://planetlotus.github.io/2022/12/17/the-database-that-never-lets-go.html) | [Part 4](https://planetlotus.github.io/2022/12/19/the-shifty-database.html) | Part 5 |
| --------------------- | -------- | -------------- | ------------------ | ------------------ | -------------------- |
| Insert 10             | 6        | 40             | 27                 | 42                 | 89                   |
| Insert 10 less common | 2        | 33             | 36                 | 26                 | 29                   |
| Get total count       | 45       | 10             | 0                  | 0                  | 0                    |
| Search 10             | 0        | 14             | 35                 | 27                 | 2                    |
| Search 10 less common | 11       | 16             | 14                 | 12                 | 2                    |
| Search 10 non\-words  | 108      | 108            | 111                | 97                 | 3                    |
| Delete 10             | 1318     | 1439           | 130                | 679                | 33                   |
| Delete 10 less common | 1277     | 1446           | 115                | 534                | 28                   |
| Delete 10 non\-words  | 1311     | 1334           | 114                | 164                | 4                    |
| Insert 10 less common | 2        | 26             | 12                 | 38                 | 35                   |
| Search 10 less common | 117      | 110            | 511                | 87                 | 5                    |

Now the exciting part:
* Search is consistently fast regardless of whether the record exists. Compare to earlier iterations where existence made a difference in performance.
* Delete is dramatically faster but is slower depending on how much is deleted (common words vs. words that aren't found).
* Insert is about twice as slow because it's no longer just an append; it has to keep the file sorted on disk which is much more work.

Some updates on the benchmark...we can no longer compare this benchmark to older blog posts because too much has changed:

* I'm running a new PC (relevant specs: i5-13600K, 32 GB DDR5, Samsung 980 Pro M.2)
* The benchmark db file size is now slightly larger

That said, comparing the previous iterations all at once still works. This is on a roughly 2 MB file and all times are in milliseconds.

The timings are low enough on a 2 MB database that there's significant variations between runs. My benchmarks don't account for this. The results above are among the worst runs I saw. In other cases I noticed 100-200x improvements. Because of this, it actually makes sense now to increase the size of the db before measuring, but the drawback is that it's not practical to compare with previous iterations at larger sizes because it takes too long to run.

Here are results for a 200 MB database (100x larger) but just for the current iteration:

| Operation             | Part 5 |
| --------------------- | -------------------- |
| Insert 10             | 973                  |
| Insert 10 less common | 1110                 |
| Get total count       | 0                    |
| Search 10             | 2                    |
| Search 10 less common | 3                    |
| Search 10 non\-words  | 2                    |
| Delete 10             | 1077                 |
| Delete 10 less common | 993                  |
| Delete 10 non\-words  | 2                    |
| Insert 10 less common | 1024                 |
| Search 10 less common | 7                    |

Critically, the search timings are about the same, proving that better-than-linear time complexities work well!

Insert doesn't scale as well because as the database grows, more bytes have to be moved to "make space" for new ones and to keep everything in sorted order. This is the biggest issue. Delete doesn't scale well more because of what I put in the database. The database is a list of words and each word can appear more than once, particularly common words like "a", "and", etc. I noticed that the 30 delete operations in the benchmark end up deleting 6% of the database! Note how fast deleting "non-words" (nothing to delete) is compared to the first delete (common words, many matches in the db). This is highly unusual for most databases so this isn't a great test. This can be improved with more realistic data patterns.

Lastly, I removed the timings for "seeding" the database because in order to make it practical, I had to cheat. I sorted the data in memory before writing it to the file. This is only done in the seed operation and not anywhere else. Based on the current designs, a proper bulk insert would be a project on its own which was too much scope creep.

Next steps
==========

I'm not committed to a single idea yet, but right now I plan to add some features to the database while keeping the text file format. I want to add basic data types (say string, integer, boolean) and add support for different columns. I think this could help make the Deletes a bit more realistic because deleting a single word would no longer necessary have to delete a large portion of the database, but this may also require adding basic query support which might be a stretch. We'll see where it goes!
