# <Feature name> — Spec & Phased Plan

> Fill this out and get it agreed **before** writing code. Delete the italic prompts as you go.
> Standing engineering conventions live in `CLAUDE.md` — don't restate them here. This file is only what's specific to this feature.

## Problem
_One line: what I'm building and for whom._

## Requirements (confirmed)
_Only what was actually asked for. If it isn't in the prompt or the PRD, it belongs under Assumptions._
-

## Assumptions (stated, unconfirmed)
_Things I decided that nobody confirmed. Keeping these visible is the point — they're what a reviewer should challenge first._
-

## Non-goals
_Named explicitly so scope creep has to argue its way in._
-

## Decisions
_Log each real decision with its one-line justification, so it survives review without re-deriving it._

| Decision | Choice | Why |
|---|---|---|
|  |  |  |

## Scope priority (if time-constrained)
_Ordered. The last item is what gets cut first — decide that now, not at minute 50._
1. _core happy path_
2.
3. _nice-to-have, **cut first**_

---

## Architecture

**Pattern:**
**Layers:** UI → ViewModel → Repository (interface) → data source

```
<package/module sketch>
```

**Justification** — _one defensible line each:_
-

### Load-bearing decisions
_The two or three choices the whole design rests on. For each: what it is, and what bug it prevents. If a decision has no failure it prevents, it probably isn't load-bearing._

**1. <name>**

**2. <name>**

### Domain rules
_State machine, validation, invariants. Be exact: this is what the tests assert against._

---

## Phased build order

_Every phase ends green, is independently reviewable, and states how you know it's done. Estimate each phase and check the total against the real budget._

### Phase 0 — Scaffold (~N min)
_Project, dependencies, DI wiring. Nothing else starts until the empty app builds and runs._
**Exit:**

### Phase 1 — Domain + rules (~N min)
_Pure Kotlin. No Android imports. The part the tests actually prove._
**Tests:**
**Exit:**

### Phase 2 — Data layer (~N min)
_Repository interface + implementation/mock._
**Tests:**
**Exit:**

### Phase 3 — Presentation (~N min)
_ViewModel: state derivation, sorting/filtering, loading/empty/error._
**Tests:**
**Exit:**

### Phase 4 — UI (~N min) ← _end-to-end path should complete here_
_All states rendered; actions wired._
**Exit:** _run it and describe what you should see._

### Phase 5 — Harden (~N min, cut first)
_Error surfacing, edge cases, README/architecture justification._

---

## Risks

_For each: the risk, why it bites, the mitigation, and a **tripwire** — a specific clock time or condition at which you change plan. A risk without a tripwire is just a worry._

**<Risk>.** _Impact and mitigation._
**Tripwire:** _if <condition> by <time>, then <fallback>._

---

## Open questions
_Things I chose not to block on. Each has a working assumption so progress continues._
-
