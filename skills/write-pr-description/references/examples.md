# Example PR descriptions

These are written for this skill. Headings follow pstack's opening-a-pr playbook (Why, What changed, Scope, Tradeoffs, Blast Radius, Verification) plus this skill's final Read the code here section. The systems and PR numbers in them are fictional, so copy the shape and the habits, not the facts.
Each example is followed by notes on what makes it work.
The first two follow the spirit of the short examples in Google's "Writing good CL descriptions" guide; the later ones show the larger changes that guide does not cover.

If `house-examples.md` has entries from the user's own repositories, prefer those for tone and section names.

---

## Example 1: small change that needs a little context

**Title:** Cap the export job's page size at 500 rows

```markdown
The nightly export job asked the API for 5,000 rows per page. Since the API started enforcing its documented 500-row maximum on 2026-03-02 (#2210), every export after the first page failed with a 400 and the job retried until it timed out. This sets the page size to 500, read from the same `MAX_PAGE_SIZE` constant the API uses, so the two cannot drift again.

Verified: ran the export against staging; it completed in 11 minutes across 38 pages (log linked in the job run). No unit test: the job has no test harness, and the constant is now shared with the API, which has one.
```

Why it works:

- The first sentence names the effect (failing exports), not the constant.
- The why has a source (#2210) and a date, so a reader can check it.
- One decision is surfaced: sharing the constant instead of hard-coding 500.
- The missing test is stated with a reason, not hidden.
- No headings. A change this size does not need them.

---

## Example 2: bug fix with a non-obvious cause

**Title:** Stop double-counting retried writes in the ingest metrics

```markdown
## Why

The `rows_ingested` metric overcounts by the number of retried batches, so dashboards showed up to 30% more ingest than the table holds during the 2026-05 incident (link). Billing reads this metric, so the overcount reached invoices for two customers.

## What changed

### Before

`BatchWriter.write()` increments `rows_ingested` before it calls `commit()`. When `commit()` aborts and the batch is retried, the increment runs again for the same rows. Successful first attempts were counted correctly, which is why the overcount only appeared under contention.

    batch of 100 rows: abort, abort, commit
    before:  rows_ingested += 100  x3  ->  300
    after:   rows_ingested += 100  x1  ->  100

### After

The increment moves after a successful `commit()`, and counts the rows the commit reports, not the rows in the request. A retried batch is now counted once. A batch that fails all retries is no longer counted at all (before, it was counted once per attempt).

## Tradeoffs

- Counting after commit, rather than de-duplicating by batch ID at read time. The metrics pipeline has no batch ID today, and adding one is a schema change across three services.

## Blast Radius

- The metric drops on deploy for any team with a high retry rate. That drop is the correction, not a regression. The billing team knows (thread linked).
- Historical values stay inflated. Backfilling them is out of scope; see #3120.

## Verification

- New test `TestRetriedBatchCountedOnce` fails on main (counts 3) and passes here (counts 1).
- Not run: the contention load test, which needs the perf cluster. Expect `rows_ingested` to match `SELECT COUNT(*)` within 0.1% on that run.

## Read the code here

- `batch_writer.go`, the new increment: does it use the committed row count, not `len(batch)`?
```

Why it works:

- A three-line example shows the overcount faster than a paragraph could, and the claim check can verify it.
- Before names the exact faulty mechanism, so a reviewer can check the fix against the cause without reading the old code.
- After states a second behavior change (failed batches no longer counted) that a reviewer could easily miss in the diff.
- The one tradeoff answers the question a reviewer would ask ("why not de-duplicate at read time?"), in one sentence.
- Verification separates what ran from what is expected, and says which test fails on main.
- The closing list points at the one line where a mistake would be invisible in prose. Everything else can be approved from the body.

---

## Example 3: feature with a design to explain

**Title:** Add per-tenant read quotas to the query service

```markdown
## Why

One tenant's batch jobs can use all read capacity in a region, which slowed every other tenant's queries by 4 to 10 times during three incidents this quarter (postmortems linked). This adds a per-tenant read quota so one tenant can no longer starve the others. Quotas are off by default and turn on per region with a flag.

## What changed

Each tenant gets a token bucket per region, refilled at its quota rate. Every read request takes one token per 1,000 rows scanned, estimated before execution from the query plan.

    message TenantQuota {
      string tenant_id = 1;
      int64 rows_per_second = 2;   // 0 means unlimited
      int64 burst_rows = 3;        // defaults to 2 * rows_per_second
    }

Behavior when a tenant is over quota:

- Interactive queries wait up to 200 ms for tokens, then fail with `RESOURCE_EXHAUSTED` and a `retry_after` hint.
- Batch queries (priority `LOW`) wait up to 30 s, then fail the same way.
- Queries from the `system` tenant are never limited.

What does not change: tenants with no `TenantQuota` row are unlimited, and the flag is off in every region, so this PR changes no behavior until the flag is turned on.

Buckets live in memory on each replica, so a tenant's effective quota is its configured quota times the number of replicas serving it. The quota table stores the per-replica value for that reason.

## Scope

Admin UI for editing quotas (planned as the next PR). Quotas for writes.

## Tradeoffs

Quotas count rows scanned, not requests: one request can scan one row or a billion, and the incidents came from a few very large scans that a request limit would not have stopped.

## Blast Radius

- A tenant whose traffic lands on one replica gets only one replica's share of its quota. This matters for tenants with very few, long-lived connections. Not handled here; #881 tracks connection rebalancing.
- `RESOURCE_EXHAUSTED` is a new error for clients. The two internal SDKs retry it already; external clients may not.
- Rollback: turn the flag off. No stored data depends on this PR.

## Verification

- Unit tests cover refill, burst, the wait limits for both priorities, and the `system` exemption. Not covered: clock skew between replicas.
- Ran the replay of the 2026-07 incident traffic in staging with the flag on: other tenants' p99 stayed under 1.3x baseline (dashboard linked). With the flag off, the replay matched production behavior.

## Read the code here

- `quota/bucket.go`, `refill()`: is clock skew between replicas handled? The body gives the rule, but the arithmetic is where a mistake would hide.
- The default burst size (2x the per-second quota) comes from incident traces, not load testing. Is that enough evidence?
```

Why it works:

- The opening states the problem with its size and source, the effect of the change, and that it ships off. A reviewer knows the risk level from three sentences.
- The closing list names exactly which code the body does not fully specify, and one call the author is unsure of. Everything else can be approved from the body.
- The design gives the contract (proto), the behavior in each case, and what does not change. A reviewer could write a test from it.
- A non-obvious consequence of the design (quota multiplied by replica count) is stated, not left for the reviewer to deduce.
- It keeps one decision, the one a reviewer would raise ("why not limit requests?"), and drops the others made along the way. Estimating cost before execution and keeping buckets per replica are visible in What changed; nobody needs a paragraph defending them.

---

## Example 4: one step of a planned series

**Title:** Add the `orders_v2` table and dual-write to it

```markdown
## Why

Part 2 of 4 of the plan to move order IDs from 32-bit to 64-bit integers (plan linked; part 1 was #5120). This PR creates `orders_v2` with a 64-bit `order_id` and makes every write go to both tables. Reads still use `orders`, so no user-visible behavior changes.

## What changed

- `OrderStore.insert()` and `update()` write to `orders` first, then `orders_v2`, in the same transaction. A failure in either fails the write.
- `orders_v2` has the same columns and indexes as `orders`, except `order_id INT64`.
- A `dual_write_orders` flag (default on in staging, off in production) gates the second write.

## Scope

Backfill and read switch (part 3), dropping `orders` (part 4).

## Tradeoffs

- Same transaction, rather than an async copy. An async copy can lag or drop rows, and part 3 needs the two tables to agree exactly before it switches reads.

## Blast Radius

- Write latency rises with the second write; staging showed +1.8 ms p50 and +4 ms p99 (dashboard linked).
- Rows written before the flag is on are not in `orders_v2`. Part 3 backfills them.

## Verification

- Tests for insert, update, and a forced failure in each table's write.
- Staging has run with the flag on since 2026-09-20 with no dual-write errors (query linked).
```

Why it works:

- The first line places the PR in the series and links the plan, so a reviewer does not ask "why is nothing reading this table?".
- It says up front that there is no user-visible change, which sets the review level.
- The one decision explains a constraint that comes from a later PR.

---

## Anti-example: the usual agent draft

```markdown
## Summary

This PR implements comprehensive improvements to the quota handling system, making it more robust and scalable.

## Changes

- Added `quota/bucket.go` with `TokenBucket` struct
- Added `refill()` method
- Updated `handlers/read.go` to call `quota.Check`
- Updated `handlers/scan.go` to call `quota.Check`
- (12 more lines, one per handler)
- Added tests

## Testing

- All tests pass
- Tested thoroughly in staging
```

What is wrong, and which rule it breaks:

- No why. A reviewer cannot tell what problem this solves or how big it is. (Step 1: collect the why from a source.)
- "Comprehensive", "robust", "scalable" grade the work instead of describing it. (pstack `unslop`.)
- The Changes list restates the file list, which the diff already shows, and omits every behavior: what happens over quota, who is exempt, what the default is. (Step 3.)
- No decisions, no risks, no mention that the flag ships off. A reviewer has to read all of the code to learn any of it, so the description does nothing for the goal of approving from the body.
- "All tests pass" repeats CI. "Tested thoroughly in staging" is unsupported: what ran, and what did it show? (Step 3, Verification.)

---

## Anti-example: the branch's history leaks into the body

```markdown
## What changed

Initially I added a mutex around the cache, but that caused contention under load, so the next version switched to a sharded map. After review feedback, the final approach uses `sync.Map` instead, which is simpler. The earlier sharding code was removed.

## Tradeoffs

- Considered a mutex (v1): too much contention.
- Considered sharding (v2): more code than needed.
- Considered an LRU library: not needed yet.
```

The reviewer sees one diff against the base: a new cache backed by `sync.Map`. The mutex and the sharded map never existed on the base, so every sentence about them describes something the reviewer cannot see. Written as if the final version were the first:

```markdown
## What changed

Resolved tenant configs are cached in a `sync.Map`, so concurrent readers do not serialize on a lock. Entries never expire; the cache holds at most one entry per tenant, about 2,000 today.
```

No tradeoffs section: nobody reviewing a `sync.Map` cache would ask why it is not a sharded map. The "entries never expire" line stays, because that is a behavior a reviewer needs.
