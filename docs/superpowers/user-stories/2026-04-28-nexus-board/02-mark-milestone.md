# Story 02: Mark a milestone (with optional hard deadline)

**As a** manager
**I want** to mark significant deliverables — and flag the ones tied to hard deadlines
**so that** I can see what's important at a glance and not miss commitments.

## Why this matters
Scientific projects produce occasional, high-signal deliverables (a published result, a working prototype, a frozen dataset) that are qualitatively different from day-to-day status updates. Some of those carry hard external commitments — a grant deadline, a conference submission — and those need to surface earlier and more loudly than ordinary milestones, so the manager doesn't lose track of them between weekly reviews.

## Trigger
Same entry point as a status update (spec §5):
- Click the `+` at the end of a project rail to open the inline form.
- In the form, change `type` to `milestone`.
- For a milestone scheduled beyond today, the manager may instead click on the rail beyond today (spec §5 "Add planned node"); the form opens with `planned` pre-selected and the manager changes `type` to `milestone` (data implication discussed below — a future-dated milestone is still `type=milestone`, not `type=planned`).

## UI flow
1. Manager clicks `+` at the end of the target project rail (spec §5).
2. Inline form appears with fields: person, date, text, type (default `update`).
3. Manager selects `type = milestone`.
4. A conditional `is_deadline` checkbox (or toggle) appears with helper text: "Hard deadline — surfaces in Upcoming Milestones."
5. Manager fills in the date, picks the responsible person, writes the description, optionally checks `is_deadline`.
6. On save, a POST to `/api/v1/projects/{project_id}/nodes` (spec §6) creates the Node.
7. The new node appears on the project rail at its date; if `is_deadline` and within ~30 days of today, it also appears in the sidebar's "Upcoming milestones" section (spec §4).
8. If the milestone date is ≤ today, the Live Feed (spec §4) shows the new node chronologically. Future-dated milestones do **not** appear in the Live Feed (Live Feed is past-only — see Story 06 for the shared interpretation of spec §4 "recent nodes").

## Form fields
- `person` — who owns the deliverable; required (Person reference, spec §2).
- `date` — when it lands; required.
- `text` — short description of the deliverable; required.
- `type` — fixed at `milestone` for this flow.
- `is_deadline: bool` — **PROPOSED**. The spec's Node schema (§2) lists `id, person_id, date, type, text, resolved`. There is no `is_deadline` field, yet §3.3 distinguishes a "Red flag icon" deadline-milestone visually from a "★ badge" milestone. The simplest way to drive that distinction is a boolean on Node. **Flagged for confirmation** with the spec author. Alternative considered: a separate `type` value (e.g. `deadline`) — rejected because spec §3.3 explicitly lists the type as `milestone (deadline)`, i.e. a sub-flavor of milestone, not a sixth top-level type, and §5 "Filter by type" lists Milestones as one bucket without splitting deadlines out.

## Acceptance criteria
- [ ] From the `+` form, selecting `type=milestone` reveals an `is_deadline` toggle (proposed field).
- [ ] Saving with `is_deadline=false` creates a Node rendered as a solid circle in the person's color with a ★ badge (spec §3.3).
- [ ] Saving with `is_deadline=true` creates a Node rendered as a red flag icon (spec §3.3).
- [ ] If the milestone date is in the future and `is_deadline=true`, the flag is rendered with a dashed rectangular icon variant (spec §3.6).
- [ ] If the milestone date is in the future and `is_deadline=false`, the rendering remains solid circle + ★ badge; per spec §3.6 the dashed variant only applies to milestone *flags* (i.e. deadline milestones) — see "Edge cases" for the open question.
- [ ] A milestone with `is_deadline=true` and `date` within ~30 days of today appears as a card in the sidebar "Upcoming milestones" section (spec §4).
- [ ] A milestone with `is_deadline=false` does NOT appear in the sidebar "Upcoming milestones" section.
- [ ] When a past-dated milestone (`date` ≤ today) is created, it appears in the sidebar Live Feed (spec §4).
- [ ] When a future-dated milestone (`date` > today) is created, it does **not** appear in the sidebar Live Feed — the Live Feed is past-only (consistent with Story 06's interpretation of spec §4 "recent nodes").
- [ ] The "Filter by type" chip "Milestones" (spec §5) toggles visibility of all `type=milestone` nodes regardless of `is_deadline`.
- [ ] The new node respects the active person filter (spec §4) — hidden when its person's chip is off.

## Visual result

### Non-deadline milestone (default)
- Solid circle in the responsible person's color, with a ★ badge overlaid (spec §3.3).
- Sits centered on the project rail at its `date` (spec §3.2).

### Deadline milestone
- Red flag icon (spec §3.3 — note: spec was updated from "green" to "red" flag).
- Replaces the circle+star treatment.

### Future deadline milestone
- Dashed rectangular flag icon, replacing the solid clip-path flag (spec §3.6).
- Sits on the dashed marching-ant future rail (spec §3.2, §3.6).

## Sidebar surfacing
- Deadline milestones (`is_deadline=true`) within ~30 days of today surface in the "Upcoming milestones" sidebar section (spec §4).
- Non-deadline milestones do **not** surface in that section. Spec §4 explicitly scopes that section to "Upcoming milestone deadlines"; ordinary milestones still appear via the Live Feed and on their rail.

## Data implications
- Creates one Node row with `type=milestone`, `person_id`, `date`, `text`.
- **PROPOSED extra field**: `is_deadline: bool` on Node, defaulting to `false`. Confirm with spec owner before implementation.
- Spec §2 invariant addresses pivots specifically — milestones are *not* fork points and do not require Branch records. A new milestone is appended to whatever Branch is `is_active=true` for the project.
- The `resolved` field on Node (spec §2) is documented "for blockers" and is not used by milestones.

## API touchpoints
- `POST /api/v1/projects/{project_id}/nodes` to create the milestone (spec §6).
- `GET /api/v1/projects/{project_id}/nodes` to refetch / hydrate the rail (spec §6).
- No Branch endpoints are touched — milestones never create branches (spec §2 invariant scopes branching to pivots).

## Edge cases
- **Convert update → milestone**: covered by the edit-node flow (spec §5: "Click node → tooltip expands to form"). Out of scope here.
- **Past milestone with no deadline**: rendered as solid circle + ★ badge on the past (solid) rail; never surfaces in Upcoming Milestones.
- **Future milestone without `is_deadline`**: open question — spec §3.6 only specifies a dashed variant for deadline flags, leaving the future-non-deadline case ambiguous. Default interpretation: render as solid circle + ★ badge (same as past, no visual change). The dashed marching-ant rail underneath provides the "future-ness" cue. Flag for spec owner if a dashed-star treatment is wanted.
- **Future milestone with `is_deadline`**: dashed rectangular flag icon (spec §3.6). The rail underneath remains the dashed marching-ant future rail.
- **Milestone exactly on today**: the spec doesn't specify; align with however the rail's solid/dashed split is computed (treat today as the split point — visual is whichever side `today` lands on).
- **Person filter hides the milestone**: it disappears from the rail; the sidebar Upcoming Milestones section still shows it. Per spec §4 ("hide/show their nodes on all rails") and Story 07, the person filter is rail-only — sidebar surfaces remain portfolio-wide.
- **Milestone deleted via `DELETE /api/v1/nodes/{id}`** (spec §6): removed from rail, Live Feed, and Upcoming Milestones if it was there.

## Out of scope
- Reminder / notification when a deadline approaches (spec §8: notifications/reminders excluded from v1).
- Self-reporting by team members marking their own milestones (spec §1, §8: planned for v2).
- Drag-to-extend timeline to make room for far-future deadlines (spec §3.5, §8: drag is future feature; click-to-edit on the project end handle is the v1 path).
- Automatic conversion of a past `planned` node into a `milestone` — different node types, no auto-promotion specified.
- Per-project milestone summary view — only Live Feed and Upcoming Milestones surfaces are specified (spec §4).
- Customizing the 30-day Upcoming window — spec §4 says "~30 days"; treat as a constant for v1.
