# TypeSafe (Jev) Integration Guide

Read this only when Phase 3.5b's opportunity check (in `references/structure.md`)
flags a unit as a candidate, or when a unit's cited pattern points back here.
This guide covers TypeSafe's Jev model in enough depth to (a) recognize when a
plan's subject matter fits it and (b) write an accurate, optional Key
Technical Decision naming it — it does not make `ce-plan` call the API itself.
`ce-work`'s optional verification gate (`ce-work`'s
`references/jev-verification-gate.md`) is the one place in this plugin that
actually invokes Jev at runtime; everything here is advisory content for a
plan that will become someone's implementation.

## What Jev is, and is not

Jev is TypeSafe's "System One model" — the vendor's own term for a category
distinct from a general-purpose LLM. It does not generate text, prose, or
code. It accepts **state** (text, or a JSON/text blob you provide) plus one or
more **typed questions**, and returns typed answers with calibrated
probabilities and a confidence score — nothing else. There is no chat
interface, no tool use, no multi-turn reasoning.

Three question primitives, freely mixable in one batched request:

- **Choice** — "which of these options?" Returns the selected option, a
  probability distribution over every option, and a confidence score. Use
  for classification/routing against a known or near-known set.
- **Score** — "which level, on a defined rubric?" Returns a position on an
  ordered scale (can fall between two levels), the level legend, and
  confidence. Use for graded judgments (severity, quality, urgency).
- **Noul** — "is this true?" Returns a single probability from 0 (confident
  no) to 1 (confident yes), with ~0.5 meaning genuine uncertainty rather than
  "medium." Use for binary verification/detection questions.

Mechanics that matter for plan-writing: all questions in one request run in
parallel against the same state ("speculative fan-out" — batching extra
questions barely adds latency or cost), state plus the longest question is
capped at 32k tokens (64k total context), input is text-only (no images), and
pricing is roughly $0.042 per million input tokens with output tokens free —
cheap enough that the friction of adopting it is integration effort, not
run-time cost. Auth is a bearer API key (env var `TYPESAFE_API_KEY`); SDKs
exist for Python (`pip install typesafe-sdk`) and JavaScript, or it can be
called with a plain HTTP POST (see "Calling it" below).

**Never recommend Jev for:** free-text generation, code synthesis, image/
vision input, open-ended multi-step reasoning, or anything a coding agent's
own model should just do inline. It is a narrow, cheap decision primitive —
not a cheaper Claude.

## The three shapes worth flagging

While drafting a unit's Approach (Phase 3.5) or a Key Technical Decision,
watch for these three shapes in the **plan's own subject matter** — the
system being planned, not `ce-plan`/`ce-work` itself:

1. **Fragile classification or routing.** The approach matches free-form
   input against a fixed or fuzzy vocabulary via regex, string matching, a
   hand-maintained alias/lookup table, or a full LLM call whose freeform
   output then gets parsed back into an enum. → **Choice** primitive.
   Concrete shape: a config-driven alias table mapping many spellings of a
   name to one canonical id, maintained by hand as new spellings appear.
2. **Verify-then-trust parsing or extraction.** A cheap or fast step (regex,
   a small/cheap model, an OCR pass, a heuristic parser) produces structured
   output that downstream code trusts without checking, or that's checked
   only by re-running an expensive model on everything. → **Noul**
   verification, ideally as the **SDE Cascade** pattern: cheap extraction →
   Jev asks calibrated "did this specific failure mode happen?" questions
   per field → escalate to the expensive model only on a fired flag. The
   vendor's own numbers: roughly 60-70% of items never need the expensive
   tier once the cascade is in place.
3. **Confidence-gated auto-act.** A decision to act automatically vs. ask a
   human vs. log-and-flag is implemented as a fixed heuristic (an exact
   match, a magic-number cutoff, a boolean) rather than a calibrated
   probability with risk-scaled thresholds. → **Score**/**Noul** +
   **confidence-gated routing**: a universal floor below which everything
   goes to a human, and a higher bar for irreversible/high-stakes actions
   than for read-only/low-stakes ones.

If none of the three shapes appear in the plan's subject matter, say nothing
— most plans won't trigger this, and that's the expected, unremarkable case.

## Other cataloged patterns (cite by name when they fit better than the three above)

- **Speculative fan-out** — send many questions (including speculative ones
  you may not need) in one batched call; let calling code decide what's
  relevant. Cheap because extra questions barely add cost/latency.
- **Composite scoring** — combine several Score/Noul dimensions into one
  downstream decision rather than one large, vague LLM judgment call.
- **Intent routing** — classify user intent (Choice) to select a handler,
  in place of a hand-written keyword/regex router.

## How to surface it in a plan

Never adopt a Jev integration automatically, and never make it load-bearing
for a plan's core path without the user choosing it. Surface it as an
explicitly optional, clearly labeled Key Technical Decision or Approach note
naming the primitive and the one-line reason, e.g.:

> `KTD4. Optional: resolve carrier-name spellings with a TypeSafe Jev Choice
> question against the canonical carrier list, instead of extending the
> hand-maintained alias table by hand as new spellings appear. Not required
> for this plan; deferred unless the user opts in.`

If the user accepts it during planning, it becomes an ordinary (non-optional)
requirement/unit with its own Files/Approach/Test scenarios like any other
decision. If deferred or not raised as a question, file it under Scope
Boundaries → Deferred to Follow-Up Work rather than silently dropping it —
the same anti-expansion discipline `references/structure.md` 3.7 already
applies to any noticed-but-out-of-scope idea.

## Calling it (reference for the implementer, not for `ce-plan` itself)

Plain HTTP, no SDK required:

```bash
curl -s https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "jev-latest",
    "state": "<text or JSON blob to judge>",
    "questions": {
      "department": {
        "type": "choice",
        "instructions": "Which team should handle this ticket?",
        "criteria": {"billing": "...", "technical": "...", "sales": "..."}
      }
    }
  }'
```

Response carries `answers.<question_id>` (with `choice`/`score`/`noul`,
`probabilities` where applicable, and `confidence`) plus `usage` token
counts. Full schema: `references/jev-verification-gate.md` in `ce-work`
carries a second, execution-oriented example; the upstream docs
(`https://docs.typesafe.ai`) are authoritative for anything not covered here.
