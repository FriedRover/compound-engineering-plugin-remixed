# Jev Verification Gate (Optional)

Opt-in only, off by default. Read this only when `execution-strategy.md`'s
per-unit integration step finds `jev_verification_gate` resolved to `on` (see
Resolution). Otherwise native dispatch runs exactly as `execution-strategy.md`
describes on its own, with no Jev involvement anywhere in the run.

This gate is TypeSafe's own "SDE Cascade" pattern applied to `ce-work`'s
cheap-tier native workers: a cheap/fast worker (`Model: haiku`, see `ce-plan`'s
`references/structure.md` 3.5) does the unit, a near-free Jev check looks for
specific failure modes in what it reported, and only a flagged unit pays for
real re-review. See `ce-plan`'s `references/jev-integration.md` for what Jev
is and its primitives in general; this file is the runnable protocol.

## Resolution

- Config key `jev_verification_gate`: `on` / `off`, default `off`. Read it
  the same way `plan_model` and other per-skill config keys are resolved —
  repo root, `config.local.yaml` then `config.yaml`, ordinary-key rule,
  `#`-prefixed lines ignored. Missing / commented / invalid file selects
  `off`.
- Requires `TYPESAFE_API_KEY` in the environment. If the config says `on`
  but the key is absent, disclose once — "Jev verification gate is
  configured on, but `TYPESAFE_API_KEY` is not set; skipping the gate, native
  dispatch proceeds ungated" — and continue the run without it. Never block
  the run or prompt interactively for a key.
- Scope: gates only units whose resolved `Model:` is `haiku` by default —
  the tier the SDE Cascade pattern targets (cheap, fast, most failure-prone).
  A config list `jev_verification_gate_tiers: [haiku, sonnet]` widens this;
  an absent or empty list means `[haiku]`. Units with no `Model:` field, or
  on `sonnet`/`opus`/`fable` when not listed, are never gated.
- Firing threshold: `jev_verification_gate_threshold`, default `0.4`. Lower
  than the SDE Cascade cookbook's published `0.7` — this default favors
  catching more false negatives, since the cost of over-escalating one cheap
  unit is a single extra review pass, while the cost of under-escalating is
  a wrongly-accepted broken unit reaching a commit.
- Escalation route: `jev_verification_gate_escalation`, one of `reinspect`
  (default) or `rerun` — see Gating logic below.

## When it runs

After a native worker reports a unit complete — serial or as part of a
parallel batch — and *before* the orchestrator commits that unit's changes,
inserted into the existing evidence-capture step that `execution-strategy.md`
already describes ("record the unit's verification evidence," "capture each
worker's returned verification evidence into the run's roll-up"). It never
replaces that step, only gates whether the orchestrator trusts the worker's
self-report at face value before acting on it.

## What it checks

One batched Jev request per gated unit — every question runs in parallel
against the same state, so adding checks costs little:

- `state`: the unit's Goal, its declared `Files:`, a bounded `git diff` over
  those files, and the worker's self-reported evidence (`behavior_changed`,
  tests inspected/added/changed, the red-failure or characterization claim,
  the verification run and result).
- `questions` (all type `noul`; each answer is P(this specific failure mode
  is present), so near 1 means the check fired):
  - `goal_mismatch` — "Does the diff plausibly fail to implement the unit's
    stated Goal, or implement something materially different from it?"
  - `unsubstantiated_red_green` — only when the report claims a red failure
    or characterization baseline was observed before the fix: "Does the
    evidence merely assert this happened, with nothing in the diff or report
    that would let a reader confirm the failure was actually observed first?"
  - `files_incomplete` — "Does the diff leave any file in the unit's declared
    Files list unaddressed, with no reported reason?"
  - `unreported_scope` — "Does the diff touch files or behavior outside the
    unit's declared Files/Goal, with no reason given in the report?"

Omit a question when its premise doesn't apply (e.g., skip
`unsubstantiated_red_green` for a unit with `Test expectation: none`).

## Gating logic

- No question's answer meets the firing threshold → accept the worker's
  report as-is; proceed exactly as `execution-strategy.md` already
  instructs (inspect the actual tree, run authoritative verification,
  commit).
- Any question's answer meets or exceeds the firing threshold → do **not**
  commit the unit's changes on the strength of the self-report alone.
  Escalate per `jev_verification_gate_escalation`:
  - **`reinspect` (default)** — the orchestrator itself reads the actual
    diff and re-derives verification evidence before deciding whether to
    commit, exactly like the "never trust the handoff summary alone"
    discipline `execution-strategy.md` already requires for parallel
    batches — now also applied to a flagged serial unit. If re-inspection
    confirms the work is sound, commit normally and record the false
    positive; if it confirms a real problem, fix before proceeding or treat
    the unit as failed per the ordinary broken-tree handling.
  - **`rerun`** — discard the unit's uncommitted changes and re-dispatch it
    as a fresh worker per the Fresh Worker Invariant, with its `Model:`
    field dropped for this attempt so it inherits the session/orchestrator
    default tier instead of the flagged cheap tier.
  - Never commit a flagged unit's changes without one of these two paths
    completing first, and never silently downgrade a `rerun` escalation back
    to accepting the original cheap-tier output.

## Recording

Add to the unit's verification-evidence record:
`jev_gate: {ran: true|false, fired: [<question ids that fired>], escalation: none|reinspect|rerun}`.
When `ran: false` because the key was absent or the tier wasn't in scope,
still record it so the plan's final verification roll-up is honest about
which units were gated and which weren't.

## Cost note

At Jev's published pricing (~$0.042 per million input tokens, output free),
one gate check over a typical unit's diff and report costs a small fraction
of a cent — negligible next to even one wrongly-accepted broken unit
reaching a commit. This is why the default threshold above is conservative
(low) rather than tuned for minimum gate volume.

## Calling it

Plain HTTP, no SDK dependency added to the target repo — this call is made
by the orchestrator (Claude Code), not by anything installed into the
project being worked on:

```bash
curl -s https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/ce-work-jev-gate-<unit-id>.json
```

Request body shape:

```json
{
  "model": "jev-latest",
  "state": "<goal + files + bounded diff + worker's reported evidence>",
  "questions": {
    "goal_mismatch": {"type": "noul", "instructions": "..."},
    "files_incomplete": {"type": "noul", "instructions": "..."},
    "unreported_scope": {"type": "noul", "instructions": "..."}
  }
}
```

Response: `answers.<question_id>.noul` (0-1 probability the check fired) per
question, plus `usage.{input_tokens,output_tokens}`. Write the request body
to a private temp file rather than inlining a large diff on the command
line.
