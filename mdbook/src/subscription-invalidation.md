# How Subscription Invalidation works

### Problem

The server needs a way to figure out if a subscription (live query) is invalidated by a recent change and needs to be refreshed.

For example, let’s say there’s a live query that looks like this:

```go
 .query("players")
      .withIndex("by_points", (q) => q.gt("points", 20))
      .collect();
```

Then when someone inserts a new document whose `points:30` then the query should be invalidated. On the other hand, if there’s a new document whose `points:19` then the query doesn’t need to be invalidated.

### What is invalidation?

First, let’s explain exactly what invalidation means in Convex’s system.

Convex’s architecture is built around a single append-only log, containing all the mutations to the database, where the logs have monotonically increasing `commit_ts`.

Each live subscription is known to be valid up to a specific `commit_ts`. Let’s call this the `processed_ts`. In order to keep the subscription result fresh, the server needs to “advance” its `processed_ts` to the `latest_commit_ts`. However, if one of the commits between `(processed_ts, latest_commit_ts]` contains a change that can alter the query’s output, the query must be refreshed, otherwise it’s inconsistent with the latest writes.

The process of detecting whether it’s safe to advance the `processed_ts` to a higher `commit_ts` is known as invalidation.

Conceptually, it’s a simple function that takes the recent commits and the previous query result and returns a boolean: `is_valid(recent_commits, previous_query_result)`.

### What makes a good invalidation implementation?

Invalidation is judged on:

- how many false positives the implementation generates
- how much memory the implementation requires

The invalidation function doesn’t need to be perfect. False positives are allowed; false negatives are not. The cost of a false positive output is an unnecessary query refresh. An invalidation function that always invalidates is technically “correct” from the user’s perspective. As a result, one of the measures of a good invalidation method is how infrequent false positives are.

The second measure of a good invalidation method is memory footprint of the `recent_commits` and `previous_query_result`. The more compact representation, the better.

### Incorrect Attempt: Tracking document IDs

A naive approach would be to just track all the document IDs returned by the query. When checking if another transaction invalidates the query, see if the query modified any of the documents in the list of tracked document IDs.

However, that is not correct.

Let’s look at a query: `points>12`.

Let’s say the query returned two documents: `{ name: alice, points: 20 }` and `{ name: bob, points: 30 }`, with tracking IDs  `[alice, bob]`.

Let’s say someone added `{ name: carol, points: 40}` into the database. It doesn’t conflict with the document ID list. However, it would cause the query output to change.

This is because tracking document IDs does not keep the intent of the query, which is all documents greater than 40.

### Tracking Ranges

In Convex, a query can only read documents by walking an index. `.withIndex("by_points", q => q.gt("points", 20))` is a scan of the `by_points` index over the range `(20, +∞)`.

Even a plain `.query("players")` with no explicit index is a scan of the built-in `by_creation_time` index over its entire range, and `db.get(id)` is a scan of the `by_id` index over the single key `id`. There is no code path that reads a document without going through an index range.

A method like: `.order("desc").take(3)` doesn't start at 1000. It starts at `+∞` and walks downward through the index until it has three rows. So the range is something like `[150, +∞)`.

A query like `q.eq("team", "red")` is `["red", succ(enc("red"))` where `succ` is the next lexicographical.

This is the constraint that makes the whole design work. It means that the set of documents a query *could* have observed is fully described by a set of `(index, range)` pairs. If we record those pairs while the query runs, we have a complete description of the query's reads that is independent of how many documents were returned.

As a result, `previous_query_result` can be represented as `processed_ts` + a list of interval ranges representing indexes it touched.

### Representing Commits

Now the other side. A commit is a set of document writes. For invalidation purposes, we don't care about the document contents, only about which index keys the write touched. Each write is converted into, per index, an `Update { old, new }` of index keys.

**Example**

A single `db.patch` produces an update on every single index of the table.

Let’s assume we have the following database schema:

```go
players: defineTable({
  name: v.string(),
  team: v.string(),
  points: v.number(),
})
  .index("by_points", ["points"])
  .index("by_team_points", ["team", "points"]),
```

Convex adds two indexes of its own to every table, so there are four in total. `by_id` has no indexed fields, so its key is just `(_id)`; `by_creation_time` is on `[_creationTime]`, so its key is `(_creationTime, _id)`.

Start with Dave, `{_id: d7, _creationTime: 1000, name: "Dave", team: "red", points: 40}`, and run `ctx.db.patch(d7, { points: 45 })`. The commit produces four updates:

| Index              | Key layout             | `old`             | `new`             |
| ------------------ | ---------------------- | ----------------- | ----------------- |
| `by_id`            | `(_id)`                | `(d7)`            | `(d7)`            |
| `by_creation_time` | `(_creationTime, _id)` | `(1000, d7)`      | `(1000, d7)`      |
| `by_points`        | `(points, _id)`        | `(40, d7)`        | `(45, d7)`        |
| `by_team_points`   | `(team, points, _id)`  | `("red", 40, d7)` | `("red", 45, d7)` |

The entrypoint for converting a document update to the compact representation is: `index_keys_from_full_documents`.

### Putting it altogether: is_valid(recent_commits, previous_query_result)

It basically iterates over:

- all the index key ranges that the previous query result scanned
  - all recent commits between `(processed_commit_ts, latest_commit_ts]`
    - all the index keys from the mutation

### Search Indexes

So far, we only talked about normal indexes, not search indexes.

**How text search is represented in read set**

In general, you define a search index as follows:

```go
.searchIndex("by_body", {
    searchField: "body",
    filterFields: ["channel", "author"],
  }),
```

When a search runs, Convex records two lists under the index's name. The first list is the query's **terms**: the search string lowercased and split into words. The second is its **filters**: one entry per `.eq`, holding the field and the value it was compared to

| Query                                                        | Terms                                           | Filters                           |
| ------------------------------------------------------------ | ----------------------------------------------- | --------------------------------- |
| `q.search("body", "hello")`                                  | `"hello"` (prefix)                              | —                                 |
| `q.search("body", "hello world")`                            | `"hello"` (exact), `"world"` (prefix)           | —                                 |
| `q.search("body", "Hello WORLD")`                            | `"hello"` (exact), `"world"` (prefix)           | —                                 |
| `q.search("body", "hello").eq("channel", c1)`                | `"hello"` (prefix)                              | `channel = c1`                    |
| `q.search("body", "ship the release").eq("channel", c1).eq("author", "dave")` | `"ship"`, `"the"` (exact), `"release"` (prefix) | `channel = c1`, `author = "dave"` |
| `q.search("body", "!!!")`                                    | —                                               | —                                 |

Note that the read set for a search query is not tied to the data returned. It’s predetermined solely from the query, unlike normal index scans.

### Representing commits for search indexes

For each patch, the write log entry contains, for each search index, all the terms for that search index as well as the filtered keys.

| Document                                                     | Projection into `by_body`                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `{body: "Hello world", channel: c1, author: "dave", pinned: false}` | tokens `{hello, world}`, filters `{channel: c1, author: "dave"}` |
| `{body: "hello hello hello", channel: c2, author: "eve", pinned: true}` | tokens `{hello}`, filters `{channel: c2, author: "eve"}`     |
| `{body: "", channel: c1, author: "dave", pinned: false}`     | tokens `{}`, filters `{channel: c1, author: "dave"}`         |
| `{channel: c1, author: "dave", pinned: false}` (no `body`)   | no tokens, filters `{channel: c1, author: "dave"}`           |

Let’s say someone patches:

```
await ctx.db.patch(m3, { body: "goodbye world" });
```

This produces one entry per index:

| Index              | `old`                                                        | `new`                                                        |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `by_id`            | `(m3)`                                                       | `(m3)`                                                       |
| `by_creation_time` | `(2050, m3)`                                                 | `(2050, m3)`                                                 |
| `by_channel`       | `(c1, m3)`                                                   | `(c1, m3)`                                                   |
| `by_body` (text)   | tokens `{hello, world}`, filters `{channel: c1, author: "dave"}` | tokens `{goodbye, world}`, filters `{channel: c1, author: "dave"}` |

### Determining conflict for search indexes

Two steps, in this order:

1. Every filter in the read set must match. This is a conjunction: a read filtering `channel = c1` and `author = "dave"` only matches writes to documents that are both.
2. Then any single term must match. ****This is a disjunction: matching one word of the search string is enough. If the read set has no terms at all, step one succeeding is enough on its own.