# Nexus Board — Design Spec

**Date:** 2026-04-09  
**Stack:** FastAPI full-stack template (https://github.com/fastapi/full-stack-fastapi-template)  
**Ports:** avoid 8000, 8001, 5432, 5433, 5173, 4173, 3000, 3100, 8080+  
**Docker stack name:** `nexus-board`

---

## 1. Purpose

A personal project management board for a single manager overseeing 3–10 scientific research projects with 5–15 people. Scientific projects branch, pivot, and re-prioritize quickly — the UI reflects this with a git-inspired horizontal timeline where pivots are literal branch forks.

**Primary user:** the manager (single user, now). Self-reporting by team members is a planned future feature.

---

## 2. Core Data Model

```
User
  └─ projects[]
       ├─ id, name, color, status (active | completed | paused)
       ├─ projected_end: date          # editable via handle
       ├─ people[]  ←── Person (many-to-many)
       └─ timeline_branches[]
            └─ Branch
                 ├─ id, label, is_active: bool
                 ├─ forked_from_branch_id: uuid | null  # null = main
                 ├─ fork_date: date
                 ├─ abandoned: bool
                 └─ nodes[]
                      └─ Node
                           ├─ id, person_id
                           ├─ date: date
                           ├─ type: update | milestone | blocker | pivot | planned
                           ├─ text: str
                           └─ resolved: bool             # for blockers

Person
  ├─ id, name, initials (2 chars), color: hex
  └─ projects[]  ←── Project (many-to-many, with role per project)
```

**Key invariant:** every `Node` of type `pivot` is the fork point between two `Branch` records — the old (abandoned) and the new (active). The node lives on the new branch at `fork_date`.

---

## 3. UI Architecture

### 3.1 Layout

```
┌─────────────────────────────────────────────────────────────┐
│ Topbar: brand | search | stats (projects / people / blockers)│
├──────────┬──────────────────────────────────────────────────┤
│ Sidebar  │ Timeline area                                    │
│ 240px    │   ┌─ Controls: zoom pills + filter chips ───┐   │
│          │   │                                          │   │
│ Live     │   │  Time axis  Mar──Apr──May                │   │
│ Feed     │   │                                          │   │
│          │   │  Project A ─●──●──●──○──○──┤             │   │
│ People   │   │  Project B ─●──◆──●──┐     │             │   │
│ filter   │   │               ╰──●──╯(dead)│             │   │
│          │   │  Project C ─●──●──●──────┤  │            │   │
│ Blockers │   │                                          │   │
│          │   └──────────────────────────────────────────┘   │
│ Upcoming │                                                   │
│ miles.   │                                                   │
│          │                                                   │
│ Legend   │                                                   │
└──────────┴──────────────────────────────────────────────────┘
```

### 3.2 Timeline Rendering

- One horizontal **rail per project** (not per person)
- Rail is an SVG `<line>` with a glow layer beneath
- Past rail: solid stroke. Future (post-today) rail: dashed + marching-ant animation
- Nodes sit centered on the rail, colored by person
- Horizontal axis: time. Grid lines at each week marker. "Today" line is blue

### 3.3 Node Types

| Visual | Type | Description |
|---|---|---|
| Solid circle (person color) | `update` | A logged status update |
| Solid circle + ★ badge | `milestone` | A significant deliverable |
| Solid circle + ⚠ badge + amber ring | `blocker` | Blocking issue |
| Dashed circle (person color) | `planned` | Upcoming planned work |
| Rotating diamond (amber→red gradient) | `pivot` | Direction change — fork point |
| Red flag icon | `milestone` (deadline) | Hard deadline |

### 3.4 Branch Forks

When a pivot node exists:
1. The **main rail continues** forward from the pivot point (new direction)
2. A **secondary branch** curves away (using SVG cubic bezier), runs horizontally to the fork date, dead-ends with a circle cap
3. Secondary branch is drawn with a dashed stroke, reduced opacity (0.55–0.7)
4. "ABANDONED APPROACH" label rendered as SVG `<text>` below the dead branch
5. A vertical dashed connector links the pivot diamond to the branch fork point

### 3.5 Project End Handle

Each rail terminates with a **drag handle** — a rounded rectangle with a `⋮` grip icon, colored to match the project. On hover: tooltip shows projected end date. Clicking opens an inline date picker. *Drag interaction is a future feature.*

### 3.6 Planned Nodes

Nodes with `type = planned` render as dashed-border circles (same color as person, transparent fill). The future rail behind them is a marching-ant dashed line. Planned milestone flags use a dashed rectangular icon instead of the solid clip-path flag.

---

## 4. Sidebar

| Section | Content |
|---|---|
| Live Feed | Chronological list of recent nodes across all projects (avatar + text + time + project) |
| Filter people | Toggle chips per person — hide/show their nodes on all rails |
| Blockers | Cards for all unresolved `blocker` nodes |
| Milestones | Upcoming milestone deadlines within ~30 days |
| Legend | Node type reference |

---

## 5. Interactions

| Action | How |
|---|---|
| Add update | Click `+` at end of rail → inline form (person, date, text, type) |
| Add planned node | Click beyond today on a rail → inline form with `planned` pre-selected |
| Edit node | Click node → tooltip expands to form |
| Log pivot | Special flow: select person, enter date, enter new direction description → creates `pivot` node + new branch record |
| Extend timeline | Drag the end handle (future feature — handle is visible but static now) |
| Filter | Toggle person chips in sidebar |
| Zoom | 1W / 1M / 3M / 6M pills — rescales the time axis |
| Filter by type | Chips: All / Pivots / Milestones / Blockers / Planned |

---

## 6. API Routes (FastAPI)

```
# Projects
GET    /api/v1/projects
POST   /api/v1/projects
PATCH  /api/v1/projects/{id}
DELETE /api/v1/projects/{id}

# People
GET    /api/v1/people
POST   /api/v1/people
PATCH  /api/v1/people/{id}

# Nodes
GET    /api/v1/projects/{project_id}/nodes
POST   /api/v1/projects/{project_id}/nodes
PATCH  /api/v1/nodes/{id}
DELETE /api/v1/nodes/{id}

# Branches
GET    /api/v1/projects/{project_id}/branches
POST   /api/v1/projects/{project_id}/branches   # created automatically on pivot

# Auth (FastAPI template built-in)
POST   /api/v1/login/access-token
GET    /api/v1/users/me
```

---

## 7. Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Backend | FastAPI (template) | Already chosen |
| DB | PostgreSQL (template) | Already chosen |
| ORM | SQLModel (template) | Already chosen |
| Frontend | React + TypeScript (template) | Already chosen |
| Timeline render | SVG (hand-rolled) | Full control over git-graph visuals; no charting lib gives this |
| Styling | Tailwind (template) + custom CSS vars | Dark theme tokens |
| Auth | JWT (template built-in) | Full auth requested |

---

## 8. Out of Scope (v1)

- Self-reporting by team members (noted for v2)
- Drag-to-extend timeline (handle visible, not interactive)
- Multiple user accounts / roles
- Notifications / reminders
- GitHub / Linear integration
- Mobile layout

---

## 9. Ports & Docker

```yaml
# compose.override.yml (local dev)
services:
  backend:
    ports: ["8010:8000"]   # avoids 8000/8001
  frontend:
    ports: ["5180:5173"]   # avoids 5173/4173
  db:
    ports: ["5440:5432"]   # avoids 5432/5433
```

Stack name: `nexus-board`
