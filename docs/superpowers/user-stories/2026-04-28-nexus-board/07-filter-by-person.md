# Story 07: Filter timeline by person

**As a** manager
**I want** to focus on a specific person's contributions by toggling their chip in the sidebar
**so that** I can answer "what is X working on across the portfolio" without scanning every rail.

## Why this matters
A scientific portfolio of 3–10 projects with 5–15 people produces a busy timeline (spec §1). The manager regularly needs a single-person lens — for 1:1s, performance reviews, or load-balancing — and the sidebar people filter (spec §3.1, §4) is the primary tool for that.

## Trigger
- Click on a person chip in the sidebar "Filter people" section (spec §3.1, §4, §5).

## UI flow
1. Sidebar renders one chip per `Person` row (spec §2 data model), each in `Person.color` with `Person.initials` (2 chars) as the chip label or avatar.
2. Default state on page load: all chips ON (no filter applied → everything visible).
3. Manager clicks a chip → that chip's ON/OFF state toggles.
4. Timeline updates immediately via client-side re-render; no network request fires.
5. Manager can toggle additional chips to compose a focus set; toggling a chip back ON re-shows that person's nodes.

## Filter semantics — multi-select toggle (PROPOSED)
Spec §4 says: *"Toggle chips per person — hide/show their nodes on all rails."* The phrasing "Toggle chips per person" reads as **per-chip independent ON/OFF** (multi-select), not single-select radio behavior. Proposed semantics:
- Each chip is independent ON/OFF.
- Visible-set = `{ Person : chip is ON }`.
- All ON ⇒ no filter applied; everything visible.
- Some ON ⇒ only nodes whose `person_id ∈ visible-set` are visible.
- All OFF ⇒ no nodes visible (extreme case — see Edge cases).

> ⚠ **Ambiguity (single vs. multi-select):** Spec language is mildly ambiguous. Resolved as multi-select for compositional power; flag for cross-story alignment with Story 08 (filter by node type) which spec §5 explicitly describes as having an "All" chip — implying type-filter is single-select. Person-filter should align unless the manager explicitly chose otherwise.

## Acceptance criteria
- [ ] Toggling a chip immediately shows/hides that person's nodes on **all** project rails (spec §4: *"on all rails"*).
- [ ] Project rails themselves remain visible regardless of filter state (filter is per-node, not per-rail).
- [ ] Today line, week gridlines, and rail-end handles (spec §3.5) remain visible regardless of filter state.
- [ ] Pivot nodes are hidden when their `person_id` is filtered out; the abandoned-branch SVG path remains drawn (see Edge cases).
- [ ] Live Feed (spec §4) is **not** affected by the person filter — it stays portfolio-wide.
- [ ] Sidebar Blockers cards (spec §4) are **not** affected by the person filter — they stay portfolio-wide.
- [ ] Sidebar Upcoming Milestones (spec §4) are **not** affected by the person filter — they stay portfolio-wide.
- [ ] Topbar stats — projects / people / blockers (spec §3.1) — are **not** affected by the filter (stats are global).
- [ ] Active chips have a clear pressed/active visual treatment; inactive (filtered-out) chips are visually muted.
- [ ] No API request is made when toggling chips.

## Visual result
- Filtered-out nodes are **hidden** (removed from the DOM/SVG) rather than greyed out — spec §4 verb is *"hide/show"*.
- Filter state **persists during the session** (e.g. via component state or `sessionStorage`); does **not** survive a hard reload (PROPOSED — see Ambiguity).
- Chip styling matches the existing `.pchip` pattern in the mockup (person color border/bg when ON, muted when OFF).

## Data implications
- **No data change.** Pure client-side UI state.
- No mutation to `Node`, `Branch`, `Project`, or `Person`.

## API touchpoints
- **None.** The filter operates on already-loaded nodes returned from `GET /api/v1/projects/{project_id}/nodes` (spec §6).

## Cross-surface scope
Person-filter scope is **rail-only**, per the spec's literal phrasing in §4: *"hide/show their nodes on all rails."* All sidebar surfaces remain portfolio-wide regardless of filter state.

| Surface | Filter applies? | Source |
|---|---|---|
| Project rail nodes | ✅ Yes | Spec §4 ("on all rails") |
| Live Feed | ❌ No | Out of scope — see below |
| Sidebar Blockers | ❌ No | Out of scope — see below |
| Sidebar Upcoming Milestones | ❌ No | Out of scope — see below |
| Topbar stats | ❌ No | Stats are portfolio-level |
| Search results | ❌ No | Search is global |

## Edge cases
- **All chips OFF:** No nodes visible. Proposed behavior: leave empty (manager can re-toggle); do **not** auto-snap back to all-ON. Flag for review — alternative is to disable the last ON chip's toggle so at least one stays ON.
- **Person assigned to multiple projects:** Their chip controls visibility on **every** project rail simultaneously (spec §2 — Person ↔ Project is many-to-many).
- **Pivot node whose author is filtered out:** The pivot diamond is hidden (consistent with all-nodes-hidden rule). The abandoned branch SVG path, dead-end cap, "ABANDONED APPROACH" label, and vertical connector (spec §3.4) **remain drawn** — they describe project-level structure, not a person's contribution. Flag for review.
- **Soft-deleted person referenced by historical node:** Out of scope (spec does not address person deletion).
- **Newly added person mid-session:** Their chip appears in the sidebar in the default ON state; their nodes (if any) become visible immediately.

## Out of scope
- **Extending the person filter to sidebar surfaces** (Live Feed, Blockers, Upcoming Milestones). Spec §4 scopes the filter to *"nodes on all rails"*. A v2 could revisit this if the manager wants a single-person lens across the whole UI.
- Saving named filter presets ("show only A and B").
- Filtering the timeline by **project** (not in spec — project rails are always all visible).
- Persisting filter state across hard reloads (PROPOSED behavior; revisit if the manager wants it).
- Bulk "select all" / "clear all" buttons on the chip group.
- Keyboard shortcuts for chip toggling.
- URL-encoding the filter state for shareable views (single-user app per spec §1, §8).
