---
name: flutter-skill-consolidate
description: "(Re)build or freshness-audit a consolidated specialist skill from its SOURCES.yaml."
argument-hint: "[target-skill | all] [--discover | --no-discover] [--open-pr]"
disable-model-invocation: true
---

# Skill Consolidate — build/refresh a specialist skill from curated + discovered sources

Keeps a consolidated specialist skill (widgets, async, …) current by re-synthesizing it
from pinned upstream sources **plus** a rot audit that finds new sources and flags dead ones.
Output is one consumable `SKILL.md` (+ `references/`).

> **Not a consuming-project skill.** It edits skill *content* in this repo — don't ship it into
> an app's `.claude/skills/`.

## Inputs
- `target-skill` — the skill to (re)build, e.g. `flutter-widget-expert` or `flutter-code-review`. Its folder must hold a `SOURCES.yaml`. **If omitted, a selection menu is shown (Step 0).** Pass `all` to consolidate every one in turn.
- `--discover` — run Step 1 (Source Audit & Discovery). Default **on**; the freshness guard.
- `--no-discover` — skip discovery; just re-pull the current sources (fast refresh).
- `--open-pr` — finish by opening a PR with the changes instead of leaving them in the tree.

## Process

> **Runs per selected target.** Step 0 picks the target(s); Steps 1–5 then run **once per target**
> (for `all`, loop over every specialist).

> **Audit-only targets.** A `SOURCES.yaml` with `mode: audit-only` marks a skill whose body is
> **hand-curated, not synthesized** (e.g. `flutter-resolve` — a workflow/orchestration skill, not
> distilled domain knowledge). For these, run **Step 1 (rot + discovery)** and **Step 5
> (report)** only — **skip Steps 2–4 and NEVER write the target's `SKILL.md`.** Output is a
> freshness report + a proposed `SOURCES.yaml` diff (retire dead / add candidate sources) plus a
> short "consider adapting" note. A human ports any ideas by hand, selectively.

### Step 0 — Select target(s)
- If a `target-skill` arg was given, use it. If `all` was given, select every specialist.
- **Otherwise, present a menu.** Discover consolidatable skills by scanning for any skill folder
  that contains a `SOURCES.yaml` — under `flutter-specialists/` (the specialist skills) **and**
  `flutter-skills/` (core skills that opt in). Ask via `AskUserQuestion` with one numbered option
  **per skill found**, plus an **"All"** option:
  ```
  Which skill do you want to consolidate?
    1. <skill>                 (one option per folder found with a SOURCES.yaml)
    …
    n. All
  ```
  The list is **built dynamically** from what's on disk — dropping a `SOURCES.yaml` into any
  `flutter-specialists/<name>/` or `flutter-skills/<name>/` makes it appear automatically, no edit here.

### Step 0.1 — Load (for the selected target)
Read `{target}/SKILL.md` + `{target}/SOURCES.yaml` (sources, `domain`, `discovery` config, licenses,
`mode`).

### Step 1 — Source Audit & Discovery  ← the freshness guard (skip with `--no-discover`)

**1a. Rot sweep — kill dead/outdated sources.** For each `status: active` source:
archived? no commits in > 12 months? README says "deprecated / unmaintained"? guidance uses
superseded APIs (e.g. pre-null-safety, pre-Material-3, deprecated `withOpacity`)? → propose `status: retired` **with a reason**.

**1b. Discovery — find newer/better.** Bounded web research:
- `WebSearch` the domain using the `discovery.queries` seeds + the current year, AND check the
  `authority_anchors` (flutter.dev / dart.dev / api.flutter.dev) for guidance changes.
- Collect candidate sources **not already listed**.

**1c. Vet candidates** against explicit criteria — drop anything that fails:
| Criterion | Test |
|---|---|
| Maintained | commits within ~12 months / not archived |
| Authoritative | credible author, strong adoption (stars/pub.dev score), or official |
| Relevant | squarely within `domain` |
| License-compatible | permits derivation + redistribution (else cite-only) |
| Non-duplicate | adds something the active set lacks |
| Actually good | a quick read confirms quality, not SEO filler |

Rank survivors; keep the top few.

**1d. Propose — never auto-add.** Present **sources to retire** (with reason) + **candidates to
add** (with why + license). **Human approves.** Apply approved changes to `SOURCES.yaml`
(`status`, reasons). Discovery never silently changes the source set.

### Step 2 — Fetch
For each `status: active` source: fetch at HEAD (`gh` / clone / `WebFetch`); record `pinned_commit`
+ `last_synced` (pins make re-runs deterministic and upstream changes diffable). Confirm `license`
(fill if `TBD`); anything non-redistributable → **cite/link only**.

### Step 3 — Extract (fan-out)
One sub-agent per source → pull its domain guidance as structured notes: *claim · rationale · code idiom · source*.

### Step 4 — Synthesize
Merge into `{target}/SKILL.md` (overflow detail → `references/`) under the **house style**:
- **Precedence — the `primary: true` source wins.** The official source (Flutter/Dart docs — flutter.dev,
  dart.dev, api.flutter.dev — whichever the skill's `SOURCES.yaml` marks `primary: true`) is the **source of truth**
  and overrides community sources on any conflict, gated to the project's `flutter_version`/`min_sdk`. Community
  sources fill in practical patterns and mistake heuristics, not semantics.
- **Dedupe** overlapping advice; **flag** genuine conflicts in a "Contested" note (don't silently pick).
- **Organize by topic** appropriate to the domain (e.g. for widgets: state, composition, layout, performance, navigation, animation, accessibility, testing).
- **Attribute** each non-obvious recommendation to its source.
- **Header format** — one blockquote only: "**Generated skill** — original wording, consolidated by
  `flutter-skill-consolidate`; provenance, sources, and licenses live in `SOURCES.yaml`. Don't hand-edit —
  change `SOURCES.yaml` and re-consolidate." No source roster or dates in the header — `SOURCES.yaml`
  is the single record.

Done when every extracted claim from Step 3 is either placed in a topic section or explicitly dropped.

### Step 5 — Record + report
Update `SOURCES.yaml` (`last_consolidated`, SHAs, dates, status changes). Emit:
- **Source-audit summary** — retired / added / unchanged (with reasons).
- **Content diff summary** — what changed in the guidance + any contested points.
  - *Audit-only targets:* no content diff; emit the "consider adapting" note instead.

Then **stop for human review** — never auto-commit or auto-push. With `--open-pr`, open a PR;
else leave it in the working tree.

## Modes
- **On-demand:** `flutter-skill-consolidate flutter-widget-expert` (add `--no-discover` for a quick same-sources refresh).
- **Scheduled:** a monthly routine running `--discover --open-pr` so freshness checks surface as PRs you review — keeps skills from rotting without daily babysitting.

## Cost control
"Quick research," not a thesis: cap discovery at a handful of targeted searches + light fetches.
