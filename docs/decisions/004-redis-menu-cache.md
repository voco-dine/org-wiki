# ADR-004: Redis as Menu Read Cache

**Status:** Accepted  
**Date:** 2026-04

## Decision

Use Redis as a read cache on top of NeonDB for menu data during live conversations. NeonDB remains the source of truth.

## Context

Live conversations require sub-millisecond menu reads to stay within the <1 s latency budget. NeonDB queries average 5–20 ms — acceptable for writes and analytics, but too slow to be in the per-turn hot path.

## Architecture

```
Adapter sync → NeonDB (source of truth)
                    ↓ (cache warm-up at call start, periodic sync)
                  Redis (read cache)
                    ↓ (miss or failure)
                  NeonDB (fallback)
```

NeonDB holds the canonical menu data: items, categories, modifiers, prices, and descriptions. Redis is populated by the adapter sync and kept warm with tiered TTLs. On a cache miss or Redis failure, the agent falls back to NeonDB directly — no data loss, just a slower read.

## TTL Strategy

| Data Type | TTL | Rationale |
|---|---|---|
| Menu structure (items, categories, modifiers, prices) | 30–60 min | Changes infrequently; adapter sync keeps it fresh |
| Availability (in-stock flags, daily specials) | 2–5 min | Changes more frequently; short TTL limits stale reads |

## Modifier Validation

At sync time the adapter pre-computes a Redis set per item: `menu:{restaurant}:item:{id}:valid_modifiers`. Validation during a call is a single `SISMEMBER` (O(1)) rather than a SQL JOIN at query time.

## Trade-offs Accepted

- All relational constraints must be denormalised into Redis at sync time — no joins at query time.
- Redis data is lost on restart unless AOF or RDB persistence is configured. Fallback to NeonDB covers this, but adds latency on that call.
- Schema evolution is by convention; no migration tooling for the cache layer.
