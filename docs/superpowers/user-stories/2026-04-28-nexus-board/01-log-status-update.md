# Story 01: Log a status update on a project rail

**As a** manager
**I want** to quickly log a progress update for one of my team members on a project
**so that** I have an at-a-glance record of activity across all projects without writing reports.

## Why this matters
Overseeing 3-10 scientific projects with 5-15 people means constant context-switching. The manager needs a friction-light way to capture what was reported in a 1:1 or standup, so the timeline stays current without becoming a second job.

## Trigger
Manager clicks the `+` button at the end of the relevant project rail (spec §5: "Add update"). The button sits just before the project end handle described in spec §3.5.

## UI flow
1. Manager identifies the right project rail in the timeline area.
2. Clicks `+` at the end of the rail (past the most recent node, before the end handle).
3. Inline form expands in place, anchored to the rail.
4. Manager picks the person from a dropdown limited to people assigned to this project (per spec §2 — `Project.people[]` many-to-many).
5. Date defaults to today; manager can adjust if backfilling.
6. Manager types the update text (single line, free-form).
7. Type is preset to `update` but the dropdown is selectable (per spec §5 form fields: "person, date, text, type").
8. Manager submits (Enter or explicit Save).
9. Form collapses; node appears on the rail at the chosen date in the person's color (spec §3.3).
10. Live Feed sidebar (spec §4) prepends a new entry referencing this node.

## Form fields
- **person** — dropdown of `Person` records linked to this project
- **date** — defaults to today; date picker
- **text** — free-form short text (single line)
- **type** — defaulted to `update`, dropdown over the five valid types from spec §2 (`update | milestone | blocker | pivot | planned`). Note: selecting `pivot` from this form is disallowed because pivots use a special flow (spec §5: "Log pivot — Special flow"); see edge cases.

## Acceptance criteria
- [ ] Clicking `+` at end of rail opens inline form anchored to that rail
- [ ] Person dropdown only shows people assigned to this project (not all people in the system)
- [ ] Date field defaults to today
- [ ] Submitting creates a Node row with `type=update` and `resolved=null/false` (resolved is for blockers per spec §2)
- [ ] Node renders as a solid circle in the person's color (spec §3.3, row 1)
- [ ] Node appears on the **active** branch of the project (spec §2: `Branch.is_active`)
- [ ] New node appears in the Live Feed sidebar (spec §4) chronologically
- [ ] Empty text submission is rejected with inline validation (inferred — spec doesn't explicitly state, flag for confirmation)
- [ ] Future-dated entries with `type=update` are rejected; the form should suggest switching to `type=planned` instead (inferred — spec §3.6 designates `planned` as the future-work type, flag for confirmation)

## Visual result
- Solid circle in the person's color, centered on the project rail at the chosen date (spec §3.3)
- Node sits on the **past** (solid stroke) section of the rail, since updates describe what already happened (spec §3.2)
- New entry prepended to Live Feed: avatar + text + time + project (spec §4)
- No change to the topbar stats counters (those track projects/people/blockers per spec §3.1, not updates)

## Data implications
Per spec §2:
- Creates one `Node` row with: `person_id`, `date`, `type='update'`, `text`, `resolved=null` (resolved only meaningful for blockers)
- The node belongs to the project's currently `is_active=true` branch (spec §2 — there is exactly one active branch per project)
- No new `Branch` or `Person` rows created
- No change to the parent `Project` record

## API touchpoints
Per spec §6:
- `POST /api/v1/projects/{project_id}/nodes` — create the update node
- The Live Feed in the sidebar is populated by reading nodes across all projects; spec §6 exposes `GET /api/v1/projects/{project_id}/nodes` per-project. Live Feed assembly happens client-side or through a backend aggregation endpoint not yet listed (flag for confirmation).

## Edge cases
- **Future-dated update** — Inferred answer: reject. Spec §3.6 designates `planned` for future-dated work, and §3.2 says "Past rail: solid stroke. Future (post-today) rail: dashed". A solid `update` circle on a dashed future rail would be visually inconsistent. **Ambiguity:** spec doesn't explicitly forbid this — confirm with user.
- **Empty text** — Inferred answer: reject with validation. The Live Feed and node tooltip are useless without text. Spec doesn't explicitly require non-empty text — flag for confirmation.
- **Person not assigned to this project** — Should not appear in dropdown at all (filter at the source). If somehow submitted, backend should reject with 422.
- **Project is `paused` or `completed`** — Inferred answer: still allow updates (managers may log retroactive notes on completed projects). Spec §2 lists these as valid statuses but doesn't constrain node creation by status.
- **No active branch on the project** — Should be impossible per spec invariant (every project has exactly one active branch); if it occurs, surface an error rather than silently creating an orphan node.
- **Date before project's earliest existing node** — Allowed; nodes are not inherently ordered beyond their `date` field.

## Out of scope
- Self-reporting by the team member directly (spec §8 — planned for v2)
- Multi-line rich text, attachments, mentions (spec describes `text: str`, no rich formatting)
- Notifications when an update is logged (spec §8 — notifications out of scope)
- Mobile layout (spec §8)
- Editing the node — covered by a separate edit interaction (spec §5: "Edit node — Click node → tooltip expands to form")
- Deleting the node — uses `DELETE /api/v1/nodes/{id}` from spec §6, not part of this story
