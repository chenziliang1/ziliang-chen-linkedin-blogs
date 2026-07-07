# Turning 15 Million Rows Into a Queryable Database

The performance of this entire platform rests on one table. Get it wrong, and a query that should take a fraction of a second takes 45. Get it right, and the same query runs in the time it takes to blink. This post is about how I got it right — or at least, how I got it right the second time.

## What's actually in a row

The core table, `events_table`, holds around 15 million rows of 2024 North American GDELT events — the queryable table that Dashboard, Forecast, and Chat all read from. It's a smaller number than the roughly 17 million events I mentioned in the first post, because that figure includes additional processing batches beyond what lives in this one table. Each row is a single extracted event, with fields covering:

- **Time** — `SQLDATE`
- **Participants** — `Actor1Name`, `Actor2Name`
- **Event semantics** — `EventCode`, `EventRootCode`, `QuadClass`
- **Intensity and sentiment** — `GoldsteinScale` (a conflict/cooperation score) and `AvgTone` (media sentiment)
- **Coverage volume** — `NumArticles`
- **Geography** — `ActionGeo_Lat`, `ActionGeo_Long`, `ActionGeo_FullName`, a geometry column `ActionGeo_Point`, and `ActionGeo_CountryCode`

At this scale — 15,236,852 rows, to be exact, backed by 16 total indexes — small mistakes stop being small. A poorly written `ORDER BY` clause on a table like this doesn't cost you a few milliseconds. It can be the difference between a query that returns instantly and one that takes 45 seconds. I'll walk through that exact case in the next post. This one is about the two things that have to be right before you even get to that point: how the data gets in, and how it gets indexed.

## Data first, indexes second — always

There's a strict order to setting this database up, and it's not optional:

```bash
# 1. Create the schema (once)
docker exec -i gdelt_mysql mysql -u root -prootpassword < db_scripts/gdelt_db_v1.sql

# 2. Import the CSV data (before anything else)
python db_scripts/import_event.py

# 3. Build the base indexes (only after data is loaded)
docker exec -i gdelt_mysql mysql -u root -prootpassword gdelt < db_scripts/add_indexes.sql

# 4. Fix spatial data (needed before using the spatial index)
```

The rule that matters most here is simple: **import the data before you build the indexes, not the other way around.** Building an index on an empty table accomplishes nothing, and doing it before the bulk import actually slows the import down — every insert has to update the index as it goes, instead of the index being built once against a finished dataset.

The other rule I stuck to: none of the raw data goes into git. The GDELT CSVs and the MySQL data volume live outside the repository entirely. They get distributed through the import scripts themselves or a separate database dump — the `data/` directory in the repo holds nothing but a placeholder. And after each import, I checked the basics before trusting the data was actually usable: total row count, whether `SQLDATE` was populated, whether the geographic coordinates were null.

## Sixteen indexes, each earning its place

Once the data was in, the real design work started. An index isn't free — it costs storage and slows down writes — so each one needed to correspond to an actual query pattern rather than existing just in case. The final baseline landed on 16:

```sql
idx_sqldate         (SQLDATE)                        -- date range filters, used everywhere
idx_actor1          (Actor1Name(20))                 -- primary actor lookups
idx_actor2          (Actor2Name(20))                 -- secondary actor lookups
idx_goldstein       (GoldsteinScale)                 -- conflict/cooperation filtering
idx_lat / idx_long  (ActionGeo_Lat / ActionGeo_Long) -- geographic coordinates
idx_date_geo        (SQLDATE, ActionGeo_Lat, ActionGeo_Long)  -- time + location, combined
idx_date_actor      (SQLDATE, Actor1Name(20))        -- time + actor, combined
idx_geo_fullname    (ActionGeo_FullName(100))        -- city/region name, prefix indexed
idx_event_root      (EventRootCode)                  -- event type
idx_date_country    (SQLDATE, ActionGeo_CountryCode)
idx_numarticles     (NumArticles)                    -- turned out to matter more than expected
idx_geo_point       (ActionGeo_Point) SPATIAL         -- for ST_Distance_Sphere queries
```

Two design choices did most of the work here. The first is prefix indexing: `Actor1Name(20)` and `ActionGeo_FullName(100)` only index the first N characters of the field instead of the whole thing, which keeps the index size manageable — around 2-3GB across all 16 indexes for 15.89 million rows — without giving up much match accuracy. The second is composite indexing: `idx_date_actor` and `idx_date_geo` combine two columns in one index, which lets certain queries get answered straight from the index itself, without touching the underlying table row at all.

One index in that list, `idx_numarticles`, looked unremarkable when I added it. It turned out to be the single most important index in the whole schema — not for filtering, but for sorting. That's the subject of the next post.

## What building the right index actually buys you

The numbers make the case better than any explanation could:

| Query type | Before indexing | After indexing | Improvement |
|---|---|---|---|
| Time range (1 year) | ~60s | ~0.5s | 120x |
| Actor lookup | ~30s | ~1s | 30x |
| Location lookup | ~45s | ~2s | 22x |
| Geographic radius search | timeout | ~2s | ∞ |

That last row is worth sitting with. A geographic radius search on this table didn't just get slow without a spatial index — it never finished at all.

---

Getting the schema and the indexing right made most queries fast. It didn't make all of them fast. One particular query kept coming back at 45 seconds no matter how reasonable the indexes underneath it looked, and the reason turned out to have nothing to do with which indexes existed — it was about how the query itself was written. That's the case study in the next post, and it's the single most instructive bug I hit on this entire project.
