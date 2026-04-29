# Story 09: Zoom the time axis

**As a** manager
**I want** to switch the timeline between 1W / 1M / 3M / 6M scales
**so that** I can quickly check "what happened this week" or zoom out to "what's the 6-month picture across all projects".

## Why this matters

Scientific projects move at different cadences — some require day-to-day tracking during a sprint while others need a quarterly overview to see pivots and milestones in context. A single fixed scale either crowds short-term detail or hides the long arc; a zoomable axis lets the manager match the lens to the question.

## Trigger

- Click on a zoom pill in the **Controls** bar (spec §3.1, §5)
- Pills sit alongside the type-filter chips at the top of the timeline area

## UI flow

1. Zoom pills are presented as a horizontal pill group: `1W | 1M | 3M | 6M`
2. **Default = 1M** on initial load (spec does not specify — *AMBIGUITY FLAGGED*)
3. The currently active pill is visually highlighted (filled background, others outlined)
4. Clicking a pill rescales the time axis (per spec §5)
5. All project rails update simultaneously — there is one shared time axis at the top of the timeline area (spec §3.1)
6. Week/grid markers (spec §3.2) re-lay out at the new density
7. The "Today" line (spec §3.2) remains visible as a vertical blue marker

## Anchor (PROPOSED: today)

- Visible window **anchors on today's date** — today's blue line stays at a fixed horizontal position within the timeline area
- **Past extent** = window length (1W → 7 days back; 6M → ~180 days back)
- **Future extent** = small forward buffer to keep planned nodes near today legible (PROPOSED: ~25% of window length, so 1W shows ~2 days ahead, 6M shows ~6 weeks ahead) — *AMBIGUITY FLAGGED*
- Rationale: today is the most relevant reference point for an active manager; the alternative anchors (end of timeline / start of timeline) would either hide today on long-running projects or strand the past out of view
- *AMBIGUITY FLAGGED:* spec is silent on whether today sits at center, left-quarter, or right-of-center. Propose **right-of-center** (≈75% across the visible area) so most of the visible width shows the past, where the manager spends most of their attention

## Week marker density (PROPOSED: adaptive)

Per spec §3.2: "Grid lines at each week marker." Literal weekly markers become unreadable at 1W (one mark) and crowded at 6M (~26 marks). PROPOSED adaptive density:

| Zoom | Marker cadence | Approx. markers visible |
|---|---|---|
| 1W | Daily | 7–9 |
| 1M | Weekly | 4–5 |
| 3M | Bi-weekly | 6–7 |
| 6M | Monthly | 6–7 |

This keeps roughly 5–10 markers visible at every zoom level — readable, never crowded. *AMBIGUITY FLAGGED:* exact cadences and label format (e.g. "Apr 12" vs "W15" vs "Apr") are not in the spec.

## Acceptance criteria

- [ ] Four zoom pills are rendered in the Controls bar: 1W / 1M / 3M / 6M
- [ ] Exactly one pill is active at any time (default = 1M on first load)
- [ ] Clicking a pill rescales the visible time window without page reload
- [ ] The "Today" blue line stays at its anchor position across zoom changes (PROPOSED: ~75% across)
- [ ] All project rails share the same time axis and rescale together
- [ ] Grid/week markers re-lay out at adaptive cadence per the table above
- [ ] Project rails truncate visually if their nodes/end-handle fall outside the window (off-screen content is not drawn or is clipped at the viewport edge)
- [ ] Filter-chips state (Story 08) is preserved across zoom changes
- [ ] Person-filter state (Story 07) is preserved across zoom changes
- [ ] No data is mutated when zoom changes — refreshing the page resets to default zoom (no persistence required)

## Interaction with end-handle (Story 10)

- The rail end handle (spec §3.5) is positioned at the project's `projected_end`. When `projected_end` falls outside the visible window, the handle is **off-screen / clipped** and cannot be hovered or clicked
- To access an off-screen handle, the manager zooms out to a window that contains `projected_end`
- *AMBIGUITY FLAGGED:* spec does not describe a "clip indicator" (e.g. an arrow at the right edge hinting that the rail extends beyond view). Propose: rail simply terminates visually at the viewport edge with no indicator — keep v1 minimal

## Data implications

- **No data change.** Zoom is purely a presentation concern; it does not create, modify, or delete any `Project`, `Person`, `Branch`, or `Node` records (spec §2)

## API touchpoints

- **None.** Zoom does not call any route from spec §6
- Initial render uses `GET /api/v1/projects` + `GET /api/v1/projects/{id}/nodes` + `GET /api/v1/projects/{id}/branches` as usual; subsequent zoom changes operate on already-fetched data client-side

## Edge cases

- **Project with `projected_end` far in the future + 1W zoom:** end handle is off-screen. Manager zooms out (e.g. to 6M) to access it
- **Pivot's abandoned branch falls outside window:** abandoned branch is drawn relative to its `fork_date` and the post-fork dead-end (spec §3.4). At narrow zooms, it may be partially visible — the SVG path is clipped at the viewport edge
- **Empty board (no nodes anywhere):** rails still render with their dashed future portion (spec §3.2, §3.6); zoom still works on the empty axis
- **All nodes fall before the visible window** (e.g. only old nodes + 1W zoom): the visible window shows empty rails with the today line and the dashed future portion — this is the correct rendering, no special empty state needed
- **Node date exactly at window boundary:** node renders at the very edge; if half-clipped, this is acceptable for v1
- **Rapid pill clicking:** transitions should debounce or be instant; no animation requirement is stated in spec, propose instant rescale for v1

## Out of scope

- **Custom zoom** (drag-to-zoom, free-scroll panning, scroll-wheel zoom) — spec §5 explicitly lists only the four discrete pills
- **Per-rail zoom** — spec §3.1 shows a single shared time axis at the top of the timeline area; per-rail axes are not in scope
- **Zoom persistence across sessions** — spec is silent; assume default-on-load behavior for v1
- **Vertical zoom** (e.g. compressing rail height) — only horizontal time axis is zoomable
- **Bookmarkable URL state** for the active zoom — out of scope for v1
