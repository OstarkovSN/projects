# Story 03: Record a blocker

**As a** manager
**I want** to log a blocker against a project so it surfaces in the sidebar and topbar count
**so that** I can see at a glance what's stuck and act on it before it stalls a project.

## Why this matters
Blockers are the highest-attention items in the portfolio — a stuck person on one project is a leading indicator of slipped milestones. Surfacing them in two persistent locations (topbar count, sidebar cards) means the manager sees the problem without having to scan rails.

## Trigger
Click the `+` handle at the end of a project rail → inline form opens (per spec §5, "Add update" pattern). The manager selects `blocker` from the type dropdown, defaulting the form into blocker-creation mode.

## UI flow
1. Manager clicks the `+` at the right end of the project's rail.
2. Inline form appears anchored to the rail end with fields: person, date, text, type.
3. Manager selects type = `blocker`.
4. Manager picks the person who is blocked (must already be assigned to this project per spec §2 Person↔Project many-to-many).
5. Manager picks a date (defaults to today; see edge case below).
6. Manager writes the blocker description in the `text` field.
7. Manager confirms (e.g. presses Enter or clicks "Save").
8. Form closes, the blocker node appears on the rail at the chosen date, the topbar count increments, the sidebar Blockers card list grows by one, and the Live Feed shows a new entry.

## Form fields
- `person` — selected from the project's assigned people (spec §2)
- `date` — date of the blocker
- `text` — description of what is blocked
- `type` — fixed at `blocker` for this story
- `resolved` — defaults to `false`, NOT user-set at creation (spec §2 lists `resolved: bool # for blockers`; resolution belongs to Story 04)

## Acceptance criteria
- [ ] Blocker renders on the project rail with solid circle (person's color) + ⚠ badge + amber ring (spec §3.3)
- [ ] Topbar blocker count (spec §3.1: "stats (projects / people / blockers)") increments by 1
- [ ] Sidebar Blockers section (spec §4) shows a card for the new blocker
- [ ] Sidebar Live Feed (spec §4) shows the new blocker entry chronologically (avatar + text + time + project)
- [ ] Persisted Node has `type=blocker` and `resolved=false`
- [ ] Node lives on the project's currently active branch (spec §2 — Node belongs to a Branch; the active branch is the default target)
- [ ] Blocker is hidden when the manager filters out that person via the people-filter chip (spec §4)
- [ ] Blocker is hidden when the manager applies the "Pivots / Milestones / Planned" type chip (and shown when "All" or "Blockers" is selected) per spec §5
- [ ] Form validation: `text` cannot be empty, `person` and `date` must be set
- [ ] Cancelling the form (e.g. Escape) closes it without creating a node

## Visual result
- Solid circle in the person's color (spec §2: Person.color hex)
- ⚠ badge overlay
- Amber ring around the circle
- Centered on the rail at the X position corresponding to its date (spec §3.2)

## Data implications
Creates one Node row (spec §2):
- `id` — generated
- `person_id` — FK to Person
- `date` — chosen date
- `type` — `blocker`
- `text` — entered description
- `resolved` — `false`

The Node attaches to the project's currently active Branch (the one with `is_active=true`). No Branch records are created or modified.

## API touchpoints
- `POST /api/v1/projects/{project_id}/nodes` (spec §6) — request body carries `person_id`, `date`, `type=blocker`, `text`. Server sets `resolved=false` and assigns the node to the active branch.
- After success, the client re-reads or patches local state for: the rail render, Live Feed, Blockers sidebar card list, topbar count.

## Edge cases
- **Multiple unresolved blockers on the same project:** each is its own Node and its own sidebar card (spec §4 says "Cards for all unresolved `blocker` nodes" — plural). Topbar count reflects all of them.
- **Blocker dated in the future:** ambiguous in the spec. A blocker is by definition something already happening — future-dated work is the `planned` type (spec §3.3, §3.6). **Proposed rule:** disallow blocker dates strictly greater than today; surface a validation error in the form. **Flagged as ambiguity** for cross-story consistency review.
- **Blocker on a person not assigned to this project:** the form's person dropdown only lists people on the project's `people[]` (spec §2 many-to-many). Out-of-project people are not selectable; this is enforced client-side and should also be validated server-side.
- **Network failure on save:** form remains open, error surfaced inline; no optimistic insert.
- **Past-dated blocker (back-fill):** allowed — the manager may be logging something that started yesterday or last week. The node still appears unresolved until Story 04 closes it.

## Out of scope
- Resolving a blocker (Story 04)
- Editing the `text` or `person` of an existing blocker (general node-edit, spec §5 "Edit node")
- Deleting a blocker (spec §6 has `DELETE /api/v1/nodes/{id}` but is a separate concern)
- Auto-converting unresolved blockers to pivots (not in spec)
- Notifications or reminders about open blockers (spec §8 explicitly out of scope)
- Self-reporting by the blocked person themselves (spec §1, §8 — v2 only)
