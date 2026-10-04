# This is a personal fork

This repo is [jmatthewhouse](https://github.com/FriedRover)'s fork of
[`EveryInc/compound-engineering-plugin`](https://github.com/EveryInc/compound-engineering-plugin)
(the upstream marketplace also referenced by its old alias,
`EveryInc/every-marketplace`), deliberately renamed to
**`compound-engineering-plugin-remixed`** — both the GitHub repo and the
Claude Code marketplace name declared inside it — so it's never mistaken
for, and never collides with, a stock upstream install under the shared
plugin name `compound-engineering-plugin`. It carries two custom features
on top of upstream: **per-unit model selection** and an **optional
TypeSafe/Jev integration**. Everything else — all 36 skills, docs, install
paths for other hosts — is unmodified upstream content. See the
[README](README.md) for what Compound Engineering itself does.

If an agent has just been pointed at this repo's URL and told to "install
this," follow **Install** below, then read **The customizations** so you
know what's different from stock Compound Engineering.

## Install (Claude Code)

```text
/plugin marketplace add https://github.com/FriedRover/compound-engineering-plugin-remixed.git
/plugin install compound-engineering@compound-engineering-plugin-remixed
```

Or from a shell:

```bash
claude plugin marketplace add https://github.com/FriedRover/compound-engineering-plugin-remixed.git
claude plugin install compound-engineering@compound-engineering-plugin-remixed
```

Restart the Claude Code session afterward — plugin installs and updates
require a restart to take effect.

**Do not** install from `EveryInc/compound-engineering-plugin` /
`every-marketplace` directly — that's stock upstream and lacks the
customizations below. Always point at this fork's URL. Because the
marketplace name itself differs (`compound-engineering-plugin-remixed` vs.
upstream's `compound-engineering-plugin`), the two can even be registered
side by side without conflict — unlike before this rename, when adding this
fork under the same name as an existing upstream registration would be
refused outright (see "Why renamed" below).

Other hosts (Cursor, Codex, etc.): follow the per-host sections in the
[README](README.md#install), substituting this fork's URL/owner wherever
the README names `EveryInc/compound-engineering-plugin`. Note that this
rename only changed the Claude Code marketplace identity
(`.claude-plugin/marketplace.json`); the other hosts' own marketplace
manifests (`.cursor-plugin/`, `.grok-plugin/`, etc.) are untouched upstream
content and still declare upstream's names.

### Refreshing after this fork gets updated

Same two-step order the upstream project documents in
[`docs/install/upgrading.md`](docs/install/upgrading.md) — refresh the
marketplace snapshot *before* updating the plugin, or `update` reads a
stale cached snapshot and no-ops:

```text
/plugin marketplace update compound-engineering-plugin-remixed
/plugin update compound-engineering
```

Then restart.

### Why renamed

Originally this fork kept upstream's exact marketplace name
(`compound-engineering-plugin`) and GitHub repo name, just with a different
URL/owner. That caused two sharp edges: (1) `claude plugin marketplace add
<this-fork>` on a machine that already had the stock plugin registered
failed outright with "its network source differs from the one declared for
it in settings," and (2) nothing about the install made it visually obvious
you were on a customized fork rather than stock Compound Engineering. The
`-remixed` suffix on both the repo and the declared marketplace name fixes
both: it's a distinct marketplace identity that can coexist with upstream,
and it's self-evidently not the stock plugin wherever its name appears
(`/plugin marketplace list`, `enabledPlugins` keys, this URL).

The **plugin id itself stays `compound-engineering`** (unchanged) — only
the marketplace half of `plugin@marketplace` changed. This plugin still
is Compound Engineering; it's the distribution channel that's marked as
remixed.

## The customizations

### 1. Per-unit model selection

`ce-plan` can annotate an Implementation Unit with an optional `Model:`
field, and `ce-work` reads it when dispatching that unit's native
subagent, so different units in the same plan can run on different Claude
model tiers instead of all inheriting the session model.

**Where it lives:**
- `skills/ce-plan/references/structure.md` (Phase 3.5) — the field
  definition, the four tiers, and the all-or-nothing rule
- `skills/ce-plan/references/plan-sections.md` — one-line pointer to it
  from the unit-field summary
- `skills/ce-work/references/execution-strategy.md` — native dispatch
  reads `Model:` and passes it as the `model` override on the
  `Agent`/`Task` dispatch call

**The four tiers, in brief** (full definitions and examples in
`structure.md` 3.5 — that file is authoritative if this summary drifts):

| Alias | Use for |
|---|---|
| `haiku` | Mechanical, low-judgment work: boilerplate, renames, config/data scaffolding, repetitive multi-file edits with no design decisions |
| `sonnet` | **The baseline tier.** Ordinary feature work with clear requirements — typical CRUD/API/UI/service logic, standard test-writing, moderate multi-file coordination that never rises to heavy interdependency or correctness-critical reasoning. Most units belong here. |
| `opus` | Heavy interdependencies, non-obvious architecture, or correctness-critical logic: concurrency, migrations, security boundaries, subtle-edge-case algorithms, or a unit whose approach touches many other units' contracts |
| `fable` | The unit's output is primarily prose/narrative/creative generation rather than code |

**All-or-nothing per plan:** once any unit gets a `Model:` field, every
unit gets one — including explicit `Model: sonnet -- <reason>` on
baseline units. An absent field on some-but-not-all units would leave a
reader unable to tell "sonnet on purpose" from "not yet triaged," so
`ce-plan` never does that. A plan where nothing warrants a non-default
tier carries no `Model:` fields at all.

This is unrelated to, and does not interact with, the existing
reasoning-elevation (`ce-plan`'s `references/reasoning-elevation.md`) or
cross-model execution (`ce-work`'s `references/cross-model-execution.md`)
contracts — those route a single authoring step, or a whole unit, to a
different external harness/CLI entirely; this feature only varies which
Claude model tier serves an ordinary in-harness native subagent.

### 2. Optional TypeSafe/Jev integration

[TypeSafe's Jev](https://docs.typesafe.ai/introduction) is a "System One"
model — not a text/code generator, a narrow, very cheap (~$0.042/M input
tokens) model that answers typed Choice/Score/Noul questions against text
you give it and returns calibrated probabilities. Two independent,
both-optional hooks, neither on by default:

**Advisory detector (`ce-plan`, no runtime dependency).** While drafting a
unit's Approach or Key Technical Decisions, `ce-plan` watches for three
shapes in the *plan's own subject matter* (not in `ce-plan`/`ce-work`
itself): fragile classification/routing against a fixed or fuzzy
vocabulary, verify-then-trust parsing/extraction with no real check, or a
confidence-gated auto-act decision built as a fixed heuristic. When one
appears, it reads `skills/ce-plan/references/jev-integration.md` and
surfaces an explicitly optional, clearly labeled KTD naming the matching
Jev primitive — never adopted automatically. Most plans won't trigger this
at all; that's expected. Since this only ever produces plan *text*, it adds
no runtime dependency to `ce-plan` itself.

**Verification gate (`ce-work`, a real runtime dependency, config-gated
`off`).** When config key `jev_verification_gate` resolves to `on` (repo
`config.local.yaml`/`config.yaml`, off by default) and `TYPESAFE_API_KEY`
is set, `ce-work` runs TypeSafe's own "SDE Cascade" pattern on cheap-tier
(`Model: haiku`) unit reports before trusting them: a batch of Noul checks
("does the diff plausibly implement the Goal," "is any declared file left
unaddressed," etc.) against the worker's diff and self-report. Any check
firing above the threshold (default `0.4`) blocks a silent commit and
either re-inspects the unit itself or re-dispatches it to a stronger tier,
per `skills/ce-work/references/jev-verification-gate.md`. With the config
key absent/`off` (the default) or the API key unset, `ce-work` never
contacts TypeSafe at all.

**Where it lives:**
- `skills/ce-plan/references/jev-integration.md` (new) — what Jev is, the
  three shapes to watch for, other cataloged patterns, how to phrase the
  optional KTD, and a plain-`curl` call example
- `skills/ce-plan/references/structure.md` (Phase 3.5b, new) — the hook
  that tells `ce-plan` when to read the guide above
- `skills/ce-work/references/jev-verification-gate.md` (new) — the
  runnable protocol: config resolution, the exact Noul checks, gating
  logic, escalation paths, recording, and the `curl` call
- `skills/ce-work/references/execution-strategy.md` — the config-resolution
  pointer plus the two integration-step hooks (serial and parallel) that
  call the gate before a commit, when active

Neither hook interacts with the per-unit-model-selection customization
above beyond reading its `Model:` field to decide gate scope, and neither
touches the existing reasoning-elevation or cross-model-execution
contracts.

## Keeping this fork in sync with upstream

This fork is exactly `upstream/main` plus one commit (kept up to date in
place — amend it rather than stacking more commits on top, so there is
always exactly one commit to replay; the commit message documents both
customizations above, and the rename below). Two remotes are configured in
the working clone at
`~/.claude/plugins/marketplaces/compound-engineering-plugin-remixed` (this
is Claude Code's own marketplace clone — its path is derived from the
declared marketplace name, so it moved here when the marketplace was
renamed):

- `origin` → this fork (`FriedRover/compound-engineering-plugin-remixed`)
- `upstream` → `EveryInc/every-marketplace`

**`claude plugin marketplace remove <name>` deletes that clone's directory
outright** (confirmed when this fork was renamed — the pre-rename clone at
the old path vanished the moment its marketplace entry was removed, even
though everything in it had already been pushed to `origin`). If you ever
remove-then-re-add this marketplace again — the only reason to do so is a
rename or a from-scratch reset — a fresh `marketplace add` reclones from
`origin` with only an `origin` remote; `upstream` will need re-adding:

```bash
cd ~/.claude/plugins/marketplaces/compound-engineering-plugin-remixed
git remote add upstream https://github.com/EveryInc/every-marketplace.git
git fetch upstream
```

An ordinary rebase (below) never removes the marketplace, so this only
matters after a deliberate remove/re-add.

### When to rebase

Rebase when any of these are true — there's no fixed schedule, this is a
low-traffic personal fork:

- **You want an upstream fix or new skill/feature** and it postdates your
  last sync (check `CHANGELOG.md` at the tip of `upstream/main` for what
  landed).
- **Something in `ce-plan` or `ce-work` misbehaves** and it might be a bug
  upstream already fixed — sync before spending time working around it.
- **It's been a while** (a few weeks+) and you're about to lean on this
  plugin for a big planning/work session — cheap insurance against being
  meaningfully behind.
- **A rebase attempt reports conflicts in a touched file** — that means
  upstream restructured the unit-field docs or the native-dispatch section;
  resolve it once, deliberately, rather than letting it silently drift out
  of sync. The two files this fork adds outright (`jev-integration.md`,
  `jev-verification-gate.md`) can't conflict; the risk is entirely in the
  three existing upstream files this fork also edits (`structure.md`,
  `plan-sections.md`, `execution-strategy.md`) plus, now, the one-line
  rename in `.claude-plugin/marketplace.json` — `README.md` is deliberately
  never touched by this fork, precisely to avoid that conflict surface.

Check whether you're behind without changing anything:

```bash
cd ~/.claude/plugins/marketplaces/compound-engineering-plugin-remixed
git fetch upstream
git log --oneline main..upstream/main   # non-empty = upstream has moved
```

### How to rebase

```bash
cd ~/.claude/plugins/marketplaces/compound-engineering-plugin-remixed
git fetch upstream
git rebase upstream/main
# Resolve conflicts if git stops (most likely in structure.md,
# plan-sections.md, execution-strategy.md, or the marketplace.json name —
# re-read the surrounding upstream text and re-apply the same intent from
# "The customizations"/"Why renamed" sections rather than blindly keeping
# "ours"; the two new jev-*.md files can't conflict since upstream doesn't
# have them).
git push --force-with-lease origin main
```

Then refresh the installed plugin (**Refreshing after this fork gets
updated** above), and re-verify both customizations *and* the rename
survived:

```bash
grep -n "All-or-nothing per plan" skills/ce-plan/references/structure.md
grep -n "Per-unit model selection" skills/ce-work/references/execution-strategy.md
grep -n "3.5b Optional: TypeSafe/Jev Opportunity Check" skills/ce-plan/references/structure.md
grep -n "Optional Jev verification gate" skills/ce-work/references/execution-strategy.md
grep -n '"name": "compound-engineering-plugin-remixed"' .claude-plugin/marketplace.json
test -f skills/ce-plan/references/jev-integration.md && echo "jev-integration.md present"
test -f skills/ce-work/references/jev-verification-gate.md && echo "jev-verification-gate.md present"
```

If any of these come back empty/missing after an update, the merge/rebase
dropped that customization (or reverted the rename) — reapply it from this
file's **The customizations** / **Why renamed** sections before trusting
`ce-plan`/`ce-work` output.

**Note on the installed-plugin cache:** Claude Code's plugin runtime loads
from a pinned, extracted copy under
`~/.claude/plugins/cache/compound-engineering-plugin-remixed/<version>/`
(the cache directory name follows the marketplace name, so it moved when
the marketplace was renamed), not from this git checkout directly. The
marketplace/update commands above are what refresh that cache from this
fork's `main`. If you ever hand-patch the cache directly for a
same-session fix (as was done once, in-band, before this rename, back when
the cache still lived under the old `compound-engineering-plugin` path),
remember the fork is still the source of truth — reconcile the cache back
to whatever `main` says on the next refresh.
