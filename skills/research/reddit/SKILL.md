---
name: reddit
description: "Search and analyze the local offline Reddit corpus (posts, comments, subreddits) via PostgreSQL."
version: 1.0.0
author: M
license: private
platforms: [linux, macos]
metadata:
  hermes:
    tags: [Research, Reddit, Database, FTS, Postgres, Local]
    related_skills: []
---

# Reddit (Offline Database)

Search and analyze a large local archive of Reddit posts and comments stored
in Postgres database. No external API, no rate limits, no API key — direct read-only DB access!

Use this when the user asks for any **offline Reddit** research:
- "What are people saying about X in offline Reddit?"
- "Find discussions of X in offline Reddit r/foo"
- "What's the sentiment / consensus around X in offline Reddit?"
- etc.

**Don't** use this for **live** Reddit content — the local corpus is a
periodic snapshot, not real-time. For live posts, you must fall back to `web_search`
or a direct fetch.

Check the database connection with:
```bash
PGPASSWORD=${REDDIT_OFFLINE_DB_PASS_RO} psql -h ${REDDIT_OFFLINE_DB_HOST} -p ${REDDIT_OFFLINE_DB_PORT} -U ${REDDIT_OFFLINE_DB_USER_RO} -d reddit -c 'SELECT 1;'
```

Best way to interact with the db is by using bash `psql` or heredoc python.

## Full-text search

The `reddit_comments` table has a `body_tsv` tsvector column with a GIN index (`idx_comments_body_fts`).
**Generally use FTS instead of ILIKE for text search on comment bodies.**

Note: **ILIKE does a full sequential scan which takes minutes while FTS uses the index and returns in seconds.**

```sql
-- FTS (very fast search)
WHERE c.body_tsv @@ to_tsquery('english', 'meditation')

-- ILIKE (very slow)
WHERE c.body ILIKE '%meditation%'
```

The `to_tsquery` syntax supports `&` (AND), `|` (OR), `!` (NOT), and `<->` (adjacent words). Words are automatically stemmed (e.g. "agricultural" matches "agriculture").
Still, be mindful of FTS limitations. **There might be specific cases where it's better to use ILIKE for searches.**

FTS will miss things ILIKE catches:
- '%meditat%' via ILIKE matches "meditative", "meditator", "premeditated" as substrings. FTS for 'meditation' will match "meditative" and "meditator" via stemming, but won't match "premeditated" (different lexeme).
- '%meditat%' also catches words inside URLs or code like example.com/meditation-guide — FTS may tokenize these differently
- ILIKE can find stop words and partial fragments

FTS will find things ILIKE misses:
- `to_tsquery('english', 'meditation')` matches "meditation", "meditations", "meditating", "meditate", "meditated", "meditates" — all stemmed to the same lexeme. ILIKE '%meditation%' only catches "meditation" and "meditations" (words that contain the exact substring).

**Stop words.** `to_tsquery('english', 'the')` returns nothing useful; Postgres strips common stop words. Don't be surprised if a single short word returns zero hits.
**Multi-word phrases.** "machine learning" as two words is `machine & learning` (both stems present anywhere in the body). For true adjacency, use `machine <-> learning`.

## Big tables — think before you scan

`reddit_comments` is HUGE AMOUNT of rows. Any query that has to touch every row will sit there for minutes and burn DB resources. Before running ad-hoc SQL — including throwaway probes, sanity checks, and "just verifying" queries — ask yourself whether it can be answered without a full scan.

Specifically:
- **Never `SELECT count(*) FROM reddit_comments`** (or `reddit_posts`). Use `pg_class.reltuples` for an estimate, or scope the count with an indexed predicate.
- **To check column shape / permissions, use `LIMIT 0`** — `SELECT * FROM big_table LIMIT 0` resolves the column list without reading rows. `\d table` is even better.
- **For "does this row exist" checks, use `EXISTS` with an indexed predicate**, not `count(*) > 0`.
- **Unbounded `ILIKE` on `body`** is a sequential scan over the whole comments table. Prefer to use FTS (see above). You can use it, but use it smartly, where needed, knowing it will cost compute.
- **`EXPLAIN` first** if you're unsure — a `Seq Scan` on `reddit_comments` in the plan means stop and rethink.

The cost of pausing to think is a few seconds; the cost of a runaway scan is minutes of wasted time and a query that has to be killed.

## Quick Reference

### "Who's writing about X?"

```bash
PGPASSWORD=${REDDIT_OFFLINE_DB_PASS_RO} psql -h ${REDDIT_OFFLINE_DB_HOST} -p ${REDDIT_OFFLINE_DB_PORT} -U ${REDDIT_OFFLINE_DB_USER_RO} -d reddit -c "SELECT a.name, COUNT(*) AS n, SUM(c.score) AS total_score FROM reddit_comments c JOIN reddit_authors a ON a.id = c.author_id WHERE c.body_tsv @@ to_tsquery('english', 'rust & async') AND c.score > 5 GROUP BY a.name ORDER BY total_score DESC LIMIT 20;"
```

### "Show me the most active subreddits about X"

```bash
PGPASSWORD=${REDDIT_OFFLINE_DB_PASS_RO} psql -h ${REDDIT_OFFLINE_DB_HOST} -p ${REDDIT_OFFLINE_DB_PORT} -U ${REDDIT_OFFLINE_DB_USER_RO} -d reddit -c "SELECT s.name, COUNT(*) AS hits FROM reddit_comments c JOIN reddit_subreddits s ON s.id = c.subreddit_id WHERE c.body_tsv @@ to_tsquery('english', 'meditation & insomnia') GROUP BY s.name ORDER BY hits DESC LIMIT 25;"
```

### Using python

```python
import os, psycopg

with psycopg.connect(
    host=os.environ["REDDIT_OFFLINE_DB_HOST"],
    port=int(os.environ["REDDIT_OFFLINE_DB_PORT"]),
    dbname=os.environ["REDDIT_OFFLINE_DB_NAME"],
    user=os.environ["REDDIT_OFFLINE_DB_USER_RO"],
    password=os.environ["REDDIT_OFFLINE_DB_PASS_RO"],
) as conn, conn.cursor() as cur:
    cur.execute("SELECT 1;")
    print(cur.fetchone())
```
which you can write and run from `/tmp` for more complicated searches/analisys!

## Schema Cheat-Sheet

```
reddit_subreddits (id, name, title, description, public_description, status)
reddit_authors (id, name, …)
reddit_posts (id, subreddit_id, author_id, title, body, url, score, num_comments, created_utc, body_tsv, source)
reddit_comments (id, subreddit_id, author_id, body, score, parent_id, link_id, created_utc, body_tsv, source)
```

Both `posts` and `comments` have `body_tsv` (generated, indexed) and
`created_utc` as Unix seconds (integer, indexed). Subreddits join via
integer `subreddit_id`; you almost always start from a subreddit name and
resolve to id via `reddit_subreddits`.
