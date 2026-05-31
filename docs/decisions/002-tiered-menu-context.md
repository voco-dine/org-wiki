# ADR-002: Tiered Menu Context Injection

**Status:** Accepted  
**Date:** 2026-04

## Decision

Use a three-tier context injection strategy for menu data rather than placing a vector DB retrieval step in the critical call path.

## Context

The agent needs menu awareness on every turn. The naive approach is to embed the full menu into every prompt, or to run a vector similarity search on each turn to retrieve relevant items. Both have problems: full menu inclusion is token-heavy and slow; mandatory vector retrieval adds 200–400 ms per turn against a <1 s latency budget.

## Rationale

For typical restaurant menus (30–150 items), the LLM can resolve item names, fuzzy matches, and common modifier requests directly from a compact in-context representation — no retrieval needed. The tiered approach scales context with conversation complexity rather than menu size.

| Tier | Content | When Loaded | Cost |
|---|---|---|---|
| **1** | Names, categories, prices only | Always in system prompt | ~200–400 tokens, cached by LLM provider after first turn |
| **2** | Full category detail (sizes, toppings, variants) | Returned on demand by the `get_category_items` tool | Active category only |
| **3** | Item-level detail (modifiers, allergens, descriptions) | On-demand via `get_item_details(item_id)` tool call | Single Redis lookup, microseconds |

**Vector DB kept as conditional fallback only.** If the LLM cannot confidently resolve a vague query (e.g. "something light and spicy"), it can call a semantic search tool. The latency cost is acceptable in that context — the agent can naturally say "let me check" for unusual requests.

## Trade-offs Accepted

- Requires careful prompt design so Tier 1 stays compact as menus grow.
- Very large menus (200+ items) may need a stricter categorisation strategy to keep Tier 2 injections manageable.
- Fuzzy matching relies on the LLM's in-context reasoning; accuracy degrades for highly ambiguous or misspelled inputs (mitigated by Tier 3 fallback).
