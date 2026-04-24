# Evaluation

The system is evaluated across three dimensions.

## 1. Task Success (Objective)

**Metric:** Order accuracy — does the final structured JSON match what the caller intended?

**Method:** Compare agent output against manually annotated ground-truth transcripts from test calls.

## 2. System Performance (Technical)

| Metric | Target | Method |
|---|---|---|
| End-to-end latency per turn | < 1 s | Structured logs from LangGraph node transitions |
| Grounding accuracy | High % of confirmed orders containing only valid menu items/combos | Log validation failures at confirmation node |
| Human handoff rate | Minimised | % of calls escalated to staff |

## 3. User Satisfaction (Subjective)

| Instrument | Detail |
|---|---|
| WhatsApp rating | 1–5 scale, sent ~30–45 min post-call. See [ADR-005](decisions/005-whatsapp-sms-feedback.md) |
| Free-text follow-up | For ratings ≤ 3 only |
| SUS questionnaire | Standard 10-question System Usability Scale, administered to the test group after sessions |

## Open Questions

- **LLM choice benchmarking** (GPT-4o vs. Claude): under consideration, may be out of FYP scope. TBD.
- **Baseline comparison**: comparing against a naive LLM-only approach (no grounding, no state management) to demonstrate measurable improvement from the stateful architecture — under consideration as part of the evaluation design.
- **Ground truth annotation**: who annotates test calls and how many calls are needed for a statistically meaningful sample — TBD.
