# Enterprise PPM Dashboard — Build Specification

**Stack:** Angular (frontend) · Java/Spring Boot (backend) · MongoDB (database)
**Status:** Phase/Milestone/Task/Risk tables with inline create/edit already exist. Dashboard tab with 5 tiles (Analytics, Phase, Milestone, Task, Risk) already exists as a shell. This spec defines the data model those tables should conform to, and the full behavior required for the Analytics tab.

---

## 1. Hierarchy

```
Project
 └── Phase
      └── Milestone
           └── Task
                └── Risk (risks may also link directly to a Phase or Milestone, not only a Task)
```

Every Phase belongs to exactly one Project. Every Milestone belongs to exactly one Phase. Every Task belongs to exactly one Milestone. Every Risk links to exactly one parent entity (Task, Milestone, or Phase) via a polymorphic reference — see §3.5.

---

## 2. Data Model

### 2.1 Project (assume this collection already exists; listed for completeness)

| Field | Type | Required | Notes |
|---|---|---|---|
| `_id` | ObjectId | auto | |
| `name` | String | Yes | |
| `description` | String | No | |
| `owner` | Reference → User | Yes | |
| `status` | Enum | Yes | See §3.1 |
| `startDate` | Date | Yes | |
| `endDate` | Date | Yes | |

### 2.2 Phase

| Field | Type | Required | Notes |
|---|---|---|---|
| `_id` | ObjectId | auto | |
| `projectId` | Reference → Project | Yes | |
| `name` | String | Yes | |
| `description` | String | No | |
| `category` | Enum | No | See §3.2 — used for grouping/reporting, not workflow logic |
| `owner` | Reference → User | Yes | |
| `status` | Enum | Yes | See §3.1 |
| `priority` | Enum | Yes | See §3.3 |
| `plannedStartDate` | Date | Yes | Baseline |
| `plannedEndDate` | Date | Yes | Baseline |
| `actualStartDate` | Date | No | Null until phase starts |
| `actualEndDate` | Date | No | Null until phase completes; used as **Forecast End** while in progress if a forecast field isn't separately tracked |
| `percentComplete` | Number (0–100) | Yes | Manual entry OR auto-rolled-up from child milestones — **decide one and document it**; recommend auto-rollup, computed server-side |
| `ragOverride` | Enum | No | Red / Amber / Green — optional manual override of the system-computed RAG (see §4) |
| `createdAt` / `updatedAt` | Date | auto | System-managed |

### 2.3 Milestone

| Field | Type | Required | Notes |
|---|---|---|---|
| `_id` | ObjectId | auto | |
| `phaseId` | Reference → Phase | Yes | |
| `name` | String | Yes | |
| `description` | String | No | |
| `type` | Enum | No | See §3.4 |
| `owner` | Reference → User | Yes | |
| `status` | Enum | Yes | See §3.1 |
| `priority` | Enum | Yes | See §3.3 |
| `plannedDueDate` | Date | Yes | Baseline |
| `achievedDate` | Date | No | Null until achieved |
| `ragOverride` | Enum | No | Red / Amber / Green |
| `createdAt` / `updatedAt` | Date | auto | |

### 2.4 Task

| Field | Type | Required | Notes |
|---|---|---|---|
| `_id` | ObjectId | auto | |
| `milestoneId` | Reference → Milestone | Yes | |
| `name` | String | Yes | |
| `description` | String | No | |
| `type` | Enum | No | See §3.5 |
| `assignee` | Reference → User | Yes | The "owner" of a task |
| `status` | Enum | Yes | See §3.1 |
| `priority` | Enum | Yes | See §3.3 |
| `percentComplete` | Number (0–100) | Yes | Manual entry |
| `plannedStartDate` | Date | Yes | Baseline |
| `plannedEndDate` | Date | Yes | Baseline |
| `actualStartDate` | Date | No | |
| `actualEndDate` | Date | No | Forecast end while in progress |
| `dependsOn` | Array of Reference → Task | No | Predecessor task IDs, for dependency-line rendering in the Gantt |
| `ragOverride` | Enum | No | Red / Amber / Green |
| `createdAt` / `updatedAt` | Date | auto | |

### 2.5 Risk

| Field | Type | Required | Notes |
|---|---|---|---|
| `_id` | ObjectId | auto | |
| `linkedEntityType` | Enum | Yes | `Phase` \| `Milestone` \| `Task` — polymorphic parent type |
| `linkedEntityId` | ObjectId | Yes | ID of the Phase/Milestone/Task this risk is attached to |
| `name` | String | Yes | Risk title |
| `description` | String | No | |
| `category` | Enum | Yes | See §3.6 |
| `owner` | Reference → User | Yes | |
| `probability` | Enum | Yes | Low / Medium / High — see §3.7 |
| `impact` | Enum | Yes | Low / Medium / High — see §3.7 |
| `status` | Enum | Yes | See §3.8 |
| `mitigationPlan` | String | No | Free text |
| `identifiedDate` | Date | Yes | |
| `targetResolutionDate` | Date | No | |
| `resolvedDate` | Date | No | Null until closed/mitigated |
| `createdAt` / `updatedAt` | Date | auto | |

---

## 3. Dropdown / Enum Definitions

Use these exact value sets unless you have a specific reason to diverge — the Analytics tab's computed logic (RAG, severity, mitigation priority) depends on matching these labels.

**3.1 Status** (Phase / Milestone / Task — same list, since the Gantt and Exception Summary treat all three uniformly; Milestone typically only uses a subset)

| Value | Applies to |
|---|---|
| Not Started | Phase, Task |
| In Progress | Phase, Task |
| Blocked | Phase, Task |
| On Hold | Phase, Task |
| Completed | Phase, Task |
| Cancelled | Task |
| Pending | Milestone |
| Due | Milestone |
| Achieved | Milestone |
| Missed | Milestone |

**3.2 Phase Category** (optional, for grouping/reporting only)
`Planning` · `Design` · `Build` · `Testing` · `Deployment` · `Closure` · `Other`

**3.3 Priority** (Phase, Milestone, Task)
`Low` · `Medium` · `High` · `Critical`

**3.4 Milestone Type** (optional, for reporting)
`Internal` · `Client-Facing` · `Contractual` · `Regulatory`

**3.5 Task Type**
`Development` · `Testing` · `Design` · `Documentation` · `Deployment` · `Review` · `Support/Maintenance` · `Other`

**3.6 Risk Category**
`Technical` · `Schedule` · `Resource` · `Financial` · `Vendor/Third-Party` · `Compliance/Regulatory` · `Scope` · `Security` · `Operational`

**3.7 Risk Probability / Impact** (two separate fields, same value set)
`Low` · `Medium` · `High`

**3.8 Risk Status**
`Open` · `Monitoring` · `Mitigated` · `Escalated` · `Closed`

**3.9 RAG (Red / Amber / Green)** — used as an optional manual override field on Phase/Milestone/Task
`Red` · `Amber` · `Green`

**3.10 Owner / Assignee** — not a static dropdown. Populate from the Users/Resources collection (typeahead select), not a hardcoded list.

---

## 4. Computed / Derived Fields (server-side, not directly editable)

These must be computed consistently in one place (recommend a Java service layer, not duplicated in Angular) so every screen agrees on the same numbers:

| Field | Applies to | Logic |
|---|---|---|
| `isOverdue` | Phase, Milestone, Task | `endDate/dueDate < today` AND status not in resolved set (Completed/Achieved/Cancelled) |
| `isDueSoon` | Phase, Milestone, Task | Not overdue, not resolved, and `endDate/dueDate − today ≤ 7 days` |
| `varianceDays` | Phase, Task | `actualEndDate (or forecast) − plannedEndDate`, in days |
| `rag` | Phase, Milestone, Task | If `ragOverride` set, use it. Else derive: Red if overdue or blocked; Amber if due soon or variance > 0; Green otherwise |
| `riskSeverity` | Risk | Combine `probability` + `impact` (e.g. High+High = Critical, any High = High, else Medium/Low) |
| `mitigationPriority` | Risk | Derived from `riskSeverity` + `status` (Critical/open → Immediate; High/open → This Week; else Monitor) |
| `healthScore` | Phase/Project (aggregate) | Weighted score from % complete, count of blocked items, count of open high-priority risks, count of slipped tasks — used for the KPI "Project Health" card |
| `percentComplete` (Phase) | Phase | If not manually entered, roll up from average of child Milestone/Task completion |

---

## 5. Dashboard / Analytics Tab Requirements

The Analytics tab must behave as **one synchronized application**, not independent widgets. A prototype of this exact behavior exists (reference file attached/available) — replicate its interaction model precisely.

### 5.1 Tiles (already built — confirm behavior)
5 tiles: Analytics (default view), Phase, Milestone, Task, Risk. Each non-Analytics tile shows the **total count** of that entity type and, on click, opens a module view listing **every item of that type** (not just exceptions), sorted: Overdue first → Due Soon → On Track → Completed, each row tagged with a status badge. The row count on that screen must always equal the tile's number.

### 5.2 Summary / KPI Cards (5 cards, default = whole project)
1. **Overall Progress** — % complete rollup
2. **Project Health** — `healthScore`, with a hint label (Healthy / Attention Required / At Risk)
3. **Forecast Completion** — forecast finish date + variance vs. baseline (or, when a specific node is focused, show that node's own variance in days)
4. **Open High-Priority Risks** — count of unresolved, High-severity risks in scope
5. **Blocked Items** — count of Blocked-status tasks/phases in scope

A **"Viewing: X › Y › Z"** breadcrumb/context indicator sits next to this section's header. It reads "Portfolio-wide" when nothing is selected, and shows the live focused hierarchy path (clickable at every level) when a node is selected in the Gantt.

### 5.3 Exception Summary Table
One row per module (Phases / Milestones / Tasks / Risks) × columns: **Overdue, Due Soon, Blocked, High Risk, Critical Dependencies** (count of items with `varianceDays > 0` and unresolved). Overdue/Due Soon counts are clickable and open the matching module view. All counts must scope to the currently focused node (see §5.6); default to whole-project counts when nothing is focused.

### 5.4 Gantt Chart (primary "work zone")
- Hierarchical rows (Phase → Milestone → Task → Risk), collapsible per branch, sticky left label column
- Zoom levels: Week / Month / Quarter (changes pixels-per-day, not the data)
- Toggle switches: Show Baseline, Show Dependencies, Show Today marker
- Quick-filter chips: All / Overdue / Due Soon / Blocked / High Risk / Completed — filtering keeps ancestor rows visible for context even if they don't individually match, so the tree structure never breaks
- Free-text search + Type filter + Priority filter, combinable with the chips above
- Dependency lines drawn between parent/child (and any `dependsOn` links on Tasks); overdue dependencies render in a stronger red, blocked dependencies render dashed amber
- **Focus Mode (critical — see §5.6 for full behavior)**

### 5.5 Detail Panel (right-hand sticky panel next to the Gantt)
Collapsible sections, all populated from the currently selected node (default: project root):
- **General:** Name, Type, Owner, Status, Priority, RAG
- **Schedule:** Start, Baseline Start, Finish, Baseline Finish, Forecast Finish, Duration, Variance
- **Dependencies:** Parent Item, Child Items (names), Child Count, Upstream, Downstream, Dependency Count
- **Risks:** Linked Risks (count), Risk Severity (highest among open linked risks), Risk Owner
- **Resources:** Assigned Resources (list), Allocation (task count), Utilization (indicative %)
- Breadcrumb trail at the top of the panel — every non-current segment is clickable and re-focuses the dashboard to that ancestor

### 5.6 Focus Mode — Cross-Widget Synchronization (highest priority requirement)

This is the core behavior. Selecting **any** Project, Phase, Milestone, Task, or Risk row in the Gantt must, in a single action:

1. Highlight the selected row (bold border / glow) and its full ancestor + descendant chain in the Gantt; fade every unrelated row to ~15–20% opacity
2. Thicken and recolor dependency lines belonging to the focused execution path; fade unrelated connector lines
3. Update the Detail Panel (§5.5) to the selected node
4. Update the breadcrumb in both the Detail Panel and the Summary section context indicator (§5.2)
5. Recompute the 5 KPI cards (§5.2) scoped to the selected node's branch only
6. Recompute the Exception Summary (§5.3) scoped to the selected node's branch only
7. Recompute Resource & Workload — owner workload table, utilization bars, and upcoming deliverables (§5.7) scoped to the branch
8. Recompute Risk & Dependency Watch (§5.8) scoped to the branch
9. Auto-expand any collapsed ancestor rows needed to reveal the selection, and smooth-scroll the Gantt to bring it into view
10. Apply a brief animated transition (fade/pulse, ~300ms) across the KPI cards and affected sections so the context switch feels intentional, not jarring

Clicking empty space outside the Gantt/detail panel, or clicking the top-level breadcrumb segment, clears the focus and returns every section above to whole-project scope.

### 5.7 Resource & Workload (collapsed by default — expand on click)
- **Owner Workload table:** Owner, Assigned (task count), Completed, Blocked, Average % Complete — scoped per §5.6
- **Utilization bars:** one bar per owner, indicative load based on open task count (label this explicitly as "indicative, not logged hours")
- **Upcoming Deliverables table:** next unresolved milestones sorted by due date, with Due/Overdue/Soon/Pending badge — scoped per §5.6

### 5.8 Risk & Dependency Watch (collapsed by default — expand on click)
- **Risk Register table:** Risk, Owner, Probability, Impact, Status, **Severity** (computed, §4), **Mitigation Priority** (computed, §4), Mitigation Plan — scoped per §5.6
- **Dependency Impact table:** for every delayed, unresolved Task (`varianceDays > 0`): Delayed Item, Blocks Milestone (parent milestone name), **Affected Tasks** (sibling task count under same milestone), **Affected Milestones** (1 if parent is a milestone, else 0), Delay (days), **Severity**, **Mitigation Priority** — scoped per §5.6

---

## 6. Suggested API Endpoints

```
GET    /api/projects/{id}
GET    /api/projects/{id}/tree              → full nested Phase→Milestone→Task→Risk tree for the Gantt
GET    /api/phases?projectId=
POST   /api/phases
PUT    /api/phases/{id}
DELETE /api/phases/{id}

GET    /api/milestones?phaseId=
POST   /api/milestones
PUT    /api/milestones/{id}
DELETE /api/milestones/{id}

GET    /api/tasks?milestoneId=
POST   /api/tasks
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}

GET    /api/risks?linkedEntityType=&linkedEntityId=
POST   /api/risks
PUT    /api/risks/{id}
DELETE /api/risks/{id}

GET    /api/projects/{id}/summary?scopeNodeId=      → KPI cards + exception summary, scoped
GET    /api/projects/{id}/resources?scopeNodeId=    → workload/utilization/upcoming deliverables, scoped
GET    /api/projects/{id}/risk-watch?scopeNodeId=   → risk register + dependency impact, scoped
```

Decide during implementation whether scoping (`scopeNodeId`) happens server-side (recommended once real data volumes are large) or client-side against a fully-loaded tree (acceptable for a single project's worth of data).

---

## 7. Non-Functional Requirements

- **Validation:** enforce that `plannedEndDate ≥ plannedStartDate`, dates fall within parent Phase's window where applicable, and required-field rules from §2 at both the Angular form layer and the Java API layer (never trust client-side validation alone)
- **Performance:** the Gantt and dependency-line rendering must remain responsive with realistic data volumes (hundreds of tasks per project) — avoid full-tree re-render on every filter/focus change; use Angular `OnPush` change detection and debounce recalculation on scroll/resize
- **Accessibility:** keyboard navigation for Gantt row selection and section toggles, ARIA labels on interactive elements, and a non-hover-dependent way to view row details (the prototype's hover tooltip is not sufficient alone)
- **Consistency:** all computed fields (§4) must be calculated in exactly one place server-side and reused everywhere — do not let the frontend and backend calculate RAG/severity/overdue status independently, or the numbers will drift between screens

---

## 8. Exact UI Replication Requirement

**A working HTML/CSS/JS prototype exists and must be treated as the literal visual and interaction source of truth — not a rough guide.** Attach `project-dashboard-merged.html` alongside this spec when handing this to Devin. Everything in §5 describes *behavior*; this section locks down the *exact look* so nothing is left to interpretation, especially for the Gantt.

**General instruction to Devin:** Open the attached HTML file, inspect it (it runs standalone in a browser — no build step needed), and replicate its layout, spacing, colors, typography, and every interaction pixel-for-pixel in Angular. Where this section and the HTML file ever disagree, the HTML file wins — this section exists to make sure the exact values survive translation, not to override the file.

### 9.1 Design tokens (reuse exactly)

```css
--bg: #0f172a        (page background)
--panel: #1e293b      (card/section background)
--panel2: #0b1220     (nested/table background)
--border: #334155
--text: #e2e8f0
--muted: #94a3b8
--blue: #3b82f6       (Task color)
--indigo: #6366f1     (Project color / focus & selection color)
--emerald: #10b981    (Phase color)
--amber: #f59e0b      (Milestone color / due-soon)
--rose: #f43f5e       (Risk color / overdue / critical)
--gray: #64748b
Font: Arial, sans-serif throughout
```

### 9.2 Gantt chart — exact structure

- **Label column** (sticky left, always visible while horizontally scrolling): fixed width **260px**
- **Timeline axis**: dates render as a percentage of the full axis range (project start → project end), not as fixed day-widths, so bars reposition correctly at any zoom level
- **Zoom levels** control pixels-per-day of the rendered timeline width, not the data itself:
  - Week view: **34px per day**
  - Month view: **11px per day**
  - Quarter view: **4px per day**
  - Total rendered width = `total project days × px-per-day for current zoom`, with a minimum equal to the visible viewport width (so short projects don't render a tiny cramped chart)
- **Header** (sticky on vertical scroll): two stacked rows — a month band (bold month name, e.g. "Jun", each spanning its proportional width) above a week-tick row (thin vertical lines every 7 days with small "M/D" date labels)
- **Today marker**: a solid 2px vertical rose-colored line spanning the full chart height at today's date position, with a small "TODAY" pill label at its top; toggleable on/off
- **Row structure**, left to right: expand/collapse chevron (▾/▸, invisible/disabled for leaf rows) → small 8×8px colored square indicating type → row name (bold, 12.5px) with a smaller subtitle line underneath showing `TYPE • OWNER` in muted gray, 10px

### 9.3 Bar rendering — exact visual anatomy

Each row's timeline bar is not a single rectangle — it's a layered composite:
1. **Baseline indicator** (optional, toggleable): a thin 2px dashed/striped horizontal line at the *planned* start/end dates, positioned behind the actual bar, in muted gray at ~55% opacity — shows schedule slip at a glance when the solid bar extends past it
2. **Outline bar**: a 14px-tall rounded-rectangle (6px border-radius) positioned at the item's actual/forecast start–end dates. Border and fill color depend on node type: Project = indigo, Phase = emerald, Milestone = amber, Task = blue, Risk = rose
3. **Progress fill**: inside the outline, a solid fill from the left edge covering `percentComplete`% of the bar's width, same color as the outline
4. **Remainder texture**: the uncompleted portion of the bar (100% − percentComplete) is filled with a diagonal-hatch pattern (45° repeating stripes) at ~22% opacity in the same color — visually distinct from "just empty," signals "remaining work" rather than "no data"
5. **Percentage label**: small bold text immediately to the right of the bar's right edge showing the exact %, e.g. `64%`
6. **Health glow**: the outline bar gets a soft colored box-shadow ring around it — red ring if overdue, amber ring if due soon, faint green ring if healthy/on-track — so status is visible even before hovering

### 9.4 Dependency connector lines

- Drawn as SVG elbow-paths (horizontal → vertical → horizontal, not straight diagonal lines) connecting a parent bar's vertical midpoint to each child bar's vertical midpoint
- Default state: thin gray line (1.5px), arrowhead marker at the child end
- If the child is **overdue**: line renders in rose/red, thicker (2.5px)
- If the child is **Blocked** status: line renders dashed amber (2.5px, 5-3 dash pattern)
- When Focus Mode is active and this connector is part of the focused execution path: line thickens further (3px), gets a soft colored drop-shadow glow matching its state color, and unrelated connectors fade to ~10% opacity
- Recalculate all connector paths on scroll, zoom change, filter change, and window resize

### 9.5 Row interaction states

- **Hover**: subtle light overlay on the row background
- **Selected** (the exact node currently clicked): stronger background tint (indigo at ~16% opacity) plus the bar itself gets a 2px indigo outer ring and glow, overriding its normal health-glow color while selected
- **On focus path** (ancestor/descendant of the selection, but not the selection itself): light indigo background tint (~10% opacity), full normal opacity — reads as "part of the story" without being the star
- **Dimmed** (unrelated to current selection): opacity drops to ~18% and the row is desaturated (grayscale ~60%) — must use a smooth CSS transition (~200ms) when entering/leaving this state, never an instant snap
- **Hidden by filter** (fails the active search/type/priority/quick-filter, and has no visible descendant): row is fully removed from layout (`display:none`), not just dimmed — filtering and focus-dimming are visually distinct states and must not be confused

### 9.6 Tooltip (on row hover)

A small floating card follows the cursor, showing: item name (bold header) with `type • status` as a subtitle, then a two-column key/value grid: Owner, Start, Finish, Duration, Progress, Variance, Priority. Appears within ~14px offset from the cursor, disappears on mouseleave.

### 9.7 Toolbar controls (exact set, in this order)

Top toolbar row: chart title/date-range label (left) — Zoom control as a 3-way segmented button group (Week/Month/Quarter) — Expand All button — Collapse All button — Export dropdown (Excel/PDF/PNG/CSV).

Second row (filter chips): a row of pill-shaped quick-filter buttons — All Items / Overdue / Due Soon / Blocked / High Risk / Completed — single-select, active one filled indigo.

Third row: free-text search box, Type dropdown, Priority dropdown, three labeled toggle switches (Baseline / Dependencies / Today), and a Reset button that clears all filters and Focus Mode back to defaults in one click.

### 9.8 Side detail panel

Sits directly to the right of the Gantt (not below it), same height, non-scrolling independent of the Gantt's own scroll — this is a fixed side-by-side layout, not a modal or a tab.


## 9. Definition of Done

- [ ] Phase/Milestone/Task/Risk collections match the schema in §2, with the enums in §3 enforced at the API layer
- [ ] Computed fields (§4) are calculated server-side and returned as part of each entity's API response
- [ ] Analytics tab implements all of §5.1–§5.8
- [ ] Selecting any node anywhere in the Gantt triggers the full Focus Mode sync described in §5.6, with no section left un-updated
- [ ] Clearing focus returns every section to whole-project scope
- [ ] Non-functional requirements in §7 are met
- [ ] Gantt visual output matches §8 exactly — bar anatomy (outline/fill/hatched-remainder/health-glow), zoom px-per-day values, connector line states, and row interaction states (hover/selected/focus-path/dimmed/hidden) all render as specified, verified side-by-side against the attached prototype file
