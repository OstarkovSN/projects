# Story 10: Adjust a project's projected end date via the end handle

**As a** manager
**I want** to update a project's projected end date by clicking its rail-end handle and picking a new date
**so that** the timeline reflects realistic timelines as the project's scope evolves.

## Why this matters
Scientific research projects re-prioritize quickly (spec §1). The projected end date is the manager's working forecast — keeping it current is the lightweight equivalent of "re-baselining" without ceremony, and the rail-end handle puts that edit one click away from the very rail it adjusts.

## Trigger
- Hover the rail-end handle → tooltip shows current `projected_end` (spec §3.5)
- Click the handle → inline date picker opens (spec §3.5)

## Reconciling spec §3.5 vs §8
Spec §3.5 says: hover shows a tooltip; clicking opens an inline date picker; *drag* is a future feature. Spec §8 lists "Drag-to-extend timeline (handle visible, not interactive)" as out of scope, with a parenthetical that reads as if the entire handle is non-interactive.

**Resolution for v1:** the parenthetical in §8 is misleading. Only the **drag** affordance is out of scope; **hover (tooltip) and click (date picker) are interactive in v1** per the explicit, more detailed §3.5 description and the §5 "Edit node" pattern of click-to-open. The handle is *partially* interactive: clickable, hoverable, not draggable.

## UI flow
1. Manager hovers handle → tooltip appears showing current `projected_end` (e.g. "End: 2026-06-15")
2. Manager clicks handle → inline date picker overlays at the handle's position
3. Manager picks a new date in the picker
4. Manager confirms (Enter or click "Save") → request fires; rail length and handle position re-render
5. On error → date picker shows inline validation message; nothing persists until valid

## Form fields
- Single field: new `projected_end` date (date picker, no time component — `date` per spec §2)

## Acceptance criteria
- [ ] Hovering the handle shows a tooltip with the current `projected_end`, formatted as a human-readable date (e.g. "End: 2026-06-15")
- [ ] Clicking the handle opens an inline date picker positioned at the handle
- [ ] The date picker is pre-populated with the current `projected_end`
- [ ] Submitting a new date issues `PATCH /api/v1/projects/{id}` with the updated `projected_end` (spec §6)
- [ ] On success, the rail re-renders with the new length and the handle moves to the new horizontal position
- [ ] On success, any rail-overlay computations that depend on `projected_end` (e.g. future/marching-ant region clipping per §3.2) update accordingly
- [ ] Pressing `Esc` or clicking outside the picker dismisses it without saving
- [ ] If the new `projected_end` is in the past (< today): **disallow** — picker shows inline error "Projected end cannot be in the past." Rationale: a past date conflicts with the meaning of "projected"; completion is tracked via `Project.status = completed` (§2), not by backdating the end. *Flagged for product confirmation.*
- [ ] If the new `projected_end` is before any existing node's `date` on the project (across all branches): **warn but allow** — picker shows "This will leave N node(s) past the rail end" with confirm. Rationale: the manager may be intentionally shrinking scope; downstream visuals can render the orphaned nodes hanging past the handle, or be clipped at the handle by the renderer. *Flagged for product confirmation.*
- [ ] Handle visual style is unchanged: rounded rectangle, `⋮` grip icon, project color (spec §3.5)
- [ ] Drag is **not** wired up — mousedown-and-drag on the handle does nothing (or shows the same tooltip), per §3.5/§8

## Visual result
- Handle remains a rounded rectangle with `⋮` grip icon, colored to match the project (spec §3.5)
- Rail length updates to reach the new handle position on the time axis
- The boundary between solid (past) rail and dashed marching-ant (future) rail (spec §3.2) is unaffected by this change — only the rail's right end moves

## Interaction with zoom (Story 09)
- The handle's horizontal position is computed from `projected_end` against the current zoom window (1W / 1M / 3M / 6M, spec §5)
- If the new `projected_end` falls outside the current zoom window, the handle renders off-screen at the right edge or is clipped — the manager would zoom out (e.g. 3M → 6M) to see it
- The date picker itself is unaffected by zoom (it accepts any date)

## Data implications
- Exactly one `UPDATE` on the `Project` row, mutating `projected_end`
- No changes to `Branch`, `Node`, or `Person` records
- No new rows created

## API touchpoints
- `PATCH /api/v1/projects/{id}` (spec §6) with body `{ "projected_end": "YYYY-MM-DD" }`

## Edge cases
- **`projected_end` < today:** disallow (see acceptance criteria). Project completion is modeled via `status = completed` (§2), not via past `projected_end`.
- **`projected_end` < latest existing node date:** warn-and-confirm. The manager may be deliberately shrinking the project; visuals beyond the handle are a renderer concern.
- **`projected_end` equal to current value:** no-op; close picker without firing PATCH.
- **Concurrent edits:** not a concern — single user (spec §1, and §8 lists "Multiple user accounts / roles" as out of scope).
- **Project with `status = completed` or `paused`:** still editable; status is independent of `projected_end` per the §2 data model. (Whether a completed project should expose the handle at all is a UX decision for a future story.)
- **Network/API failure:** picker stays open and shows an inline error; no optimistic UI commit until 200 OK.

## Out of scope
- Drag-to-extend the rail (spec §3.5: "Drag interaction is a future feature"; spec §8 confirms)
- Auto-suggesting a `projected_end` from existing planned nodes
- Notifications/reminders as `projected_end` approaches (spec §8: "Notifications / reminders" out of scope)
- Editing `projected_end` from anywhere other than the handle (e.g. a project settings panel) — out of scope for this story
- Bulk-editing multiple projects' end dates
- Showing a history/audit trail of past `projected_end` changes
- Hiding the handle for completed/paused projects (UX decision deferred)
