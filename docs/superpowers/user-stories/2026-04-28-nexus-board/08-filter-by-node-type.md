# Story 08: Filter timeline by node type

**As a** manager
**I want** to narrow the timeline to a specific node type (e.g. only Pivots, only Blockers)
**so that** I can answer focused questions ("show me all the pivots in the last 3 months", "where is everything currently blocked?") without visual clutter from routine updates.

## Why this matters
The timeline is dense — 3-10 project rails populated with `update`, `milestone`, `blocker`, `pivot`, and `planned` nodes (spec §3.3). Routine updates dominate the visual field, drowning out the "notable" events the manager needs to act on. Type filters let the manager flip into a question-specific view without losing the rail/temporal context.

## Trigger
- Click a chip in the timeline controls area (spec §3.1: "Controls: zoom pills + filter chips"; spec §5: "Filter by type — Chips: All / Pivots / Milestones / Blockers / Planned")

## UI flow
1. On load, five chips render in the timeline controls strip: **All** / **Pivots** / **Milestones** / **Blockers** / **Planned** (spec §5)
2. Default state: **All** is the active chip; every node renders normally
3. Manager clicks **Pivots** → only `pivot`-type nodes remain visible on rails; **All** deselects, **Pivots** highlights
4. Manager clicks **All** → filter clears, all nodes return
5. Filter state is purely client-side; no network round-trip

## Filter semantics (PROPOSED: single-select)
- Exactly one chip is active at any time — click is a radio-button selection, not a toggle-into-set
- **All** = no type filter (sentinel "off" state)
- Click any non-**All** chip = filter narrows to that single type only
- Re-clicking the currently active non-**All** chip → returns to **All** (PROPOSED — provides an obvious "exit" gesture)
- **AMBIGUITY FLAGGED:** spec §5 lists the chip set but does not state single- vs multi-select. Multi-select (e.g. "Pivots ∪ Blockers") would be more flexible but adds visual ambiguity (which chips are selected? what's the combinator?). Single-select is simpler and sufficient for the listed use cases. Revisit if multi-select becomes necessary.

## Interaction with the person filter (Story 07)
- Filters compose with logical **AND**
- A node is visible iff: `(passes person filter)` **AND** `(passes type filter)`
- Both filters operate on the same client-side node set; neither changes data

## Acceptance criteria
- [ ] All five chips render in the controls strip per spec §5; **All** is selected on first load
- [ ] Selecting **Pivots** shows only nodes with `type == "pivot"` (spec §2 data model)
- [ ] Selecting **Milestones** shows only nodes with `type == "milestone"` — both ★-badge variant and red-flag (deadline) variant per spec §3.3
- [ ] Selecting **Blockers** shows only nodes with `type == "blocker"`, including both resolved and unresolved (filter is by type, not by `resolved` field)
- [ ] Selecting **Planned** shows only nodes with `type == "planned"` (dashed circle visual per spec §3.6)
- [ ] `update` nodes are visible only under **All** — there is no Updates chip per spec §5
- [ ] **Abandoned-branch SVG visibility** (PROPOSED — see Cross-spec interaction): the curved bezier path, dead-end circle cap, "ABANDONED APPROACH" label, and vertical dashed connector (spec §3.4) render under **All** and **Pivots**; they are hidden under **Milestones**, **Blockers**, **Planned**
- [ ] Today line, project rails (solid past + dashed future), and end handles (spec §3.5) always render regardless of chip selection — these are project-level structure, not nodes
- [ ] Topbar stats (projects / people / blockers count) are unaffected by chip selection — stats reflect data state, not view state
- [ ] **Live Feed** (sidebar §4): PROPOSED — does **not** filter by chip; the feed has its own scope (chronological recent activity across all projects). **AMBIGUITY FLAGGED:** spec does not say. Argument for honoring chip: visual consistency. Argument against: the feed is a summary widget independent of the timeline lens, and forcing it to follow the chip would surprise users who want to scroll for context while focusing the timeline.
- [ ] **Sidebar Blockers cards** (§4): unaffected by type chip — they reflect the unresolved-blocker dataset, independent of timeline view
- [ ] **Sidebar Upcoming Milestones** (§4): unaffected by type chip — same rationale
- [ ] Composing with person filter: visible nodes pass both predicates (Story 07 `AND` Story 08)
- [ ] Active chip has a distinct selected style; inactive chips are dimmer

## Cross-spec interaction: abandoned-branch visibility
This is the trickiest part of the story.

**The structural fact (spec §3.4):** an abandoned branch is the visual rendering of a `Branch` record (data model §2), not of a single node. It consists of: curved bezier fork, horizontal dashed segment, dead-end circle cap, "ABANDONED APPROACH" SVG `<text>`, vertical dashed connector linking the pivot diamond down to the fork point.

**The question:** when filtering by node type, do these structures remain?

**PROPOSED rule:**
- Under **All**: all structures visible.
- Under **Pivots**: structures visible. The abandoned branch is intrinsic to a pivot's meaning — hiding it would orphan the pivot diamond and break the spec §2 invariant ("every Node of type pivot is the fork point between two Branch records") visually.
- Under **Milestones / Blockers / Planned**: structures hidden. The user has explicitly asked to focus on those node types; pivot scaffolding is noise.

**Alternatives considered:** keep abandoned-branch structures always visible (would clutter Blockers view); hide always except under Pivots (would lose pivot context under All — wrong default).

**FLAGGED for review.** If the maintainer disagrees, single-line change in the rendering layer.

## Visual result
- Filtered-out nodes: hidden (not greyed). Nothing renders in their slots
- Rails, today line, end handles, week gridlines, project labels: unchanged
- Active chip: highlighted (border + background per the design tokens)
- Empty state on a rail (no matching nodes for the chip): rail still renders normally, just sparse — see Edge cases

## Data implications
- **No data change.** Pure client-side filter state, mirroring Story 07's person filter.
- No `Node.hidden` field, no soft-delete, no API parameter. Purely a render-time predicate.

## API touchpoints
- **None.** No spec §6 endpoint is invoked by chip interaction.
- Initial node fetch (`GET /api/v1/projects/{project_id}/nodes`) returns the full set; the client applies the predicate.

## Edge cases
- **Selecting Blockers with no blockers anywhere:** rails render with today line, end handles, gridlines — but no node markers. Empty-set-but-not-broken state.
- **Selecting Pivots with no pivots:** rails render normally, no diamonds, no abandoned-branch structures. (Trivially: there are none to draw.)
- **Selecting Pivots with one pivot but no abandoned branch yet** (data malformation — should not happen per §2 invariant): pivot diamond renders alone. Treat as data-integrity issue, not a filter concern.
- **Selecting Planned where every planned node has a date ≤ today** (e.g. all overdue/converted): empty timeline (filter-by-type, not filter-by-date — past planned nodes still match the type predicate). Their visual (dashed circle) renders against the **solid** past-rail segment, which looks slightly off but is acceptable; they should be promoted to `update` as soon as the work is logged.
- **Combining type filter with person filter:** intersection. E.g. "Pivots" + only "Alice" toggled on → only pivots authored by Alice (`person_id == alice.id`).
- **Switching chips rapidly:** purely a state update; no flicker, no debounce required.
- **Zoom (Story 09) interaction:** orthogonal. Zoom rescales the time axis; type filter narrows visible node types. Both compose freely.

## Out of scope
- Custom-defined filter sets (e.g. "Pivots + Blockers")
- Persisting the chip selection across page reloads
- A "saved filter" feature
- Filtering by `resolved` (e.g. "only unresolved blockers") — that's a sub-attribute, not a node type. The sidebar Blockers section already covers unresolved-only.
- Filtering by date range beyond what zoom (Story 09) already provides
- Server-side filtering (entire dataset is small enough for client-side per spec §1: 3-10 projects, 5-15 people)
- Multi-select chips (revisit if needed; see Filter semantics)
