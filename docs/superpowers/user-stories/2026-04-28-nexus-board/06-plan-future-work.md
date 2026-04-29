# Story 06: Plan future work with planned nodes

**As a** manager
**I want** to add tentative future work as planned nodes beyond today on a project rail
**so that** I can sketch what should happen next, distinct from already-completed work.

## Why this matters
The rail is not just a record of what happened — it's also a planning surface. The manager
needs to mark *intended* upcoming work for a person on a project so the team's next steps
are visible at a glance, without committing to it as a hard milestone.

## Trigger
- Spec §5: "Click beyond today on a rail → inline form with `planned` pre-selected"
- Concretely: hover/click on the dashed future-rail region of any project (the rail to the
  right of the blue "Today" line per spec §3.2). The inline form opens with `type=planned`
  pre-filled. The form is the same shape as the Add update form (spec §5).

## UI flow
1. Manager hovers anywhere on the future-rail region of a project (right of the Today line).
   A `+` affordance appears at the cursor's position on the rail.
2. Manager clicks. An inline form opens, pre-populated with:
   - `type` = `planned` (locked or pre-selected — see ambiguity below)
   - `date` = the date inferred from the click x-position on the time axis
   - `person` = blank (manager picks)
   - `text` = blank
3. Manager picks a person from the project's `people[]` (spec §2 — only people assigned to
   this project should be selectable).
4. Manager optionally edits the date (must remain > today — see ambiguity below).
5. Manager types a short text description.
6. Manager submits. A `POST /api/v1/projects/{project_id}/nodes` request is sent (spec §6).
7. On success, the new node renders immediately on the dashed future rail at the chosen date.

## Form fields
- `person_id`: required, must be a `Person` linked to this `Project` (spec §2).
- `date`: required, must be > today (proposed; see ambiguity below).
- `text`: required, free-form short string.
- `type`: locked to `planned` for this flow.

## Acceptance criteria
- [ ] Form is reachable by clicking the future-rail region of a project (spec §5).
- [ ] Form opens with `type` pre-selected as `planned` (spec §5).
- [ ] Submission requires non-empty `person_id`, `date`, and `text`.
- [ ] `date` must be > today (proposed; flag as ambiguity — spec doesn't state this).
- [ ] Person selector is restricted to the project's assigned people (spec §2).
- [ ] On successful submit, the node appears as a **dashed-border circle** in the person's
      color with **transparent fill** (spec §3.6), centered on the rail at the chosen date.
- [ ] The future rail behind the node is rendered as a marching-ant dashed line (spec §3.6).
- [ ] The new node respects active person and type filters (spec §4 + §5):
      hidden when its person is filtered off, hidden when `Filter by type` is set to a
      non-`Planned`/non-`All` chip.
- [ ] The node is **not** added to the sidebar Live Feed (proposed; flag as ambiguity —
      spec §4 says "recent nodes", which we interpret as past-only).
- [ ] The node is **not** added to "Upcoming milestones" (spec §4 — that section is for
      `milestone` deadlines only, not `planned` nodes).
- [ ] Clicking the new node opens the standard edit-node form (spec §5).

## Visual result
- Dashed-border circle, person's color, transparent fill (spec §3.6).
- Sits on the dashed marching-ant future rail (spec §3.6).
- Person's color is consistent with all other nodes for that person across rails.
- No badge, no flag — `planned` is a circle-only visual (per the §3.3 row).

## Disambiguation: planned vs future-dated milestone
- **This story covers `type=planned` nodes only** — informal, tentative future work
  (the dashed-circle visual in spec §3.3 / §3.6).
- A future-dated **milestone** (with hard deadline) is a separate concept and is covered by
  Story 02 (Mark a milestone). Per spec §3.6 it renders as a **dashed rectangular flag**
  (not a solid clip-path flag), but its `type` is still `milestone` — not `planned`.
- **Interpretation chosen:** `planned` is its own type used for tentative upcoming work.
  Future-dated milestones remain `type=milestone` and are styled dashed because the rail
  itself is dashed beyond today. The `planned` type and `milestone` type never overlap on
  the same node — a node is one or the other, not both.
- **Flagged ambiguity:** the spec lists the types as a flat enum
  (`update | milestone | blocker | pivot | planned`) and never combines `planned` with
  `milestone`. An alternative interpretation would treat `planned` as an orthogonal flag,
  but this would contradict the enum shape in spec §2 — so we reject that interpretation.

## Data implications
- Creates one `Node` row (spec §2) with:
  - `type = planned`
  - `date > today`
  - `person_id` set
  - `resolved` left at default (the field is "for blockers" per spec §2 — N/A here)
  - parent `branch_id` = the project's currently-active `Branch` (spec §2:
    `is_active: true, abandoned: false`).
- No `Branch` rows are created by this flow.
- No changes to `Project` or `Person`.

## API touchpoints
- `POST /api/v1/projects/{project_id}/nodes` (spec §6) — create the node.
- `PATCH /api/v1/nodes/{id}` (spec §6) — used for the edit-node flow if the manager later
  opens this node and changes anything.
- `GET /api/v1/projects/{project_id}/nodes` (spec §6) — already returns this node along
  with all others; consumers must respect `date > today` to render it on the future rail.

## Edge cases
- **Date ≤ today on submit:** propose rejection ("planned work must be in the future"),
  since past tentative work is better expressed as an `update`. **Flag as ambiguity** —
  spec doesn't state this constraint.
- **Date pushed past `Project.projected_end`:** allow it; the rail will need to extend
  visually to cover the node, but `projected_end` is an editable hint (spec §2, §3.5),
  not a hard ceiling. Adjusting `projected_end` is a separate flow (Story 10).
- **Editing a planned node's date to ≤ today:** if the manager edits the date to past-or-
  today, propose preserving `type=planned` rather than auto-converting. Conversion is a
  manual edit-node action. **Flag as ambiguity.**
- **Converting a planned node into an `update` once the work is done:** propose this is
  the standard edit-node flow (spec §5) — the manager opens the node and changes
  `type` from `planned` to `update`. No automation. **Flag as ambiguity.**
- **Planned node on an abandoned branch:** if the parent branch is later abandoned via a
  pivot (spec §3.4), the planned node is hidden from rendering — abandoned branches are
  drawn only up to their dead-end (spec §3.4: "runs horizontally to the fork date,
  dead-ends with a circle cap"). The node row is preserved in the DB.
- **No people assigned to the project:** the form can't be submitted; show a hint to add
  a person to the project first (spec §2 — Project has `people[]`).
- **Click overlaps an existing planned node:** the click should select that node for edit
  (spec §5: "Click node → tooltip expands to form"), not start a new add flow.

## Out of scope
- Drag-to-reorder planned nodes along the timeline.
- Auto-conversion of `planned` → `update` when the date passes today (must be a manual edit).
- Recurring planned items / templates.
- Notifications when a planned date approaches (spec §8: notifications out of scope).
- Bulk creation of multiple planned nodes at once.
- Self-reporting by team members (spec §8).
