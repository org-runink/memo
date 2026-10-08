> **Moved.** This module now lives at **[github.com/org-runink/stdx](https://github.com/org-runink/stdx)**.
> The code is unchanged; only the import path differs. This repository is archived.

# memo

In-process memoization for Go with **single-flight coalescing**, TTL expiry and
LRU eviction. **Zero dependencies** — standard library only.

```go
c := memo.New[string, Answer](memo.Options{Capacity: 4096, TTL: 10 * time.Minute})

key := memo.Hash("model-v3", 0.2, map[string]any{"q": q, "lang": "en"})
ans, err := c.Do(ctx, key, func(ctx context.Context) (Answer, error) {
    return expensive(ctx, q)   // runs once, however many callers arrive
})
```

## Three things it does that a plain LRU does not

**1. Single-flight is inside the cache, not beside it.**
A plain cache helps the *second* caller. Under concurrency the problem is the
first N callers arriving together on a cold key: each misses, each runs the
expensive call. `Do` coalesces them — one execution, every caller gets the
result. Sequential repeats and concurrent duplicates are different populations
and neither mechanism substitutes for the other, so this package does both.

**2. Errors are not cached by default.**
Caching a failure turns a transient fault into a sticky one for the whole TTL.
It is the most common way a memo cache makes a system worse. A failed call is
**not stored**, but it *is* still shared with the callers that coalesced onto it
— so a stampede against a failing dependency produces one call, not N. Opt in
with `CacheErrors` if you mean it.

**3. `Hash` makes "the same request" a decision you can inspect.**
The hard part of exact-match caching is not storage, it is deciding two requests
are the same.

```go
memo.Hash("model-v3", map[string]any{"q": q, "top_k": 5})
```

- **Map keys are sorted.** Go randomises map iteration, so hashing a map
  naively produces a different key every run — a cache that never hits and never
  tells you why. This is the bug the package exists to prevent.
- **Values are type-tagged**, so `Hash("1")`, `Hash(1)` and `Hash(true)` differ.
- **Lengths are prefixed**, so `{"ab","c"}` and `{"a","bc"}` cannot collide.
- **NaN and −0.0 are canonicalised**, so they hash consistently.

`NormalizeSpace` is offered separately rather than applied inside `Hash`, because
only the caller knows whether whitespace is noise (a re-wrapped prompt) or
meaning (code).

## Benchmarks

```
goos: linux   goarch: amd64   cpu: AMD Ryzen 7 8840U
go test -bench . -benchmem -benchtime=1000x
```

| Benchmark | ns/op | B/op | allocs/op |
|---|---:|---:|---:|
| `Do` — hit | **14.34** | 0 | **0** |
| `Do` — miss (store + LRU insert) | 694.2 | 358 | 5 |
| `Do` — hit, `RunParallel` on 16 threads | 134.9 | 9 | 0 |
| `Hash` — 4-field map | 1,579 | 656 | 36 |
| stampede, 8 concurrent callers, cold key | 5,332 | 1,287 | 19 |
| stampede, 64 concurrent callers, cold key | 24,324 | 4,136 | 75 |

Dev-laptop figures, not capacity numbers.

**The hit path is 14 ns and allocates nothing**, so the cache is free relative to
anything worth memoizing.

**The parallel hit is 135 ns, not 14.** That is mutex contention across 16
threads, and it is stated rather than hidden: `Store` takes one lock per
operation. At ~7M hits/sec aggregate it is far from the bottleneck for LLM or
network calls, but if you are memoizing something that costs less than a
microsecond, shard the store or use a different design.

**The stampede benchmark is the case single-flight exists for**: 64 goroutines on
a cold key cost 24 µs and **one** execution of the function, not 64.

Test coverage is **82.4%**, including `-race`.

## What this is not

- **Not distributed.** Per-process, no network, no shared state. Two replicas
  have two caches. The moment a cache is shared it needs invalidation, coherence
  and a failure mode of its own; this one has none of those because it has no
  peers.
- **Not a semantic cache.** Keys match exactly. There is no embedding and no
  similarity threshold, so it cannot return a confidently wrong answer because
  two requests looked alike.
- **Not persistent.** Nothing survives a restart.

## API

| | |
|---|---|
| `New[K,V](Options)` | build a store |
| `Do(ctx, key, fn)` | memoized call with single-flight |
| `Get` / `Set` / `Invalidate` / `Purge` | direct access |
| `Stats()` → `HitRate()` | hits, misses, coalesced, evictions, expired |
| `Hash(parts...)` | deterministic key from structured input |
| `NormalizeSpace(s)` | collapse whitespace before hashing free text |

`Options.Now` injects a clock, so tests advance time instead of sleeping.

## Licence

BSD-3-Clause.
