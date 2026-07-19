# Prompt / Workflow — GenAI Client Intelligence Mini-Case

## Workflow

```
Coach-client transcript (raw text)
        │
        ▼
Single LLM call to Claude
  • system prompt below, enforcing a closed confidence-tier taxonomy
  • output constrained to a strict JSON schema
  • temperature-sensitive: no free-text commentary allowed
        │
        ▼
Client-side JSON parse
  • light fence-stripping (in case of ```json wrappers)
  • schema validated by rendering logic
        │
        ▼
Rendered report with human-in-the-loop review
  • each finding shown with its confidence tier + evidence quote
  • coach can Approve / Edit / Reject every individual finding
  • nothing is treated as final until a human reviews it
```

One LLM call is sufficient for a single week of conversation. For longer histories (months) or multiple clients at once, this would split into two passes instead: a **per-day extraction pass** (pulling out raw facts/mentions per category) followed by a **weekly aggregation pass** (synthesizing across days, generating the summary and risk flags) — this avoids overloading one call's context and token budget, and makes the contradiction-detection logic (see prompt rule 4 below) easier to enforce explicitly rather than hoping the model catches it inside one long pass.

## System prompt (used verbatim in the prototype)

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

Schema:
{
  "weekly_summary": string,
  "findings": [
    {
      "id": string (slug),
      "category": one of ["nutrition_adherence","exercise_steps","sleep","water_intake",
                            "symptoms_stress","engagement_level","key_barriers","pending_actions"],
      "label": short human-readable label,
      "value": one-sentence finding,
      "confidence_tier": one of ["confirmed_fact","client_reported","ai_inference","missing_data"],
      "evidence": [{"quote": string, "day": string}]
    }
  ],
  "risk_flags": [
    {"flag": string, "severity": one of ["high","medium","low"], "rationale": string,
     "confidence_tier": string, "evidence":[{"quote":string,"day":string}]}
  ],
  "recommended_next_action": {
    "action": string,
    "rationale": string,
    "confidence_tier": one of ["client_reported","ai_inference"]
  }
}

Produce exactly one finding per category listed above (8 findings total), plus 1-3 risk_flags,
plus one recommended_next_action. Be concise — this output must fit within a strict token budget.
```

## Design logic behind the prompt

- **Confidence tiers are defined by evidentiary source, not a fuzzy "confidence score".** A coach doesn't need a 0–100 number; they need to know whether something is a logged data point, a self-report, a synthesized judgment, or simply absent — because that determines whether they act on it directly or verify with the client first.
- **`confirmed_fact` is deliberately strict.** Numbers the client mentions casually in chat ("slept around 5 hours") are `client_reported`, not `confirmed_fact` — only structured tracker-style log entries (from the "Accountability Coach" role in this transcript) earn the confirmed tier. This is the main lever against false confidence in the output.
- **Mandatory verbatim evidence** forces the model to point at its source rather than summarize from impression, and lets the coach verify a finding in one click without re-reading the whole transcript.
- **The no-unhedged-causality rule** exists because health/coaching transcripts invite plausible-but-unproven causal stories (poor sleep → low energy → mistakes at work); the prompt forces those into the `ai_inference` tier with hedged language instead of stated as fact.
- **The contradiction rule** (point 4) exists specifically to stop the model from quietly picking the more flattering of two conflicting self-reports across the week, which is a common way multi-turn LLM summaries drift overly positive.
