# Cross-Story Review — Nexus Board User Stories

**Reviewed:** 2026-04-28
**Stories reviewed:** 10
**Spec reviewed:** `/home/claude/workdirs/projects/docs/superpowers/specs/2026-04-09-nexus-board-design.md`

## Summary

- **Contradictions found: 2**
- **Spec violations: 1** (essentially the same issue as C1, called out separately for compliance tracking)
- **Consistently-noted open ambiguities (not contradictions): ~22**

The stories are largely consistent with each other. Most spec ambiguities are acknowledged identically across stories or are scoped to a single story. The two real contradictions are both about the **scope of sidebar surfaces under filtering / activity rules** — places where the spec uses one short phrase (e.g. "on all rails", "recent nodes") and stories interpret it differently.

---

## Contradictions

### C1: Does the person filter apply to sidebar lists, or only to project rails?

- **Story 02** (`02-mark-milestone.md`, line 80): "Person filter hides the milestone: it disappears from the rail; the **sidebar Upcoming Milestones section may still show it** (TBD — spec §4 doesn't say person filter applies to sidebar; default: **rail-only, sidebar always shows**)."
- **Story 07** (`07-filter-by-person.md`, lines 35–37, 58–60): explicit acceptance criteria — "**Live Feed (spec §4) applies the same filter** — only entries authored by visible people appear (PROPOSED). Sidebar Blockers cards (spec §4) **apply the same filter**. Sidebar Upcoming Milestones (spec §4) **apply the same filter**."
- **Why it matters:** This is a load-bearing UX rule. If a manager filters by person, the entire sidebar either re-scopes to that person (Story 07) or stays portfolio-wide (Story 02). Story 04 also reads sidebar Blockers as the live "what's stuck right now" surface — its meaning differs sharply between the two interpretations. Implementing both stories as written would leave the team with no consistent rule.
- **Spec position:** §4 row "Filter people" reads: *"Toggle chips per person — hide/show their nodes on all rails."* The phrase **"on all rails"** is scoping language that points to rail-only. The spec is otherwise silent on sidebar behavior.
- **Recommended resolution: Align with Story 02 (rail-only).** It matches the spec's literal phrasing. Story 07 should be revised: drop the "applies to Live Feed / Blockers / Upcoming Milestones" acceptance criteria, keep only the rail effect, and move the sidebar-filter idea to "Out of scope (could be revisited if requested)." The cross-surface scope table in Story 07 should be updated to reflect rail-only.

### C2: Live Feed — does it include future-dated nodes (e.g. future milestones, planned nodes)?

- **Story 02** (`02-mark-milestone.md`, line 24, AC line 41): For *any* new milestone — including future-dated ones — "the new node appears in the sidebar Live Feed." No past-only carve-out.
- **Story 06** (`06-plan-future-work.md`, lines 51–52): Explicit AC: "**The node is not added to the sidebar Live Feed** (proposed; flag as ambiguity — spec §4 says 'recent nodes', which we interpret as **past-only**)."
- **Why it matters:** The two stories pick mutually exclusive readings of `recent` in spec §4. If Live Feed is past-only (Story 06's reading), future-dated milestones from Story 02 wouldn't appear there either — directly contradicting Story 02's AC. If Live Feed shows everything recent (Story 02's reading), planned nodes from Story 06 should appear there too — contradicting Story 06's AC.
- **Spec position:** §4 row "Live Feed" reads *"Chronological list of recent nodes across all projects."* "Recent" is ambiguous between "happened lately" (past-only) and "lately added or scheduled" (all-times).
- **Recommended resolution: Past-only (align with Story 06).** Three reasons:
  1. Stories 01 (update) and 03 (blocker) already reject future-dated entries entirely (their proposed rule: past-only); the past-only Live Feed reading is consistent with that direction of the system.
  2. The Live Feed's UX role is "what just happened" — future-scheduled work is best surfaced via the Upcoming Milestones sidebar (deadlines specifically) and the dashed future-rail itself.
  3. Story 02 should be revised: future-dated milestones do **not** appear in Live Feed; they appear on the rail and (if `is_deadline` and ≤30 days out) in Upcoming Milestones. Past milestones still appear in Live Feed as today.

---

## Spec Violations

### V1: Story 07 extends "Filter people" beyond the spec's rail-only scope

- **Story:** `07-filter-by-person.md` (lines 35–37 and the cross-surface scope table at lines 54–64)
- **What the story says:** Person-filter chips affect Live Feed, Sidebar Blockers, and Upcoming Milestones in addition to project rail nodes.
- **What the spec says:** §4 row "Filter people": *"Toggle chips per person — hide/show their nodes on all rails."* The scope qualifier is **"on all rails"**, not "everywhere on the page."
- **Recommended fix:** Same as C1's recommended resolution — restrict person-filter effect to project rails. Move sidebar-list filtering to "Out of scope" and note it as a v2 candidate. This realigns Story 07 with the spec and resolves C1 simultaneously.

(Stylistic note: this is the same issue as C1, surfaced separately under "Spec Violations" because it isn't only an inter-story disagreement — Story 07 has gone past what the spec authorizes.)

---

## Consistently-Noted Open Ambiguities (NOT contradictions, no fix needed)

These are flagged identically (or non-contradictorily) across stories; resolution should come from the spec author, not from the fixer agent.

**Schema additions (proposed but unconfirmed):**
- `is_deadline: bool` on Node — proposed by Story 02; Story 08 references the deadline visual variant without proposing a different mechanism. Consistently flagged.
- `resolved_at: datetime | null` on Node — Story 04 explicitly flags as an open question; not adopted.

**Date constraints (each story scopes its own type):**
- Future-dated `update` rejection (Story 01).
- Future-dated `blocker` rejection (Story 03).
- Future-dated `pivot` rejection (Story 05).
- `planned` requires date > today (Story 06).
- All four are consistent: past = `update`/`blocker`/`pivot`; future = `planned` (or future-dated `milestone` for hard deadlines per Stories 02 and 06, agreed).

**UI entry points spec didn't specify:**
- Pivot trigger UI surface (Story 05) — proposed fork-glyph button or kebab menu.
- Sidebar Blockers card "Mark resolved" affordance (Story 04).
- Default zoom level (Story 09: proposed 1M).
- "Today" anchor position in zoom (Story 09: proposed ~75% across).
- Future buffer in zoom window (Story 09: proposed ~25%).
- Adaptive grid-marker cadence (Story 09: daily/weekly/bi-weekly/monthly).

**Visual states spec didn't specify:**
- Resolved blocker visual on rail (Story 04: proposed keep ⚠ badge, drop amber ring).
- Future non-deadline milestone visual (Story 02: proposed unchanged solid circle + ★).
- Abandoned-branch SVG visibility under type-filter chips (Story 08: proposed visible under "All" + "Pivots" only).

**Behavioral edge cases:**
- All person chips OFF (Story 07).
- Future-dated nodes on a freshly-abandoned branch (Story 05).
- Planned-node-conversion on date passing today (Story 06: manual edit, no automation).
- `projected_end` < today (Story 10: disallow).
- `projected_end` before latest existing node (Story 10: warn-and-confirm).
- Pivot atomicity through `POST /nodes` (Story 05).

**Filter selection mode (acknowledged, not contradicted):**
- Person chips: multi-select (Story 07).
- Type chips: single-select with "All" sentinel (Story 08).
- Both stories acknowledge the asymmetry. Spec §4 ("Toggle chips per person") supports multi for people; spec §5 (lists "All" as a chip) supports single for types. Asymmetry is intentional, not a contradiction.

---

## Verification Status

- [x] All 10 stories read in full
- [x] Spec read in full
- [x] Cross-referenced data fields (`is_deadline`, `resolved`, `resolved_at`, `branch_id`, `projected_end`)
- [x] Cross-referenced API endpoints (POST/PATCH/DELETE on `/nodes`, `/projects`, `/branches`)
- [x] Cross-referenced filter semantics (person multi-select, type single-select, AND-composition)
- [x] Cross-referenced visual rules (§3.3 node visuals, §3.4 branch fork, §3.5 end handle, §3.6 planned)
- [x] Cross-referenced sidebar surfaces (Live Feed, Blockers, Upcoming Milestones, Filter people)
- [x] Cross-referenced future-dated policy across types

---

## Re-verification (2026-04-29)

### Status of previously-flagged issues

- **C1 / V1 (person filter scope):** **RESOLVED.** Story 07 now restricts the person filter to project rails only:
  - Acceptance criteria lines 35–38 explicitly state Live Feed, Sidebar Blockers, Sidebar Upcoming Milestones, and Topbar stats are **not** affected.
  - Cross-surface scope table (lines 54–65) flips Live Feed / Blockers / Upcoming / Topbar / Search to ❌ No, citing spec §4 "on all rails".
  - Out-of-scope section (line 74) deferes cross-surface filtering to a v2 candidate.
  - Multi-select semantics, all-OFF edge case, and pivot-author edge case all preserved unchanged.

- **C2 (Live Feed past-only):** **RESOLVED.** Story 02 now treats Live Feed as past-only:
  - UI flow step 8 (line 24): past-dated → appears in Live Feed; future-dated → does not, with explicit cross-reference to Story 06.
  - Acceptance criteria lines 41–42 codify the same split into two bullets.
  - All other Story 02 surfaces (Upcoming Milestones, rail rendering, type-filter chip behavior, person-filter respect on rail) are unchanged and remain consistent with Stories 06, 07, and 08.

### New contradictions or regressions found

**None substantive.** Both fixes are clean. Cross-checks performed:

- **Story 01 (update) ↔ Story 02 fix:** Story 01 already rejects future-dated updates (lines 40, 61). Past-only Live Feed is consistent.
- **Story 03 (blocker) ↔ Story 02 fix:** Story 03's rule that blockers must be past-or-today aligns with past-only Live Feed.
- **Story 06 (planned) ↔ Story 02 fix:** Story 06 line 51 was the canonical past-only reading; Story 02 now cross-references it explicitly. No conflict.
- **Story 07 fix ↔ Story 08 (filter by type):** Story 08's "Live Feed / Blockers / Upcoming Milestones unaffected by chip" rule (lines 42–44) was already aligned with the spec — Story 07's fix now matches that broader "sidebar is portfolio-wide" stance.
- **AND-composition of filters:** Story 08 line 28–30 still composes person filter (multi-select rail) AND type filter (single-select rail) on the **rail**. Both fixes preserve this; no regression in filter math.
- **Story 02 ↔ Story 07 cross-reference (person filter on sidebar Upcoming Milestones):** Story 02 line 81 was written before the fix and still uses "TBD" wording for whether the person filter applies to the sidebar Upcoming Milestones section. Its **proposed default** ("rail-only, sidebar always shows") happens to match Story 07's post-fix rule, so this is **not** a contradiction — it is a stale "TBD" annotation that could be tightened in a future polish pass. Flagged as documentation quality, not a contradiction.

### Open ambiguities (unchanged by the fixes)

The ~22 cross-story open ambiguities catalogued above (schema additions, date constraints, UI entry points, visual states, behavioral edge cases, filter selection mode) are **untouched** by the two fixes. They remain valid and still require spec-author resolution.

### Re-verification status

- [x] Story 02 re-read (with focus on fixed sections — lines 23–24, 41–42 verified)
- [x] Story 07 re-read (with focus on fixed sections — lines 35–38, 54–65, 74 verified)
- [x] Other 8 stories re-read for regressions (01, 03, 04, 05, 06, 08, 09, 10)
- [x] Cross-story interactions re-checked (filter composition, Live Feed scope, edit flows, abandoned-branch visibility)
- [x] Spec re-read; no spec violations remain
