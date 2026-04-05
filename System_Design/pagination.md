# Pagination — The Art of Delivering Data in Chunks

You search for "shoes" on Amazon and get 50,000 results.

Does Amazon load all 50,000 products into your browser at once? Of course not. That would take minutes, use hundreds of megabytes of bandwidth, and crash your browser tab.

Instead, it shows you **24 products at a time** and gives you a "Next" button. You only load more when you want more.

That is **pagination**.

In API design, pagination means breaking a large dataset into smaller, manageable **pages** that are returned one at a time. The client requests a specific page, and the server returns only that portion of the data along with metadata about how to get the next portion.

It sounds simple, but choosing the wrong pagination strategy is one of the most common mistakes in API design — leading to duplicate records, missing data, slow queries, and terrible user experiences at scale.

---

## Table of Contents

1. [Why You Need Pagination](#1-why-you-need-pagination)
2. [Pagination Strategies Overview](#2-pagination-strategies-overview)
3. [Offset-Based Pagination](#3-offset-based-pagination)
4. [Cursor-Based Pagination](#4-cursor-based-pagination)
5. [Keyset Pagination](#5-keyset-pagination)
6. [Page Number Pagination](#6-page-number-pagination)
7. [Comparison Table](#7-comparison-table)
8. [Pagination at Scale — Real-World Patterns](#8-pagination-at-scale--real-world-patterns)
9. [Senior Interview Questions](#9-senior-interview-questions)
10. [Summary](#10-summary)

---

## 1. Why You Need Pagination

### Without Pagination

```
Client: GET /api/products/
Server: Here are ALL 2,000,000 products in JSON...

Response size: ~800 MB
Time to serialize: 45 seconds
Time to transfer: 2 minutes
Client memory: Browser tab crashes
Database: Full table scan, locks, OOM
```

### With Pagination

```
Client: GET /api/products/?page=1&page_size=20
Server: Here are 20 products and metadata

Response size: ~8 KB
Time to serialize: 2 ms
Time to transfer: 50 ms
Client memory: Negligible
Database: Indexed query, fast
```

### The Real Costs of No Pagination

| Problem | Impact |
|---|---|
| **Memory exhaustion** | Server loads millions of rows into memory → OOM kill |
| **Slow serialization** | Converting millions of objects to JSON takes seconds |
| **Network bandwidth** | Huge responses clog the network, increase costs |
| **Database strain** | Full table scans lock tables, block other queries |
| **Client crashes** | Browsers cannot render 100,000 DOM elements |
| **Poor UX** | Users wait forever for data they will never scroll through |

Every production API **must** paginate any endpoint that returns a list of resources.

![Pagination Overview](/System_Design/images/pagination_overview.svg)

---

## 2. Pagination Strategies Overview

There are four main approaches, each with different tradeoffs:

| Strategy | How It Works | Best For | Weakness |
|---|---|---|---|
| **Offset-Based** | `OFFSET 40 LIMIT 20` | Simple UIs, small datasets | Slow at high offsets, data drift |
| **Cursor-Based** | `WHERE id > last_seen_id` | Infinite scroll, real-time feeds | No random page access |
| **Keyset** | `WHERE (col1, col2) > (val1, val2)` | Large sorted datasets | Complex with multi-column sorts |
| **Page Number** | `?page=3&size=20` | Traditional web UIs | Same issues as offset under the hood |

![Pagination Strategies](/System_Design/images/pagination_strategies.svg)

---

## 3. Offset-Based Pagination

### How It Works

Offset pagination uses two parameters: an **offset** (how many rows to skip from the beginning) and a **limit** (how many rows to return). The database executes something like `SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40` to fetch page 3.

### The Bookshelf Analogy

Imagine you have a bookshelf with 10,000 books arranged by title. Someone asks, "Give me books 5001 to 5020."

You have to **count past the first 5000 books one by one** before you can start picking the 20 books they want. You don't read them — you just count past them. That's wasted effort, and it gets worse the further you go.

That is exactly what `OFFSET 5000` does in a database. The database scans and discards 5000 rows before returning the 20 you actually want. The performance is `O(offset + limit)` — it degrades linearly as the user goes deeper into the dataset.

### The Data Drift Problem

Offset pagination suffers from **data drift** — when records are inserted or deleted between page requests. Imagine a user fetches page 1 containing items [A, B, C, D]. Before they request page 2, item B is deleted. The remaining items shift left, so offset=4 now skips past item E entirely. The user never sees it.

In a social feed, this means **missed posts**. In an e-commerce listing, this means **invisible products**. The problem gets worse with high write throughput.

### When to Use

- Small to medium datasets (< 100,000 rows)
- Admin panels, dashboards, internal tools
- UIs where "Jump to page 47" functionality is important
- Data that rarely changes during browsing

### When to Avoid

- Large datasets (millions of rows) — performance degrades at depth
- Real-time feeds where data changes frequently — data drift causes missed items
- Infinite scroll UIs — cursor pagination is far more natural and efficient

---

## 4. Cursor-Based Pagination

### How It Works

Instead of saying "skip N rows," cursor pagination says "give me rows **after this specific item**." The cursor is an **opaque token** — typically a base64-encoded representation of the last item's sort key — that uniquely identifies a position in the result set.

The database uses a `WHERE id > last_seen_id ORDER BY id LIMIT 20` query, which leverages the **B-tree index** to jump directly to the right position. No scanning, no counting. Whether you're on page 2 or page 50,000, the performance is identical: `O(limit)`.

### The Bookmark Analogy

Instead of telling the librarian "give me books starting from position 5001" (which requires counting), you leave a **bookmark** at the last book you read.

Next time, you say: "Give me the next 20 books after this bookmark." The librarian goes directly to the bookmark and grabs the next 20 — no counting, no wasted effort. Doesn't matter if the library added or removed books earlier in the shelf — your bookmark still points to exactly the right spot.

### Why No Data Drift

With cursor pagination, if item B is deleted before the user requests the next page, it doesn't matter. The cursor points to item D **by identity**, not by position. The next page starts from wherever D is, and item E is correctly included. No items are ever skipped or duplicated.

### Cursor Encoding

Cursors should be **opaque** to the client — encoded as base64 strings that the client cannot and should not parse. This gives the server freedom to change the internal cursor format (e.g., switching from single-column to multi-column cursors) without breaking any clients. The API contract is simply: "send this token back to get the next page."

### Tradeoffs

Cursor pagination cannot support "jump to page N" — users can only move forward (or backward) one page at a time. It also makes total count expensive since there's no natural way to know how many items exist without a full table scan. For these reasons, cursor pagination is ideal for **forward-only traversal** like feeds, timelines, and infinite scroll, but not for traditional page-number UIs.

### When to Use

- Large datasets (millions of rows)
- Infinite scroll UIs (social feeds, timelines, chat history)
- Real-time data with frequent inserts/deletes
- Mobile apps where bandwidth and consistency matter

---

## 5. Keyset Pagination

### How It Works

Keyset pagination is a generalized form of cursor pagination that works with **multi-column sort orders**. Instead of just `WHERE id > X`, it uses a tuple comparison: `WHERE (created_at, id) < ('2025-03-15', 4532)`.

This is necessary when sorting by a non-unique column like `created_at`. If 50 products share the same timestamp, a simple `WHERE created_at < X` would either skip all 50 or include all 50 again. The solution is a **tiebreaker** — adding a unique column (like `id`) as the last sort component. The tuple comparison `(created_at, id)` uniquely identifies every row's position in the sorted result.

### Performance

Keyset pagination requires a **composite index** matching the sort order — for example, `(created_at DESC, id DESC)`. With this index, performance is `O(limit)` regardless of depth, identical to simple cursor pagination. Without the matching index, the database falls back to a full scan.

### When to Use

- When sorting by non-unique columns (timestamps, scores, names)
- Time-series data, event logs, activity feeds
- Any cursor-based scenario that needs multi-column ordering

---

## 6. Page Number Pagination

### How It Works

The most intuitive form of pagination — users request a page number like `?page=3&page_size=20`. Under the hood, the server simply converts this to an offset: `offset = (page - 1) * page_size = 40`.

Page number pagination has the **same performance characteristics as offset pagination** — it's just a friendlier interface over the same `OFFSET` mechanism. This means it inherits the same problems: `O(offset + limit)` at depth and data drift.

The response typically includes rich metadata: total count, total pages, current page number, and next/previous links. This metadata makes it easy to build traditional page controls with numbered buttons.

### When to Use

- Traditional web applications with numbered page buttons
- Admin panels, search results pages (Google-style "Page 1 2 3 ... 10")
- When users need to jump to specific pages
- Small to medium datasets where depth performance is not a concern

---

## 7. Comparison Table

### Pagination Strategy Comparison

| Criteria | Offset | Cursor | Keyset | Page Number |
|---|---|---|---|---|
| **Performance at depth** | O(offset+limit) — degrades | O(limit) — constant | O(limit) — constant | O(offset+limit) — degrades |
| **Random page access** | Yes | No | No | Yes |
| **Data consistency** | Drift risk | Stable | Stable | Drift risk |
| **Total count** | Easy (`COUNT(*)`) | Expensive | Expensive | Easy (`COUNT(*)`) |
| **Implementation** | Simple | Medium | Complex | Simple |
| **Infinite scroll** | Poor | Excellent | Excellent | Poor |
| **Sort flexibility** | Any column | Single column | Multi-column | Any column |
| **Used by** | Shopify, GitHub v3 | Slack, Twitter | Stripe | Google Search, Bing |

### Framework Support

| Feature | Django (DRF) | FastAPI | Go |
|---|---|---|---|
| **Built-in pagination** | Yes — 3 classes (PageNumber, LimitOffset, Cursor) | No — build manually with Pydantic + Query params | No — build manually with generics |
| **Setup effort** | 3 lines of config | ~50 lines | ~80 lines |
| **Async support** | Limited (Django 4.1+) | Native | Native (goroutines) |

---

## 8. Pagination at Scale — Real-World Patterns

### Hybrid Pagination

Use page numbers for early pages (1–100) and automatically switch to cursor-based pagination for deeper access. This gives users a familiar page UI for browsing while preventing the O(offset) performance cliff. If a client requests page 101+, redirect them to cursor pagination or return an error with a cursor link.

### Estimated Total Count

Running `COUNT(*)` on a table with millions of rows is expensive — it can take seconds. Instead of an exact count, use PostgreSQL's **estimated row count** from `pg_class.reltuples`. It's within ~10% accuracy and returns instantly. For most UIs, "About 1.5M results" is good enough. Alternatively, cache the exact count with a short TTL and refresh it periodically.

### Link Headers (RFC 8288)

Instead of embedding pagination metadata in the response body, use HTTP `Link` headers. This is the approach GitHub's API uses:

```
Link: <https://api.example.com/products?page=4>; rel="next",
      <https://api.example.com/products?page=2>; rel="prev",
      <https://api.example.com/products?page=1>; rel="first",
      <https://api.example.com/products?page=762>; rel="last"
```

This keeps the response body clean — it contains only the actual data — and follows a well-established web standard.

### Enforcing Limits

Always enforce a **maximum page size** (e.g., 100). Without it, a client can request `page_size=10000000` and crash your server. Always require an `ORDER BY` clause — without deterministic ordering, the same query can return different rows each time, causing duplicates and missing items. And always make cursors **opaque** — if clients can parse and construct cursors, they become part of your API contract and you can never change the internal format.

---

## 9. Senior Interview Questions

### Question 1: You're designing a social media feed API that serves 10 million users. Posts are created at ~50,000/second. Which pagination strategy would you choose and why?

**Answer:**

I would use **cursor-based pagination** with a keyset approach for several reasons:

**Why not offset?** At 50K posts/second, the data is constantly changing. Offset pagination would suffer severe **data drift** — users would miss posts or see duplicates as new content pushes items between pages. Additionally, popular users would paginate deep into their feeds, and offset performance degrades to O(offset + limit), potentially causing multi-second queries.

**Why cursor?** Cursor pagination solves both problems. The cursor points to a specific post by identity (not position), so insertions and deletions don't cause drift. Performance is O(limit) regardless of depth — page 50,000 is the same speed as page 1 because the B-tree index seek goes directly to the cursor position.

**Design details:** I would use a compound cursor encoding `(created_at, post_id)` to handle timestamp collisions. The `post_id` serves as a tiebreaker since multiple posts can share the same `created_at` value. The cursor would be base64-encoded and opaque to the client.

**Index:** A composite index on `(user_id, created_at DESC, post_id DESC)` would support the query `WHERE user_id = ? AND (created_at, post_id) < (?, ?)` efficiently.

**Total count:** I would not provide it. Total count requires a full table scan which is unacceptable at this scale. If the product requires an approximate count, I'd use a denormalized counter or PostgreSQL's `pg_class.reltuples`.

**Response structure:**
```json
{
    "posts": [...],
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNS0wMy0xNVQxMDozMDowMFoiLCJpZCI6NDUzMn0=",
    "has_more": true
}
```

The client sends the `next_cursor` value on the subsequent request. The server decodes it, applies the keyset WHERE clause, and returns the next batch.

---

### Question 2: Your API uses offset pagination and a customer reports that they're seeing duplicate items when paginating through results. What's happening and how do you fix it?

**Answer:**

This is the classic **data drift problem** with offset pagination. Here's what's happening:

**Root cause:** Between the time the customer fetches page N and page N+1, new records are inserted (or existing ones are deleted or re-ordered) that shift the position of items. For example:

- Page 1 returns items at positions 1–20
- A new item is inserted at position 5 (perhaps due to sorting by `updated_at`)
- All items from position 5 onward shift by +1
- Page 2 (offset=20) now starts at what was originally position 19
- Item 20 from page 1 now sits at position 21, so it appears again on page 2

**How to reproduce:** This is most visible on write-heavy tables sorted by a mutable column (like `updated_at`). It also occurs on tables sorted by `created_at DESC` when new records are frequently inserted.

**Short-term fixes:**
1. **Sort by an immutable, sequential column** — if you sort by `id ASC`, new inserts always go to the end and don't shift existing items. Deletions can still cause skips but not duplicates.
2. **Snapshot isolation** — use a database transaction with `REPEATABLE READ` isolation level so the client sees a consistent snapshot across all page requests. However, this holds resources and doesn't scale well for long browsing sessions.

**Long-term fix:** **Migrate to cursor-based pagination.** Cursors identify items by value (e.g., `id > 42`), not by position. Insertions and deletions do not affect the cursor's meaning. The same item can never appear twice because the cursor condition guarantees forward-only traversal past items already seen.

**Migration strategy:** Support both strategies simultaneously during a transition period. Add a `cursor` query parameter alongside the existing `page` parameter. When `cursor` is provided, use cursor pagination; otherwise, fall back to offset. Deprecate the offset path in the API documentation and eventually remove it after clients have migrated.

---

### Question 3: An interviewer asks you to design pagination for a search API (like Elasticsearch) where the result set is dynamic and scored by relevance. How does this change the pagination strategy?

**Answer:**

Search pagination is fundamentally different from database pagination because the result set is **computed, scored, and ranked at query time** — it's not a stable, ordered table.

**Why standard cursors break:** In a search engine, the "same" query can return different results moments apart because:
- New documents are indexed
- Relevance scores are recalculated
- Index segments are merged (changing internal doc ordering)

A cursor pointing to "the document with score 4.7" is meaningless if scores change on the next query execution.

**The search engine approach — Scroll/PIT (Point in Time):**

Elasticsearch and similar engines solve this with a **snapshot mechanism**. When the first search request arrives, the engine creates a **point-in-time snapshot** of the index. All subsequent pagination requests operate against that frozen snapshot, guaranteeing consistent results regardless of ongoing writes. The snapshot has a TTL (e.g., 5 minutes) and is automatically cleaned up.

This is conceptually similar to database `REPEATABLE READ` isolation but at the search index level.

**The `search_after` approach:**

For real-time search where you want to see the latest results, Elasticsearch offers `search_after` — which is essentially keyset pagination using the sort values of the last returned document. You provide the last document's sort values, and the engine returns documents that come after that position. This handles deep pagination efficiently without the O(offset) problem.

**Practical design decisions:**

1. **Cap the maximum depth.** Google shows at most ~30 pages of search results. Beyond that, the user should refine their query. I'd return an error or a "refine your search" message after page 100.

2. **Use `search_after` for real-time feeds** where freshness matters more than snapshot consistency.

3. **Use scroll/PIT for export or analytics** where the user needs to traverse the entire result set consistently (e.g., "export all matching records to CSV").

4. **Never use offset (`from/size`) for deep pagination** in Elasticsearch. It requires the coordinating node to gather and re-sort `from + size` results from every shard, leading to massive memory pressure and slow responses.

**Key tradeoff to articulate:** In search, you must choose between **consistency** (scroll/PIT — frozen snapshot, may miss new documents) and **freshness** (`search_after` — always latest results, but scores may shift between pages). The right choice depends on whether the user is browsing interactively or performing a bulk operation.

---

### Question 4: A product manager asks you to add a "total results" count to every paginated API response. The largest table has 200 million rows. How do you handle this?

**Answer:**

The naive approach — running `SELECT COUNT(*) FROM table WHERE ...` on every request — is a serious problem at this scale. In PostgreSQL, `COUNT(*)` requires a **sequential scan** of all matching rows because MVCC means different transactions may see different row visibility. On a 200M-row table, this can take 5–30 seconds and consume significant I/O, effectively doubling the cost of every paginated request.

**Strategy 1: Don't return an exact count.** The first question I'd push back to the PM with is: "Does the user actually need to know there are exactly 198,437,291 results, or is 'About 198 million' good enough?" Most UIs don't need precision. Google Search says "About 1,230,000,000 results" — that's an estimate.

For PostgreSQL, you can get a fast estimate from the planner:

```sql
SELECT reltuples::bigint FROM pg_class WHERE relname = 'products';
```

This returns instantly and is updated whenever `ANALYZE` runs (which autovacuum does regularly). For filtered queries, you can use `EXPLAIN` output to extract the planner's row estimate.

**Strategy 2: Cache the count.** Compute the exact count once, cache it in Redis with a short TTL (30–60 seconds), and serve the cached value on subsequent requests. The count only changes meaningfully over minutes, not milliseconds. Invalidate the cache on bulk inserts/deletes.

**Strategy 3: Denormalize.** Maintain a separate counter table that is updated via triggers or application-level logic whenever rows are inserted or deleted. This gives O(1) reads but adds complexity to write paths and requires careful handling of transactions and rollbacks.

**Strategy 4: Use cursor pagination and drop the count entirely.** Cursor pagination naturally doesn't need a total count — it returns `has_more: true/false`. The UI shows a "Load more" button instead of "Page 3 of 762." This is the approach Twitter, Slack, and most infinite-scroll UIs use. I would recommend this to the PM as the best long-term solution.

**What I'd recommend:** Start with Strategy 4 (no count) for the API. If the PM insists, use Strategy 2 (cached count) with a "this is approximate" disclaimer in the response. Never run `COUNT(*)` synchronously on every request against a 200M-row table.

---

### Question 5: You have a multi-tenant SaaS platform where Tenant A has 500 rows and Tenant B has 50 million rows. How do you design a single pagination API that works well for both?

**Answer:**

This is a classic problem where a one-size-fits-all approach will either over-engineer for small tenants or break for large ones. The key insight is that the **strategy can adapt based on the data characteristics** without exposing this complexity to the client.

**API contract — keep it uniform:** The external API looks the same for both tenants. The client sends `?page_size=20&cursor=abc` or `?page=3&page_size=20`. The response always includes `results`, `has_more`, and optionally `total_count`.

**Internal strategy — adaptive:**

For Tenant A (500 rows): Offset pagination is perfectly fine. `OFFSET 400 LIMIT 20` on 500 rows takes microseconds. `COUNT(*)` is instant. Page number navigation with "Page 3 of 25" is a great UX.

For Tenant B (50M rows): Offset would be catastrophic at depth. Cursor pagination is mandatory. `COUNT(*)` takes seconds and should be estimated or cached.

**How to decide at runtime:**

Option 1 — **Threshold-based switching:** Maintain an approximate row count per tenant (updated daily or via triggers). If a tenant has fewer than 100,000 rows, use offset with exact count. Above that threshold, switch to cursor pagination with estimated count or no count. The API response format stays the same — the client doesn't need to know which strategy is used internally.

Option 2 — **Always use cursor, optimize count:** Use cursor pagination for all tenants (it works fine on small datasets too — there's no performance penalty). For the count, small tenants get an exact count (it's fast), while large tenants get a cached or estimated count. This is simpler because you only have one pagination code path.

**What I'd recommend:** Option 2 — always use cursor pagination internally. It's correct at every scale. The only variation is the count strategy. This avoids maintaining two pagination implementations and eliminates an entire class of bugs where a tenant grows past the threshold and suddenly experiences degraded performance.

**Additional considerations:**
- **Rate limiting per tenant** — Tenant B paginating through 50M rows at high speed could overload the database. Implement per-tenant rate limits on pagination endpoints.
- **Partition the data** — if tenants are fully isolated, consider table partitioning by `tenant_id` so that queries only scan the relevant partition, making counts and offsets cheaper even for large tenants.
- **Max page depth** — regardless of strategy, cap how deep a client can paginate (e.g., 10,000 items). If they need bulk access, provide a separate export/streaming API.

---

### Question 6: How would you implement backward pagination (navigating to previous pages) with cursor-based pagination? Isn't cursor pagination forward-only?

**Answer:**

This is a nuanced question because cursor pagination is often described as "forward-only," but that's a simplification. **Bidirectional cursor pagination is absolutely possible** — it just requires careful design.

**How forward cursors work:** A forward cursor encodes the last item's sort values. The query uses `WHERE id > cursor_id ORDER BY id ASC LIMIT 20` to fetch the next page.

**How backward cursors work:** A backward cursor encodes the first item's sort values from the current page. The query uses `WHERE id < cursor_id ORDER BY id DESC LIMIT 20`, then **reverses the results** before returning them to the client so they appear in the correct ascending order.

**The API design:**

The response includes both `next_cursor` and `prev_cursor`:

```json
{
    "results": [...],
    "next_cursor": "eyJpZCI6NDB9",
    "prev_cursor": "eyJpZCI6MjF9",
    "has_next": true,
    "has_prev": true
}
```

The client sends either `?after=eyJpZCI6NDB9` for the next page or `?before=eyJpZCI6MjF9` for the previous page.

**Server logic:**
- If `after` is provided: `WHERE id > decoded_id ORDER BY id ASC LIMIT page_size`
- If `before` is provided: `WHERE id < decoded_id ORDER BY id DESC LIMIT page_size`, then reverse the result array

**Why this works at O(limit):** Both directions use an index seek — forward seeks right in the B-tree, backward seeks left. Neither requires scanning or counting. Performance is identical in both directions.

**The tricky part — first and last page detection:** To know if a previous page exists (for the first page, it doesn't), you can fetch `page_size + 1` items in the backward direction and check if the extra item exists. Same technique used for detecting `has_next` in forward pagination.

**Real-world usage:** Slack's message API works exactly this way — you can paginate forward through message history (`?after=`) or backward (`?before=`). Facebook's Graph API also supports bidirectional cursors with `before` and `after` parameters.

**What you cannot do:** Even with bidirectional cursors, you still cannot "jump to page N." You can only move one page forward or backward from your current position. If the product requires random page access, you need either page-number pagination (with its offset tradeoffs) or a hybrid approach.

---

## 10. Summary

| Concept | Key Takeaway |
|---|---|
| **Why paginate** | Without it, large responses crash servers, clog networks, and destroy UX |
| **Offset pagination** | Simple but O(offset) at depth and suffers data drift — use for small datasets |
| **Cursor pagination** | O(limit) at any depth, no drift — the right choice for feeds and large datasets |
| **Keyset pagination** | Generalized cursor for multi-column sorts — needs a composite index and a tiebreaker |
| **Page number** | Most intuitive for users — same offset performance under the hood |
| **At scale** | Cache or estimate counts, enforce max page sizes, make cursors opaque |
| **Search pagination** | Different beast — use scroll/PIT for consistency, `search_after` for freshness |

> **The golden rule:** If someone can scroll past page 100, you need cursor pagination. If they can't, page numbers are fine.
