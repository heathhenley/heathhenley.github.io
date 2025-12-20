---
title: Simple query refactor - 100x faster
date: 2025-12-20T12:17:28-05:00
draft: false
metathumbnail: /pg_tuple/social.png
description: A simple refactor of a postgres pagination query improved performance significantly, this is an explanation of that update. The basic is that if you have a query like a < c  or (a = c and b < d) - try re-writing it to (a, b) < (c, d). It might be significantly better depending on the specifics of your data. At least take a look at the query plan and see what you get on your system! This post also introduces cursor pagination in a simple way.
tags:
  - postgres
  - software
  - software-engineering
  - optimization
categories:
  - software development
  - software engineering
keywords:
  - postgres
  - software
  - software development
  - software engineering
---
**TL;DR** - Recently set out to speed up a slow query behind a multi-column cursor paginated endpoint. A simple switch to represent the `where` filters as a `tuple` makes a huge difference in performance on the exact same data with the same indices. Eg - change where ( a > c or (a = c and b > d )) to (a, b) > (c, d). In the `tuple` case `postgres` can use the index more efficiently and get more of the needed rows using the index condition, versus walking it, reading rows in and filtering. Feel free to just skip straight to [the simple demo](https://github.com/heathhenley/pg-tuple-comparison) without reading all this if that's more your thing.

## Intro

First I'm going to give some context / background about where the query came from, because I think it might come up in that context for other problems somewhat frequently. In this case, it was paginating through a dataset using a cursor pagination. Skip the next section if you’re already pretty familiar with cursor pagination.
## Cursor Pagination Background

### Limit / Offset

There are different ways to paginate a large dataset - limit / offset is one of the most popular. You pick a well defined sort order, if it were blog posts, maybe it's the most recent - and we request them in chunks. Maybe we start with the first 10, and then next page of results has an offset of 10 and returns 10 more records 11-20 lets say depending on your indexing.

That query might look like: 

```sql
select * from posts
order by created_time desc
limit 10
offset 10
```

### Page Number
Another option that's kind of similar is "page number" pagination, we have a set page size and with that we can generate a similar query to the "page" of data we need. For example limit  = page size, and offset = (page_number - 1) x page_size, the API different, you ask for page=x versus sending limit=x and offset=y - but the idea is the same.

### The Issue w/ these
Limit / offset and page number pagination both work for a lot of cases. But they begin to fall apart on large datasets - as you get deeper into the dataset, they both get slower and slower. The database isn't actually able to use the `offset` to skip straight to correct spot in the results without making the whole result set. So at first, it can seem fast - getting the first 10 blogs is not an issue. But getting blogs 100,000,001 - 100,00,010 - it's going to need to build the whole set and scan through that millionth record - which is... not great. A lot of these implementations also run a query for the count (using a real count and not caching or other tricks like pulling it from the table metadata) - which is very expensive on large tables.
### Cursor Pagination

Another option that performs better on large tables is to use "cursor" pagination. We define a stable ordering of the results, and instead of using a limit and offset to paginate through them, we use a "cursor" - a key that we can use to filter out everything up the last result we've seen in the sorted set. In the blog post example, a simple cursor could be `created_time` - and then our query changes to request the 10 posts with `created_time` less than the last one we’ve seen.

It might look something like: 

```sql
select * from posts
where created_time < "LAST_TIME_SEEN"
order by created_time desc
limit 10
```

With the right index (on `created_time` here), the DB can jump straight to the right spot and it will be much faster. So to page through the dataset like this using cursor pagination - on each page, we just need to figure out what the next “cursor” is (the oldest time on the current page), and send it along with the page. It’s common to base64 encode it to pass it around - so it’s not as immediately obvious, but as a first approximation, that’s really all we’re doing.

#### Cursor Pagination Downsides
One downside of cursor pagination is that you can’t easily jump to a certain spot in the result set - there’s no concept of jumping to middle, end, page 13, or offset 1050, really. You end up with your current page of data, and the cursor you need to get the next one, and that’s it (usually the previous is generated too - but trying to keep it simple). But - if you're dealing with a lot of data and implementing infinite scrolling, or generally only care about next / previous navigation for your use case, then cursor pagination can work. 

### Multi-column Cursor
The example above uses a single column as a cursor, which works if it’s unique. If the column you want to sort by isn’t totally unique, you can use multiple columns to define a unique ordering. It's sort of a tie breaker. For our example with blogs, it could be that maybe some posts have the exact same created date (maybe you moved them off sub stack or medium in bulk, for example). The duplicated times causes problems for our simple single column cursor because the sort order isn’t fixed.

> **Side Note** That exact scenario is actually a simplification of an issue we hit with batch created values and the default [cursor pagination implementation in Django Rest Framework](https://github.com/encode/django-rest-framework/blob/main/rest_framework/pagination.py#L583). Turns out we aren’t [alone](https://github.com/encode/django-rest-framework/discussions/7888) - and this [an alternative with a fix](https://github.com/farsounder/drf-multifield-cursor) was originally a PR against DRF by a contributor (not me) but they don't want to merge.

In summary, you need another column in the “cursor” to enforce a stable ordering. You could for example,  use the id (assuming it’s unique). So we sort by time first, and then any ties are broken by the id. For our page, we just need the pages data and then the `created_time` and id of the last row to use to request the next page. 

Now the query would look something like:

```sql
select * from
  posts
where (created_time < "LAST_TIME_SEEN"
 or (created_time = "LAST_TIME_SEEN" and id < "LAST_ID_SEEN")
)
order by created_time desc, id desc
limit 10
```

Of course you want to have and index on the columns used in the cursor (here `created_time` and `id`), so that in theory the database can use it and get right there.

But here’s where the surprise came for me…
## Switch to Tuple Comparison

### The Switch

Take the where above, and switch it from (a < c ) or (a = c and b < d) to (a, b) < (c, d) - and, at least on some datasets, on Postgres, with the exact same multi-column index available, these equivalent (in this case) queries perform much differently. 

More concretely, the updated query would look something like this:

``` sql
select * from
  posts
where (created_time, id) < ("LAST_TIME_SEEN", "LAST_ID_SEEN")
order by created_time desc, id desc
limit 10
```

I found the "tuple" version using Postgres at least, to be significantly faster. I have not confirmed whether this is the case on other any other database systems, or even whether doing the comparison in this way is supported or means the same thing in say MySQL.
### Reviewing Query Plans
Looking at the query plan, Postgres uses the same index in both cases. Here's an example plan output from the normal approach:

```
Verbose Plan:
  Limit  (cost=0.43..168.15 rows=100 width=16) (actual time=156.380..156.585 rows=100 loops=1)
  Buffers: shared hit=40 read=22309 dirtied=14 written=7525
  ->  Index Only Scan using idx_time_id_prod on perf_test_big_data_table  (cost=0.43..238798.35 rows=142381 width=16) (actual time=156.378..156.577 rows=100 loops=1)
        Filter: ((data_product_id = ANY ('{6,7}'::bigint[])) AND ((row_updated_time > '2020-01-01 20:00:00+00'::timestamp with time zone) OR ((row_updated_time = '2020-01-01 20:00:00+00'::timestamp with time zone) AND (id > 600001))))
        Rows Removed by Filter: 600904
        Heap Fetches: 601004
        Buffers: shared hit=40 read=22309 dirtied=14 written=7525
Planning:
  Buffers: shared hit=3
Planning Time: 0.106 ms
Execution Time: 156.603 ms
```

And then using the "tuple" approach on the same data:

```
Tuple Plan:
  Limit  (cost=0.43..136.49 rows=100 width=16) (actual time=0.014..0.167 rows=100 loops=1)
  Buffers: shared hit=41 read=1
  ->  Index Only Scan using idx_time_id_prod on perf_test_big_data_table  (cost=0.43..193709.74 rows=142371 width=16) (actual time=0.014..0.162 rows=100 loops=1)
        Index Cond: (ROW(row_updated_time, id) > ROW('2020-01-01 20:00:00+00'::timestamp with time zone, 600001))
        Filter: (data_product_id = ANY ('{6,7}'::bigint[]))
        Rows Removed by Filter: 903
        Heap Fetches: 1003
        Buffers: shared hit=41 read=1
Planning Time: 0.051 ms
Execution Time: 0.183 ms
```

So we can confirm it's using the same index in both. However, in the first it’s not able to use the whole clause as the index condition. I guess it only uses one column to read rows in order and needs to filter out TONS of rows. In the second, it’s able to use the clause as the index condition and it gets right to the data we need, filtering significantly fewer rows. It doesn’t have to read in as much data to find the results, so there are way fewer heap fetches and it's much faster.
### Conclusion 
This wasn’t Intuitive to me, I feel like the query planner is pretty smart usually and was surprised it behaved differently for these. Here’s a [small standalone test case](https://github.com/heathhenley/pg-tuple-comparison) that demonstrates the difference on a randomized dataset that I set up to play with the problem before implementing for real.

You can play with the parameters and depending on what the data looks like even a 100x speed up is possible! (depends on the data shape and your computer of course)

One major downside to switching to this approach is that you need to have the ordering on the columns going in the same direction. In that case, you can't easily set up the the tuple as above using a single comparison operator. For example, you can’t easily sort by decreasing time and increasing id. But if that is not important for your use case, and you have similar looking clause somewhere - I recommend you try it out. Profile it and check out the query plan in both cases, and see what happens!

### Key Takeaways

If you just skimmed:
- You might be able to switch (a < c or (a = c and b < d)) with (a, b) < (c, d) for a free performance gain
	- Profile first!
- You can't use it if you need the columns to go in opposite orders
- This wasn't intuitive for me - expected the planner would sort this out
- Standalone example here: https://github.com/heathhenley/pg-tuple-comparison

If you have more insight into this, or if you think I missed something I would happy to hear about it. 

**Some links**
- DRF docs on cursor pagination: https://www.django-rest-framework.org/api-guide/pagination/#cursorpagination
- Disqus blog on it: https://cra.mr/2011/03/08/building-cursors-for-the-disqus-api