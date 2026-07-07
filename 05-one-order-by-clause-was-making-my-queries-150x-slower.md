# One ORDER BY Clause Was Making My Queries 150x Slower

I had a query that took 45 seconds. Every index I'd designed in the last post looked reasonable on paper. The table had an index on the actor columns, an index on the date, an index on article count. And yet, asking for events similar to a given event fingerprint still took 45 seconds to come back. The problem, it turned out, had nothing to do with which indexes existed. It had everything to do with how the query itself was written.

## The query in question

Here's what the `similar_events` lookup looked like, given an event fingerprint like `EVT-2024-02-29-1160747286`:

```sql
SELECT GlobalEventID FROM events_table
WHERE SQLDATE BETWEEN '2024-02-01' AND '2024-03-31'
  AND GlobalEventID != 1160747286
  AND (Actor1Name = 'GOVERNMENT' OR Actor2Name = 'GOVERNMENT')
ORDER BY NumArticles * ABS(GoldsteinScale) DESC
LIMIT 20
```

Reading it, nothing screams "45 seconds." That's what made it worth digging into.

## Three sins in one query

Running `EXPLAIN` on this exposed three separate problems stacked on top of each other:

**The `ORDER BY` was sorting on a computed expression.** `NumArticles * ABS(GoldsteinScale)` isn't a column — it's a calculation performed at query time. No index can be built on the result of a multiplication done on the fly, so MySQL had no choice but to fall back to a filesort: pull every matching row, compute the expression for each one, then sort the whole set in memory.

**The `OR` condition triggered an index merge.** `(Actor1Name = X OR Actor2Name = X)` forced MySQL into `index_merge sort_union(idx_actor1, idx_actor2)` — it scanned both indexes separately, then merged and deduplicated the results. On this table, that meant touching 549,348 rows before getting anywhere near the final 20.

**The index merge still required a trip back to the table.** After the merge, MySQL still had to fetch `NumArticles` and `GoldsteinScale` from the actual rows to compute the sort expression, since neither of those values lived in the indexes it had just scanned.

Each of these problems was survivable on its own. Stacked together, on a table with 15 million rows, they compounded into 45 seconds.

## The fix, in two steps

**Step one: replace the expression sort with a single-column sort.**

```sql
-- Before
ORDER BY NumArticles * ABS(GoldsteinScale) DESC
-- After
ORDER BY NumArticles DESC
```

This isn't a perfect substitute — `NumArticles` alone doesn't capture what `NumArticles * ABS(GoldsteinScale)` was trying to express. But it's close enough for the use case, and critically, `NumArticles` on its own can be satisfied by the `idx_numarticles` index I mentioned in the last post as an unremarkable addition. It wasn't unremarkable. It was the index that made this entire fix possible.

**Step two: split the `OR` into two independent queries.**

```python
# Before: one query with an OR
AND (Actor1Name = %s OR Actor2Name = %s)

# After: two queries, each hitting a single index, merged in application code
AND Actor1Name = %s   # query 1
AND Actor2Name = %s   # query 2, run separately
```

Instead of asking MySQL to reconcile two indexes in one pass, I ran two simple, single-index queries and combined the results myself. Each one alone is trivially fast; the combination is still faster than the merged version ever was.

## What the numbers actually looked like

This is the part of the fix I'd point to if I had to justify the whole exercise in one table:

| Metric | Before | After |
|---|---|---|
| `access_type` | `index_merge` | `index` |
| `key` | `sort_union(idx_actor1,idx_actor2)` | `idx_numarticles` |
| `rows_examined` | 549,348 | 1,209 (↓454x) |
| `using_filesort` | `true` | `false` |
| `query_cost` | 1,038,027 | 361,544 |
| Actual time | ~45s | ~0.3s (~150x) |

A 150x improvement from two changes, neither of which added a new index or touched the schema. The fix was entirely in how the query was shaped.

This same expression-sort pattern wasn't unique to `similar_events` — it showed up in five other places in the codebase (`geo events`, `hot_events`, `regional_overview`, and a couple of others), and all five got the same fix in a single pass, because every one of them lived in the shared query layer I described in the last post. That's the payoff of a single source of truth made concrete: fix the pattern once, and every query built on it improves at the same time.

---

## Two more bugs in the same neighborhood

Two other issues surfaced around the same part of the codebase, and both are worth a mention because neither one is obvious until you hit it.

**SRID mismatch.** Distance-based map queries use `ST_Distance_Sphere`, which failed outright with:

```
Binary geometry function st_distance_sphere given two geometries of different srids: 4326 and 0
```

Some rows had their geometry column stored with SRID 0 — meaning "no coordinate system specified" — instead of SRID 4326, the standard for GPS coordinates (WGS84). The fix was a one-time backfill:

```sql
UPDATE events_table
SET ActionGeo_Point = ST_GeomFromText(
    CONCAT('POINT(', ActionGeo_Lat, ' ', ActionGeo_Long, ')'), 4326)
WHERE ActionGeo_Point IS NOT NULL
  AND ST_SRID(ActionGeo_Point) = 0;
```

Just over a million rows needed correcting before the spatial index and distance calculations worked reliably.

**A leading wildcard that quietly disabled an entire index.** A location search like `ActionGeo_FullName LIKE '%Washington%'` looks harmless, but a leading `%` means MySQL can't use `idx_geo_fullname` at all — the match could start anywhere in the string, so the index is useless, and the query falls back to a full scan. It's the difference between jumping straight to the right page in a phone book and reading every entry because the name you want could be anywhere on the page. That query took about 86 seconds.

The fix was to stop asking for a substring match and instead expand the user's input into a set of index-friendly conditions:

```sql
-- Before: leading wildcard, can't use the index (86 seconds)
ActionGeo_FullName LIKE '%Washington%'

-- After: index-friendly combination (2-5 seconds)
   ActionGeo_FullName LIKE 'Washington%'        -- prefix match, uses idx_geo_fullname
OR ActionGeo_FullName LIKE '%, Washington%'     -- matches after a comma
OR ActionGeo_ADM1Code = 'US_DC'                 -- state code
OR ActionGeo_CountryCode = 'USA'                -- country code
```

This also became the entry point for handling location input more broadly — Chinese city and state names, U.S. state abbreviations, and country names in either language, all expanded into the same kind of prefix- and code-based conditions. Washington DC lookups went from 86 seconds to 2-5. Texas and Beijing queries saw the same order-of-magnitude improvement.

---

## The lesson that generalizes

If I had to compress this whole post into one sentence for anyone working with a large table: **`ORDER BY` on a computed expression and `OR` across multiple columns are the two most common ways to accidentally disable your indexes.** The first kills index-based sorting outright. The second forces the database into an expensive merge operation instead of a clean single-index lookup. Neither mistake looks dangerous when you write it — the query reads naturally, and it's easy to assume the indexes you built will just handle it.

Fixing the query shape got this specific table fast. It didn't make the platform fast everywhere — a slow query and a slow feature are different problems, and the next post covers the broader patterns (caching, parallelism, streaming, and pushing aggregation into the database) that handled the rest.
