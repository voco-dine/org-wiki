# Conversational Test Cases

Scripted, end-to-end conversations for exercising the VocoDine voice ordering agent
(`voice-agent-backend` → `backend-services` → Supabase). They are written against the
**actual** agent behaviour — the prompt and tool surface in
[`src/assistant.py`](https://github.com/voco-dine/voice-agent-backend) and the matching/cart logic
in `src/menu/lookup.py` and `src/cart/` — not against aspirational design.

There are two suites:

<div class="grid cards" markdown>

- :material-check-circle:{ .lg .middle } **[Public test cases](public-test-cases.md)**

    ---

    Lower-risk, demonstrable happy-path and realistic-variation scenarios. Safe to run in front of
    stakeholders or a partner restaurant. These are the calls the system is *expected* to get right.

- :material-flask:{ .lg .middle } **[Private test cases](private-test-cases.md)**

    ---

    Internal stress, edge, and adversarial scenarios: off-menu requests, prompt injection, modifier
    abuse, STT mishearings, delivery-without-address, abusive callers. Some of these probe known
    gaps and may fail — that is the point. **Do not demo these externally.**

</div>

!!! warning "Private suite is red-team material"
    The private suite deliberately documents inputs designed to make the agent misbehave (prompt
    injection, price manipulation, abuse). Treat it as internal-only. If the wiki is ever published,
    gate or remove `private-test-cases.md`.

---

## How to use these

These are **manual / annotation scripts**, not automated unit tests. The repo-level automated tests
live in `voice-agent-backend/tests/` (LiveKit Agents eval framework + pytest). Use these documents
to:

1. **Drive a live call.** Speak the *Caller* turns into the dashboard frontend (`pnpm dev`) or a SIP
   call and check the agent's behaviour against the *Expected* block.
2. **Build ground-truth transcripts** for the [order-accuracy evaluation](../evaluation.md#1-task-success-objective).
   The "Expected order (structured)" block is the ground-truth target for that turn.
3. **Seed eval cases.** Each scripted turn maps onto a `pytest` eval (assert the right tool was
   called with the right arguments, or that the spoken reply has a given trait).

### Conventions

| Symbol | Meaning |
|---|---|
| **Caller** | What the human says (speak this, or type it in the web client). |
| **Agent** | Expected spoken reply (intent, not verbatim — TTS wording will vary). |
| `tool(args)` | The function tool the agent is expected to call, with arguments. |
| ✅ Pass | The assertion that must hold for the case to pass. |
| ⚠️ Known gap | Documented current behaviour that may be undesirable — flagged for a product decision, not a hard failure. |

**Tool surface** (all on `Assistant`): `add_item`, `remove_item`, `modify_item`,
`get_item_details`, `get_category_items`, `get_order_summary`, `confirm_order`, `request_handoff`.
See the [sequence diagram](../system-specification/sequence.md) for what each tool touches.

### Behaviour the suites rely on

These are facts about the running code that the cases assert against:

- **Greeting is fixed and proactive.** The agent opens every call with
  *"Hi there, thanks for calling Vocodine Restaurant! What can I get for you?"* (`Assistant.on_enter`).
  The agent only speaks once a participant is in the room (`wait_for_participant` before bootstrap), so
  the opening line isn't clipped while the client subscribes to the agent track.
- **The server computes totals.** The agent reads back a running total from the in-memory cart, but
  `POST /agent/orders` recomputes `total_amount` and sets `status='pending'` server-side. Treat the
  DB total as authoritative.
- **Item resolution is substring-based and exact-match-first** (`menu.lookup.find_item`): exact name
  wins; otherwise a single substring match resolves; 0 matches → `off_menu`; ≥2 matches →
  `ambiguous` (agent reads the candidates back).
- **Modifiers must belong to the named item** (`find_modifier`). A modifier the item doesn't offer
  comes back as `unknown_modifiers`; the agent must ask the caller to pick a supported option rather
  than guess. Only **Caesar Salad** and **Margherita Pizza** have modifier groups.
- **Delivery requires an address.** `confirm_order(order_type="delivery")` without an address
  returns `need_address` and does **not** POST. The address must go in the `address` field, never in
  `special_instructions`.
- **Every order requires a valid phone number.** `confirm_order` returns `need_phone` when no number
  is given and `invalid_phone` when the digit count is implausible (outside 10–15, e.g. a transcription
  that dropped all but the last 4 digits) — neither POSTs. `normalize_phone` (`cart/state.py`) only
  checks the digit count and preserves the caller's formatting; `invalid_phone` echoes back what was
  heard in `phone_heard` so the agent can read it out and re-ask.
- **Tool failures carry an `agent_action`.** Every non-success tool result (`off_menu`, `ambiguous`,
  `not_in_order`, `need_address`, `need_phone`, `invalid_phone`, `unknown_modifiers`, …) includes a
  plain-language `agent_action` string telling the agent how to recover. It is guidance for the model,
  not a line to read aloud verbatim.
- **There is no upselling step.** The agent should not push add-ons unprompted.
- **No multi-language support** (English only — see [Scope](../scope.md)).

---

## Menu snapshot

Captured from the live Supabase `stores`/`menu_*`/`modifier*` tables on **2026-06-14**. The DB is
the source of truth and is fetched fresh at the start of every call (`GET /agent/menu`); re-pull
before a test run if the menu may have changed.

**Store:** Vocodine Restaurant · timezone `Asia/Singapore` · tenant *Vocodine Dev Tenant*
**Mealtime windows** (defined in `store_mealtimes`, **not currently enforced** by the agent — see
[private E-series](private-test-cases.md#e-known-gaps-system-behaviour)): Lunch 11:00–15:00, Dinner 17:00–22:00.

### Items

| Category | Item | Price | Allergens | Modifiers |
|---|---|---:|---|---|
| Mezze & Starters | Bruschetta | $8.99 | Gluten | — |
| Mezze & Starters | Caesar Salad | $10.99 | Dairy, Eggs | **Size** (req), **Add Protein** (opt) |
| Mezze & Starters | Hummus Trio | $12.99 | Sesame, Gluten | — |
| Mains | Chicken Shawarma Bowl | $17.99 | Sesame, Dairy | — |
| Mains | Grilled Salmon | $22.99 | Fish | — |
| Mains | Lamb Kofta | $19.99 | Dairy, Gluten | — |
| Flatbreads & Pizza | Margherita Pizza | $14.99 | Gluten, Dairy | **Size** (req), **Extra Toppings** (opt) |
| Flatbreads & Pizza | Za'atar Flatbread | $13.99 | Gluten, Dairy, Sesame | — |
| Sweets | Baklava | $8.99 | Gluten, Nuts | — |
| Sweets | Tiramisu | $9.99 | Gluten, Dairy, Eggs | — |
| Beverages | Fresh Lemonade | $4.99 | — | — |
| Beverages | Turkish Coffee | $5.99 | — | — |

### Modifier groups

| Item | Group | Selection | Options (price delta) |
|---|---|---|---|
| Caesar Salad | **Size** | single, required | Regular ($0), Large (+$3.00) |
| Caesar Salad | **Add Protein** | multi, optional, max 3 | Grilled Chicken (+$4.00), Shrimp (+$6.00), Salmon (+$7.00) |
| Margherita Pizza | **Size** | single, required | 10" Small ($0), 12" Medium (+$3.00), 14" Large (+$6.00) |
| Margherita Pizza | **Extra Toppings** | multi, optional, max 5 | Mushrooms (+$1.50), Olives (+$1.50), Peppers (+$1.50), Pepperoni (+$2.00), Extra Cheese (+$2.00) |

!!! note "Modifier cardinality is metadata, not enforced"
    `is_required`, `min_selections`, and `max_selections` exist in `modifier_groups` but the agent
    and `POST /agent/orders` do **not** enforce them today. A Margherita Pizza can be confirmed with
    no Size (priced at the $14.99 base), and "Add Protein" can be requested more than 3 times. The
    private suite covers these.
</content>
</invoke>
