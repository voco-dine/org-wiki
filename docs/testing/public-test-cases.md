# Public Test Cases

Lower-risk, demonstrable scenarios — the calls the agent is **expected** to handle cleanly. Safe to
run in front of stakeholders or a partner restaurant. Read the
[conventions and menu snapshot](index.md) first; prices and modifiers below are taken from that
snapshot (2026-06-14).

Every call opens with the fixed greeting:

> **Agent:** "Hello, thank you for calling Vocodine Restaurant! I'm your voice ordering assistant.
> What can I get for you today?"

The greeting is omitted from the scripts below for brevity — assume it is turn 0.

!!! warning "Deferred while modifiers & handoff are out of scope"
    Item **modifiers** (size, toppings, proteins) and **human handoff** are currently out of scope
    (tracked in voice-agent-backend
    [#9](https://github.com/voco-dine/voice-agent-backend/issues/9) and
    [#10](https://github.com/voco-dine/voice-agent-backend/issues/10)). The cases that demo them —
    **PUB-03**, **PUB-04** (modifiers) and **PUB-10** (handoff) — are kept below for when those
    features land but are marked **DEFERRED**; don't run them in the current suite. Also, browsing is
    now answered from the **grounded in-context menu** (the prompt embeds the full menu) rather than a
    `get_category_items` call — see PUB-02.

---

## PUB-01 — Single item, direct add, pickup

**Goal:** Smoke test the happy path: name an item, confirm, pick up.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "Hi, can I get a Lamb Kofta please?" |
| 2 | Agent | Confirms Lamb Kofta added, $19.99; asks if there's anything else. |
| 3 | Caller | "That's it." |
| 4 | Agent | Reads back: 1 Lamb Kofta, total $19.99. Asks pickup or delivery. |
| 5 | Caller | "Pickup." |
| 6 | Agent | Asks for a phone number for the order. |
| 7 | Caller | "0412 345 678." |
| 8 | Agent | Confirms the order is placed; says goodbye. |

**Expected tool calls:** `add_item("Lamb Kofta")` → `get_order_summary()` (or reads from add result)
→ `confirm_order(order_type="pickup", customer_phone="0412345678")`.

- ✅ `add_item` returns `status="added"`, `order_total="19.99"`.
- ✅ `confirm_order` returns `status="confirmed"` with an `order_id`.
- ✅ **DB:** one `orders` row, `type='pickup'`, `address` NULL, `status='pending'`,
  `total_amount=19.99`; one `order_items` row.
- ✅ Agent never asked for a delivery address (pickup).

---

## PUB-02 — Browse a category, then order

**Goal:** Browsing — the agent reads one category at a time from the grounded menu, never the whole menu.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "What do you have?" |
| 2 | Agent | Names the **categories** (Mezze & Starters, Mains, Flatbreads & Pizza, Sweets, Beverages) and asks which to hear. |
| 3 | Caller | "What's in Mains?" |
| 4 | Agent | Lists Chicken Shawarma Bowl $17.99, Grilled Salmon $22.99, Lamb Kofta $19.99. |
| 5 | Caller | "I'll do the salmon." |
| 6 | Agent | Confirms Grilled Salmon $22.99 added. |
| 7 | Caller | "And a lemonade. That's all, for pickup, 0412 345 678." |
| 8 | Agent | Reads back both items + total $27.98, confirms pickup, places order. |

**Expected tool calls:** `add_item("salmon")` → `add_item("lemonade")` →
`confirm_order(order_type="pickup", customer_phone=...)`. The agent reads the Mains items **from the
grounded in-context menu** (the prompt now embeds the full menu with prices), so a
`get_category_items` call is no longer required — it may still call it, but the read-back must come
from real menu data either way.

- ✅ Turn 4 reads back the **real Mains items with prices** — no invented dishes (this is the regression guard for the hallucinated-menu bug).
- ✅ "salmon" resolves to **Grilled Salmon** (single substring match), not the Salmon protein modifier.
- ✅ DB order total `27.98` (22.99 + 4.99), two `order_items`.

---

## PUB-03 — Pizza with size and a topping

> **⚠️ DEFERRED — modifiers out of scope.** Restore when [voice-agent-backend#9](https://github.com/voco-dine/voice-agent-backend/issues/9) lands. The prompt no longer solicits size/toppings.

**Goal:** Required single-select size + an optional topping resolve correctly, and the price reflects the deltas.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "A Margherita pizza, the 12 inch medium, with extra cheese." |
| 2 | Agent | Confirms 12" Medium Margherita with extra cheese; states the price. |
| 3 | Caller | "Perfect, that's everything. Pickup. 0455 222 111." |
| 4 | Agent | Reads back, confirms, places order. |

**Expected tool calls:** `add_item("Margherita pizza", modifiers=["12 inch medium", "extra cheese"])`
→ `confirm_order(...)`.

- ✅ Both modifiers resolve (`12" Medium` +$3.00, `Extra Cheese` +$2.00); `unknown_modifiers` is empty.
- ✅ Line price = 14.99 + 3.00 + 2.00 = **$19.99**.
- ✅ DB: `order_items` row with two `order_item_modifiers`.

---

## PUB-04 — Caesar Salad with a protein

> **⚠️ DEFERRED — modifiers out of scope.** Restore when [voice-agent-backend#9](https://github.com/voco-dine/voice-agent-backend/issues/9) lands. The prompt no longer solicits size/proteins.

**Goal:** The other modifier-bearing item; multi-select optional protein.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "Caesar salad, large, add grilled chicken." |
| 2 | Agent | Confirms large Caesar Salad with grilled chicken; states the price. |
| 3 | Caller | "That's it, pickup, 0466 333 222." |

**Expected tool calls:** `add_item("Caesar salad", modifiers=["large", "grilled chicken"])` → `confirm_order(...)`.

- ✅ `Large` (+$3.00) and `Grilled Chicken` (+$4.00) resolve; line price = 10.99 + 3.00 + 4.00 = **$17.99**.
- ✅ Agent does not confuse the salad's "Salmon" protein option with the standalone Grilled Salmon main.

---

## PUB-05 — Ask for item details before ordering

**Goal:** Tier-3 single-item detail lookup (description, price, allergens).

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "What's in the Hummus Trio, and is it $12.99?" |
| 2 | Agent | Describes the trio (classic, roasted red pepper, herb), confirms $12.99, mentions Sesame and Gluten if asked about allergens. |
| 3 | Caller | "Great, one of those for collection. 0477 111 000." |

**Expected tool calls:** `get_item_details("Hummus Trio")` → `add_item("Hummus Trio")` → `confirm_order(order_type="pickup", ...)`.

- ✅ Detail comes from the in-memory menu (no extra DB round trip).
- ✅ "collection" normalizes to **pickup** (`normalize_order_type`).
- ✅ Stated price and allergens match the snapshot exactly — no invented detail.

---

## PUB-06 — Delivery with a clean address

**Goal:** The delivery branch with the address supplied up front.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "Two Chicken Shawarma Bowls for delivery." |
| 2 | Agent | Confirms 2 × Chicken Shawarma Bowl; asks for the delivery address. |
| 3 | Caller | "12 Marina Boulevard, unit 05-10, Singapore 018980." |
| 4 | Agent | Reads the address back to confirm. |
| 5 | Caller | "Correct. Number is 0488 999 000." |
| 6 | Agent | Reads back order + total $35.98, confirms delivery, places order. |

**Expected tool calls:** `add_item("Chicken Shawarma Bowl", quantity=2)` →
`confirm_order(order_type="delivery", address="12 Marina Boulevard, unit 05-10, Singapore 018980", customer_phone=...)`.

- ✅ Agent **reads the address back** before confirming (prompt step 3).
- ✅ DB: `type='delivery'`, `address` populated, `total_amount=35.98`.
- ✅ Address is in the `address` column, **not** in `special_instructions`.

---

## PUB-07 — Mid-order change (quantity)

**Goal:** `modify_item` replaces a line in place — agent does not double-add.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "One Tiramisu." |
| 2 | Agent | Confirms 1 Tiramisu, $9.99. |
| 3 | Caller | "Actually make it two." |
| 4 | Agent | Updates to 2 Tiramisu, total $19.98. |
| 5 | Caller | "That's all, pickup, 0411 222 333." |

**Expected tool calls:** `add_item("Tiramisu")` → `modify_item("Tiramisu", quantity=2)` → `confirm_order(...)`.

- ✅ Agent calls `modify_item`, **not** a second `add_item` (prompt: "do NOT call add_item again").
- ✅ Final cart has a single Tiramisu line, quantity 2 — not two lines.

---

## PUB-08 — Remove an item

**Goal:** `remove_item` and a corrected total.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "A Bruschetta and a Baklava." |
| 2 | Agent | Confirms both; running total $17.98. |
| 3 | Caller | "Actually drop the Baklava." |
| 4 | Agent | Removes Baklava; total back to $8.99. |
| 5 | Caller | "That's it, pickup, 0400 000 111." |

**Expected tool calls:** `add_item("Bruschetta")` → `add_item("Baklava")` → `remove_item("Baklava")` → `confirm_order(...)`.

- ✅ `remove_item` returns `status="removed"`, `order_total="8.99"`.
- ✅ DB order has only the Bruschetta line.

---

## PUB-09 — Read the order back on request

**Goal:** `get_order_summary` mid-call.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "A Fresh Lemonade, a Turkish Coffee, and a Bruschetta." |
| 2 | Agent | Confirms all three. |
| 3 | Caller | "What do I have so far?" |
| 4 | Agent | Reads back: Fresh Lemonade $4.99, Turkish Coffee $5.99, Bruschetta $8.99 — total $19.97. |
| 5 | Caller | "Good, pickup, 0422 333 444." |

**Expected tool calls:** three `add_item` → `get_order_summary()` → `confirm_order(...)`.

- ✅ Summary lists exactly the three lines and the correct total $19.97.

---

## PUB-10 — Polite human handoff

> **⚠️ DEFERRED — handoff out of scope.** Restore when [voice-agent-backend#10](https://github.com/voco-dine/voice-agent-backend/issues/10) lands. The prompt no longer instructs the agent to escalate, so it won't reliably hand off today.

**Goal:** The caller asks for a person; the agent escalates gracefully.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "Can I speak to someone about catering for 40 people?" |
| 2 | Agent | Acknowledges, transfers to a staff member (handoff). |

**Expected tool calls:** `request_handoff(reason="catering enquiry / large group")`.

- ✅ `request_handoff` returns `status="handoff_initiated"`.
- ✅ **DB:** the call row's `human_handoff` flips to `true` (fire-and-forget PATCH).
- ✅ Agent stays polite and does not invent a catering menu or pricing.

---

## PUB-11 — Off-menu item, graceful decline + recovery

**Goal:** A common off-menu ask is declined cleanly and the agent steers back to the real menu.

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "Do you have a cheeseburger?" |
| 2 | Agent | Says it's not on the menu; offers to share what *is* available (e.g. Mains, Pizza). |
| 3 | Caller | "Okay, a Margherita then, small. Pickup, 0433 444 555." |

**Expected tool calls:** `add_item("cheeseburger")` → returns `status="off_menu"` →
(agent declines) → `add_item("Margherita")` → `confirm_order(...)`.

- ✅ Agent does **not** invent a burger, a price, or a substitute it can't sell.
- ✅ Recovers into a valid order (Margherita Pizza, $14.99 base — size is out of scope for now).

---

## PUB-12 — Allergen question answered from real data

**Goal:** A safe, factual allergen answer drawn straight from the menu (no medical advice, no invention).

| # | Speaker | Utterance |
|---|---|---|
| 1 | Caller | "Does the Baklava have nuts?" |
| 2 | Agent | Yes — Baklava lists **Nuts** (and Gluten). |
| 3 | Caller | "Then I'll skip it. What sweets are nut-free?" |
| 4 | Agent | Tiramisu has no nuts listed (Gluten, Dairy, Eggs); offers it. |
| 5 | Caller | "One Tiramisu, pickup, 0444 555 666." |

**Expected tool calls:** `get_item_details("Baklava")` → `get_item_details("Tiramisu")` → `add_item("Tiramisu")` → `confirm_order(...)`. Allergens are **not** in the grounded menu (names + prices only), so `get_item_details` is still required here; the sweets list itself can be read from the grounded menu.

- ✅ Allergen facts match the snapshot exactly.
- ✅ Agent states only the listed allergens; it does not claim the kitchen is nut-free or give medical/cross-contamination guarantees. (Deeper allergy probing is covered in the private suite.)
</content>
