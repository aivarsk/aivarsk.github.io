---
layout: post
title: Making sense of card range query EXPLAIN plans
tags: PostgreSQL, Django, PAN, card range, fintech
---

I was listening to [a talk about finding a PAN in a haystack](https://www.youtube.com/watch?v=eTueJyRw7qQ) and this slide came up with performance numbers in seconds:

![slow card range query]({{ site.url }}/public//cardrange.png)

This topic is close to my heart because it involves FinTech, databases, and performance investigation/tuning.

A card number or PAN or Primary Account Number is a series of 12 to 19 digits. Most popular VISA and MasterCard cards use 16 digits but card systems must handle the variable length for PANs.

And then VISA and MasterCard and other card organizations provide a list of ranges: start value, end value, or sometimes just 6 to 9 first digits. For each range there are different properties and rules for transactions allowed, regions supported, etc. Ranges provided by organizations overlap and have contradicting rules but I will leave this out. And there are 100s of thousands of ranges.

So how do you check to which card range the given PAN belongs? The best way is to keep it in memory and have 2 sets of data: the active one and the one for loading a new data set. Once it's loaded, you just switch a flag/pointer to indicate which is the active one. Exactly like the "finding a PAN in a haystack" talk described.

But you can also do the same with the database and it's not that slow when done correctly. It was working well enough on spinning disks and it gets only better with SDDs. There are better solutions for modern databases, but let's stick with this one that worked in the early 2000s.

First, I will store the range as strings or `varchar` in the database language. You can do numbers, but when I first worked on this problem 64 bits were not supported in some parts of the system.

Second, to work with any PAN length I will always add padding to make `start`, `end`, and PAN 19 digits. That being said, there is this nice property:

```python
>>> '4500000000000000' < '4567891234567890'
True
>>> '45' < '4567891234567890'
True
```

I will remove the trailing `0` from the start of the range to reduce the size of the table and index records.

To store data I will create a simple table:

```python
class CardRange(models.Model):
    start_range = models.CharField(max_length=19)
    end_range = models.CharField(max_length=19)
    # Just something to store along the range
    info = models.CharField(max_length=64)

    class Meta:
        indexes = [
            models.Index(
                fields=["start_range", "end_range"], name="card_range_start_end_idx"
            ),
        ]
```

The `CardRange` will have the start and end values of the range along with some properties we want to retrieve for the card. I will have just the `info` field to fill some extra space on the disk. Most importantly, I create an index on the start and end range fields because I will use that for query.

Let's apply the initial migration and see how it works:

```bash
python3 manage.py migrate cardrange 0001
```

Once that is done we can generate some test data, around 300,000 rows.

```python
def card_ranges():
    for i in range(300000, 599999):
        start = str(i) + "0000000000"
        end = str(i) + "9999999999"
        yield (start, end)


for start, end in card_ranges():
    CardRange.objects.create(
        start_range=start.rstrip("0"), end_range=end.ljust(19, "9"), info="whatever"
    )
```

The query to find the range PAN belongs to is the following one:

```python
CardRange.objects.filter(start_range__lte="4567891234567890".ljust(19, "0"), end_range__gte="4567891234567890".ljust(19, "0")).only("info")
```

However, I will need to do some SQL tricks, so let's go and execute the query as SQL directly:

```sql
postgres=# EXPLAIN ANALYZE SELECT info FROM cardrange_cardrange WHERE start_range <= '4567891234567890000' AND end_range >= '4567891234567890000';
                                                       QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 Seq Scan on cardrange_cardrange  (cost=0.00..7303.98 rows=74836 width=9) (actual time=57.292..81.354 rows=1 loops=1)
   Filter: (((start_range)::text <= '4567891234567890000'::text) AND ((end_range)::text >= '4567891234567890000'::text))
   Rows Removed by Filter: 299998
 Planning Time: 0.521 ms
 Execution Time: 81.375 ms
(5 rows)
```

Are you surprised the database did not use the index for the query? The reason for that is: according to information available to the database, around half of the table records have `start_range` less than '4567891234567890000' and the database uses indexes when they allow to select a small subset of rows, let's say less than 5% of the table. (The number may differ but it's a single digit). We know it's always a single row, but we do not have a way to convince the database about it.

Instead, let's force the database to use the index. For Postgres, I had to install an extension. Oracle database has this built-in already.

```bash
apt install postgresql-15-pg-hint-plan
```

And then I had to load the extension:

```sql
postgres=# load 'pg_hint_plan';
LOAD
```

Another option for lazy people is to disable table scans and force the database to use indexes for all queries by executing the following command: `set enable_seqscan=false`.

I have the extension and can pass a hint to the database about using the index:

```sql
postgres=# /*+ IndexScan(cardrange_cardrange) */ EXPLAIN ANALYZE SELECT info FROM cardrange_cardrange WHERE start_range <= '4567891234567890000' AND end_range >= '4567891234567890000';
                                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using card_range_start_end_idx on cardrange_cardrange  (cost=0.42..11425.48 rows=74836 width=9) (actual time=51.953..51.955 rows=1 loops=1)
   Index Cond: (((start_range)::text <= '4567891234567890000'::text) AND ((end_range)::text >= '4567891234567890000'::text))
 Planning Time: 0.533 ms
 Execution Time: 52.001 ms
(4 rows)
```

It got slightly better, mostly because the index has a more compact structure, but it's still slow. Let's try to hint to the database that we need only a single row by using `LIMIT 1`. But that produces a similar result:

```sql
postgres=# /*+ IndexScan(cardrange_cardrange) */ EXPLAIN ANALYZE SELECT info FROM cardrange_cardrange WHERE start_range <= '4567891234567890000' AND end_range >= '4567891234567890000' LIMIT 1;
                                                                          QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.42..0.58 rows=1 width=9) (actual time=49.012..49.013 rows=1 loops=1)
   ->  Index Scan using card_range_start_end_idx on cardrange_cardrange  (cost=0.42..11425.48 rows=74836 width=9) (actual time=49.009..49.009 rows=1 loops=1)
         Index Cond: (((start_range)::text <= '4567891234567890000'::text) AND ((end_range)::text >= '4567891234567890000'::text))
 Planning Time: 0.497 ms
 Execution Time: 49.058 ms
(5 rows)
```

Now I will change the index slightly and recreate it. Can you spot the difference?

```python
    class Meta:
        indexes = [
            models.Index(
                fields=["-start_range", "end_range"], name="card_range_start_end_idx"
            ),
        ]
```

The difference was the `-` sign in front of the `start_range` field and that causes the field to be indexed in descending order. The SQL will have `"card_range_start_end_idx" btree (start_range DESC, end_range)`. Let's apply the migration:

```bash
python3 manage.py migrate cardrange 0002
```

First, let's try the query without `LIMIT 1`:

```sql
postgres=# /*+ IndexScan(cardrange_cardrange) */ EXPLAIN ANALYZE SELECT info FROM cardrange_cardrange WHERE start_range <= '4567891234567890000' AND end_range >= '4567891234567890000';
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using card_range_start_end_idx on cardrange_cardrange  (cost=0.42..11429.48 rows=74836 width=9) (actual time=0.286..58.670 rows=1 loops=1)
   Index Cond: (((start_range)::text <= '4567891234567890000'::text) AND ((end_range)::text >= '4567891234567890000'::text))
 Planning Time: 1.050 ms
 Execution Time: 58.715 ms
(4 rows)

We still get the same result as for the previous index. But let's add the `LIMIT 1` condition back to the query:

postgres=# /*+ IndexScan(cardrange_cardrange) */ EXPLAIN ANALYZE SELECT info FROM cardrange_cardrange WHERE start_range <= '4567891234567890000' AND end_range >= '4567891234567890000' LIMIT 1;
                                                                         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.42..0.58 rows=1 width=9) (actual time=0.115..0.116 rows=1 loops=1)
   ->  Index Scan using card_range_start_end_idx on cardrange_cardrange  (cost=0.42..11429.48 rows=74836 width=9) (actual time=0.113..0.113 rows=1 loops=1)
         Index Cond: (((start_range)::text <= '4567891234567890000'::text) AND ((end_range)::text >= '4567891234567890000'::text))
 Planning Time: 0.370 ms
 Execution Time: 0.151 ms
(5 rows)
```

And here we go, the query execution time is down from 58.715ms to 0.151ms! Is it magic? No. Let me tell you how indexes and index range scans work.

Multicolumn indexes are similar to single field indexes - the multiple values are concatenated with some separators between them. Indexes are sorted on those concatenated values. When you are looking for an exact match, the tree structure helps to get there quickly.

For index range scans database will do a sequential scan of the index and the tree structure helps to find the starting point for the scan (a high-level overview).

First, let's produce an ordered list of start and end ranges as if those were stored in an index:

```
def card_ranges():
    for i in range(10, 99, 10):
        yield (str(i) + "0", str(i) + "9")
```

And let's emulate an index range scan.

```python
for low, high in card_ranges():
    if low < "405":
        print("check", (low, high))
        if "405" < high:
            print("match", (low, high))
    else:
        print("skip ", (low, high))
```

For our dataset, half of the rows will have a start value less than the value we look for, and half of the rows will have an end value larger than the value we look for. During execution, the index range scan will start with the first entry in the index and will continue to compare half of the rows. The last row with the start value less than what we are looking for will be our match.

```python
check ('100', '109')
check ('200', '209')
check ('300', '309')
check ('400', '409')
match ('400', '409')
skip  ('500', '509')
skip  ('600', '609')
skip  ('700', '709')
skip  ('800', '809')
skip  ('900', '909')
```

Now let's do the same but for the index sorted in descending order: 

```python
for low, high in reversed(card_ranges()):
    if low < "405":
        print("check", (low, high))
        if "405" < high:
            print("match", (low, high))
    else:
        print("skip ", (low, high))
```

Here the scan starts by skipping the index entries. The tree structure will help to jump over those quickly. And then the first entry in the index we check will also be our match. Unfortunately, the index scan will continue to search for any other ranges that satisfy the conditions. This is the reason just changing the index was not enough, I had to force it to return after a single row is found with `LIMIT 1`.

```python
skip  ('900', '909')
skip  ('800', '809')
skip  ('700', '709')
skip  ('600', '609')
skip  ('500', '509')
check ('400', '409')
match ('400', '409')
check ('300', '309')
check ('200', '209')
check ('100', '109')
```

As a backup, here is [a repository of this exercise](https://github.com/aivarsk/django-cardrange). Come and see me at [PyCon Austria](https://pycon.pyug.at/en/speakers/) and [PyCon Italy](https://2025.pycon.it/en/event/querysetexplain-make-it-make-sense) with more database query shenanigans.
