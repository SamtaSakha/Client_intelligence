# GenAI Client Intelligence — Mini-Case Submission

## 1. What I built

An interactive prototype (`client_intelligence_prototype.html`) that takes a raw coach–client conversation transcript and turns it into a structured weekly intelligence report for the coach. It calls the Claude API live in the browser (no backend needed) with the system prompt below, parses the JSON response, and renders it as review-ready cards.

Each finding is tagged with one of four confidence tiers — **Confirmed fact**, **Client-reported**, **AI inference**, **Missing data** — shown as a colored chip and a left-edge tab on the card. Every finding (except "missing") carries an expandable evidence strip with a verbatim quote and the day it came from, so the coach can check the source in one click instead of trusting the summary blindly. Below each finding is an **Approve / Edit / Reject** control, and choosing "Edit" opens a text box for the coach's corrected value — this is the human-in-the-loop layer; nothing in the report is meant to be acted on unreviewed. 

Risk flags get their own section with severity dots (high/medium/low), and a single highlighted "Recommended next action" card sits at the bottom with its own confidence tier and rationale. A "View structured JSON" toggle exposes the raw output for engineering/QA purposes.

I used the actual anonymized sample conversation (8 days, client + coach + an "Accountability Coach" role posting daily numeric updates) as the pre-loaded default input, since it gave enough real signal — inconsistent meal timing, missing protein, bloating/acidity, a week of ~5–5.5 hr sleep culminating in the client falling asleep at a work meeting, then a good-sleep recovery day — to meaningfully test the extraction logic rather than working off a synthetic toy example.

## 2. Prompt / workflow used

**Workflow:** transcript → single LLM call (Claude, JSON-only output, temperature-sensitive fields constrained by a closed schema) → client-side JSON parse → render with human review controls. No multi-agent chaining was needed for a single week of conversation; if this scaled to months of history or multiple clients, I'd split into a per-day extraction pass + a weekly aggregation pass (see "what I'd improve").

**System prompt (used verbatim in the prototype):**

```
You are a client-intelligence analyst for fitness/wellness coaches. You will be given a raw
coach-client conversation transcript covering roughly one week.

Your job is to extract STRUCTURED, GROUNDED intelligence for the coach. You must never invent
information that is not present in the transcript.

For every finding, assign exactly one confidence_tier from this closed set:
- "confirmed_fact": stated as an objective logged data point (e.g. an explicit numeric update,
  typically from an accountability/tracking message), not just casual client chat.
- "client_reported": the client (or coach) said it in the conversation, but it is self-reported
  and not independently verified.
- "ai_inference": you are inferring a pattern, trend, or judgment (e.g. "adherence is
  inconsistent", "possible early burnout signs") by synthesizing multiple messages. This is your
  reasoning, not something anyone stated directly.
- "missing_data": the category was not covered in the transcript for some/all of the period. Say
  so explicitly rather than guessing.

Rules:
1. Never fabricate numbers (steps, litres, hours, kg) that are not explicitly present in the text.
2. Every finding except missing_data findings must include at least one short verbatim evidence
   quote (under 15 words) copied exactly from the transcript, with the day label if available.
3. Do not state inferred causal relationships (e.g. "stress caused the bloating") as fact — only
   as ai_inference, using hedged language ("may be linked to", "possible connection").
4. If information conflicts across days (e.g. client claims consistency early, then admits
   skipping meals later), reflect the conflict rather than only the more favorable statement.
5. Keep every "value" field to one concise sentence. Keep the weekly_summary to 2-3 sentences.
6. Output ONLY valid JSON matching the schema below. No markdown fences, no preamble, no
   commentary outside the JSON.

[schema — see section 3]

Produce exactly one finding per category listed [nutrition_adherence, exercise_steps, sleep,
water_intake, symptoms_stress, engagement_level, key_barriers, pending_actions] (8 findings
total), plus 1-3 risk_flags, plus one recommended_next_action.
```

Design choices baked into the prompt, and why:
- **The tier definitions are the load-bearing part of the prompt.** Rather than asking for "confidence: high/medium/low" (a fuzzy, gameable scale), I defined tiers by *evidentiary source* — was it an explicit logged number, a self-report, a synthesis, or absent? That maps directly to what a coach actually needs to know before acting: "can I trust this as-is, or should I double check with the client."
- **Confirmed vs. client-reported is intentionally strict.** Sleep hours and step counts the client mentions casually ("slept around 5 hours") are tagged `client_reported`, not `confirmed_fact` — even though they're numbers — because there's no device or independent source behind them. Only the "Accountability Coach: Today's update: Water 4 litres, Sleep 5 hours..." style structured log entries qualify as `confirmed_fact`. This distinction is the main hallucination-control lever in the design.
- **One quote requirement per finding** forces the model to point at its source instead of summarizing from "vibes," and gives the coach a one-click way to sanity-check without opening the full transcript.
- **The causality rule (point 3)** exists because this transcript specifically invites false causal claims (stress → bloating → weight, poor sleep → low energy → mistakes at work) that are plausible but not established by the conversation.

## 3. Structured output / JSON schema

```json
{
  "weekly_summary": "string, 2-3 sentences",
  "findings": [
    {
      "id": "string (slug)",
      "category": "nutrition_adherence | exercise_steps | sleep | water_intake | symptoms_stress | engagement_level | key_barriers | pending_actions",
      "label": "string, short human-readable label",
      "value": "string, one concise sentence",
      "confidence_tier": "confirmed_fact | client_reported | ai_inference | missing_data",
      "evidence": [
        { "quote": "string, verbatim, <15 words", "day": "string, e.g. 'Day 7'" }
      ]
    }
  ],
  "risk_flags": [
    {
      "flag": "string",
      "severity": "high | medium | low",
      "rationale": "string",
      "confidence_tier": "client_reported | ai_inference",
      "evidence": [{ "quote": "string", "day": "string" }]
    }
  ],
  "recommended_next_action": {
    "action": "string",
    "rationale": "string",
    "confidence_tier": "client_reported | ai_inference"
  },
  "review_meta": {
    "finding_id": "string",
    "coach_review_status": "pending | approved | edited | rejected",
    "coach_edit_value": "string | null"
  }
}
```

`review_meta` is tracked client-side per finding in the prototype (Approve/Edit/Reject state) rather than returned by the model — the LLM should never decide its own review status.

## 4. Three hallucination / failure scenarios

**1. Numeric fabrication under a "be specific" pressure.**
If a day has almost no logged data (e.g. Day 6, where the client only mentions "roasted chana and kala chana" with no step/water/sleep numbers), a model asked to fill in "exercise_steps" for that day can default to inventing a plausible-sounding number instead of admitting there isn't one. *Mitigation in this design:* the `missing_data` tier plus the explicit "never fabricate numbers" rule; the prototype visually distinguishes missing findings (grey, italic, dashed) so a coach immediately sees where the data has gaps rather than a suspiciously round number.

**2. Unlabeled causal inference presented as fact.**
The transcript has real correlation candidates (poor sleep + low energy + falling asleep at a meeting; low food intake + bloating + perceived weight gain). A model can slide from "these things co-occur" to "X is causing Y" without flagging that it's inferring. This is dangerous specifically in a health/coaching context because a coach might relay a false causal claim to the client as if it were established. *Mitigation:* explicit ban on unhedged causal language in the prompt, forcing any such claim into `ai_inference` with hedged phrasing, plus mandatory evidence citation so a coach can see the correlation isn't a stated fact.

**3. Recency bias erasing a contradiction (the "rosy summary" failure).**
The client says "walk and water done" early on and later admits skipping meals and missing ACV repeatedly. A model optimizing for a clean, upbeat weekly_summary can quietly favor the earlier positive statement and omit the later contradiction, producing an adherence summary that reads better than reality — a classic way LLM summarization drifts positive over multi-turn data. *Mitigation:* rule 4 in the prompt explicitly requires reflecting conflicts rather than the most favorable statement; in a production version I'd also add a second-pass "contradiction check" step that diffs early vs. late statements per category before the summary is finalized (see improvements below).

A related, lower-probability failure worth naming: **citation drift**, where the model's quote is close to but not exactly what was said (paraphrased and presented as verbatim). The evidence-quote requirement reduces but doesn't eliminate this — a production system should string-match each returned quote against the source transcript and flag/reject any quote that doesn't exactly appear.

## 5. Key assumptions

- The transcript is already anonymized and coach/client turns are reliably labeled (this transcript also had a third "Accountability Coach" role, which I treated as the more reliable, tracker-like source for the `confirmed_fact` tier).
- A week is the right unit of analysis for this report; longer histories would need chunking (see below).
- The coach, not the client, is the end user of this report — so language, risk flags, and the "next action" are written coach-side, not as client-facing advice.
- One conversation thread per client per week is the input granularity; multi-channel data (app logs, wearables) is out of scope for this prototype but is exactly what would upgrade more findings from `client_reported` to `confirmed_fact`.

## 6. What could go wrong (beyond the three scenarios above)

- **Schema drift:** the model occasionally emitting near-JSON (trailing commas, prose before/after) that breaks `JSON.parse`. The prototype only does light fence-stripping; production would need a JSON-repair step or Anthropic's structured-output/tool-use mode instead of free-text JSON.
- **Token budget truncation:** the response is capped at 1000 output tokens; a longer or noisier transcript could get cut off mid-JSON. Production would need dynamic budget sizing or a category-by-category call pattern instead of one big call.
- **Sensitive content handling:** this transcript touches on physical/emotional distress (falling asleep at work, "I feel very low," "I feel I can sleep for days"). A generic prompt could either over-flag (alarming the coach unnecessarily) or under-flag (missing something that warrants real follow-up). This needs care from a real clinical/product reviewer, not just a prompt-engineering fix — I treated the high-severity flag here as "recommend the coach checks in personally," not as any kind of diagnostic claim.

## 7. What I'd improve next

- Replace free-text JSON with Claude's structured output / tool-use mode so the schema is enforced by the API, not by hopeful parsing.
- Add an automatic **quote-verification pass**: after generation, string-match every evidence quote against the source transcript and auto-reject (or re-prompt) any finding whose quote doesn't appear verbatim.
- Add a **second LLM pass acting as a critic** — specifically checking for the three failure modes above (unlabeled numbers, unhedged causal claims, dropped contradictions) before the report reaches the coach.
- Persist the Approve/Edit/Reject state (currently in-memory only) so corrected values feed back into a training/eval set over time — that reviewed-vs-original delta is itself a great signal for improving the prompt.
- Extend the confidence-tier logic to reconcile against real tracker/wearable data when available, which is what would let more findings genuinely earn `confirmed_fact`.

https://cool-meringue-78ea25.netlify.app/
