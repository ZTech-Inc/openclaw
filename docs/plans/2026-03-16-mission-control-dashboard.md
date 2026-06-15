# Mission Control Dashboard — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a multi-agent mission control dashboard that lets operators manage tasks, calendars, projects, memories, departments, docs, teams, and a pixel-art office — all from a single dark-mode, Linear-inspired web UI.

**Architecture:** New views and components added to the existing `ui/` Lit web-component app, communicating with new and existing gateway RPC methods over WebSocket. State managed via Lit signals and context. Data persisted via gateway-backed SQLite storage. The dashboard is a new top-level navigation section in the existing control UI.

**Tech Stack:** Lit 3 (web components), Vite 8, TypeScript ESM, WebSocket gateway RPC, SQLite (via existing infra), CSS custom properties (dark-mode-first, Linear palette), Vitest.

**Design Language:** Dark mode only. Linear-inspired: neutral grays (`#1a1a2e`, `#16161a`, `#232326`), accent purple (`#7c5cfc`), crisp white text, minimal borders, generous whitespace, subtle hover states, system font stack (Inter where available). Cards with soft `1px` borders, no heavy shadows.

---

## File Structure

### New UI Files (views & components)

```
ui/src/ui/views/mission-control/
├── mission-control-shell.ts        # Top-level shell with sidebar nav
├── mission-control-styles.ts       # Shared Linear-dark CSS tokens
├── task-board/
│   ├── task-board-view.ts          # Kanban board root view
│   ├── task-board-column.ts        # Single kanban column
│   ├── task-board-card.ts          # Draggable task card
│   └── task-board-filters.ts       # Department/agent filter bar
├── calendar/
│   ├── calendar-view.ts            # Calendar root (month + week)
│   ├── calendar-grid.ts            # Day/week grid renderer
│   └── calendar-event-pill.ts      # Event pill component
├── projects/
│   ├── projects-view.ts            # Project list + detail split
│   ├── project-detail.ts           # Single project detail pane
│   └── project-link-chip.ts        # Linked task/memory/doc chip
├── memories/
│   ├── memories-view.ts            # Journal timeline view
│   ├── memory-day-group.ts         # Day grouping container
│   └── memory-entry-card.ts        # Single memory entry
├── departments/
│   ├── departments-view.ts         # Department list + detail
│   ├── department-detail.ts        # Single department pane
│   ├── docs-repository.ts          # Searchable docs list
│   ├── team-org-chart.ts           # Org chart tree
│   └── office-pixel-view.ts        # 2D pixel art office canvas
```

### New Gateway Methods

```
src/gateway/server-methods/
├── mission-control-tasks.ts        # CRUD + board queries for tasks
├── mission-control-projects.ts     # CRUD for projects + linking
├── mission-control-departments.ts  # CRUD for departments, docs, team
├── mission-control-calendar.ts     # Calendar event queries
└── mission-control-office.ts       # Office layout + agent positions
```

### New Gateway Protocol Schemas

```
src/gateway/protocol/
└── mission-control.ts              # TypeBox schemas for all MC methods
```

### New Data Layer

```
src/mission-control/
├── mc-store.ts                     # SQLite table init + queries
├── mc-types.ts                     # Shared domain types
├── mc-seed.ts                      # Optional seed/demo data
└── mc-store.test.ts                # Store unit tests
```

### Tests (colocated)

Each new `.ts` file above gets a colocated `.test.ts` where behavior is non-trivial.

---

## Chunk 1: Foundation — Data Layer, Protocol, Shell

### Task 1: Define domain types

**Files:**
- Create: `src/mission-control/mc-types.ts`

- [ ] **Step 1: Write type definitions**

```typescript
// src/mission-control/mc-types.ts

export type TaskStatus = "backlog" | "todo" | "in_progress" | "review" | "done";
export type TaskPriority = "urgent" | "high" | "medium" | "low" | "none";

export interface McTask {
  id: string;
  title: string;
  description: string;
  status: TaskStatus;
  priority: TaskPriority;
  assigneeAgentId: string | null;
  departmentId: string;
  projectId: string | null;
  dueDate: string | null; // ISO 8601
  createdAt: string;
  updatedAt: string;
  labels: string[];
}

export interface McProject {
  id: string;
  name: string;
  description: string;
  departmentId: string;
  status: "active" | "paused" | "completed" | "archived";
  linkedTaskIds: string[];
  linkedMemoryIds: string[];
  linkedDocIds: string[];
  createdAt: string;
  updatedAt: string;
}

export interface McMemoryEntry {
  id: string;
  agentId: string;
  departmentId: string | null;
  content: string;
  summary: string;
  day: string; // YYYY-MM-DD
  tags: string[];
  createdAt: string;
}

export interface McDepartment {
  id: string;
  name: string;
  mission: string;
  color: string; // hex
  createdAt: string;
}

export interface McTeamMember {
  id: string;
  agentId: string;
  departmentId: string;
  role: string;
  title: string;
}

export interface McDoc {
  id: string;
  title: string;
  departmentId: string;
  content: string;
  docType: "draft" | "plan" | "report" | "note";
  createdAt: string;
  updatedAt: string;
}

export interface McCalendarEvent {
  id: string;
  title: string;
  taskId: string | null;
  cronJobId: string | null;
  startTime: string;
  endTime: string | null;
  departmentId: string | null;
  recurring: boolean;
}

export interface McOfficeAgent {
  agentId: string;
  x: number;
  y: number;
  sprite: string;
  activity: "working" | "idle" | "meeting" | "away";
}
```

- [ ] **Step 2: Commit**

```bash
scripts/committer "feat(mission-control): add domain type definitions" src/mission-control/mc-types.ts
```

---

### Task 2: Create SQLite store

**Files:**
- Create: `src/mission-control/mc-store.ts`
- Create: `src/mission-control/mc-store.test.ts`

- [ ] **Step 1: Write failing tests for store initialization and basic CRUD**

```typescript
// src/mission-control/mc-store.test.ts
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import { McStore } from "./mc-store.js";
import Database from "better-sqlite3";

describe("McStore", () => {
  let db: Database.Database;
  let store: McStore;

  beforeEach(() => {
    db = new Database(":memory:");
    store = new McStore(db);
    store.init();
  });

  afterEach(() => {
    db.close();
  });

  describe("departments", () => {
    it("creates and lists departments", () => {
      store.createDepartment({ id: "eng", name: "Engineering", mission: "Ship code", color: "#7c5cfc" });
      const deps = store.listDepartments();
      expect(deps).toHaveLength(1);
      expect(deps[0].name).toBe("Engineering");
    });
  });

  describe("tasks", () => {
    it("creates a task and retrieves by department", () => {
      store.createDepartment({ id: "eng", name: "Engineering", mission: "Ship", color: "#7c5cfc" });
      store.createTask({
        title: "Build dashboard",
        status: "todo",
        priority: "high",
        departmentId: "eng",
        assigneeAgentId: null,
        projectId: null,
        description: "",
        dueDate: null,
        labels: [],
      });
      const tasks = store.listTasks({ departmentId: "eng" });
      expect(tasks).toHaveLength(1);
      expect(tasks[0].title).toBe("Build dashboard");
      expect(tasks[0].status).toBe("todo");
    });

    it("updates task status", () => {
      store.createDepartment({ id: "eng", name: "Engineering", mission: "Ship", color: "#7c5cfc" });
      const task = store.createTask({
        title: "Test task",
        status: "todo",
        priority: "medium",
        departmentId: "eng",
        assigneeAgentId: null,
        projectId: null,
        description: "",
        dueDate: null,
        labels: [],
      });
      store.updateTask(task.id, { status: "in_progress" });
      const updated = store.getTask(task.id);
      expect(updated?.status).toBe("in_progress");
    });
  });

  describe("projects", () => {
    it("creates and links tasks to a project", () => {
      store.createDepartment({ id: "eng", name: "Engineering", mission: "Ship", color: "#7c5cfc" });
      const project = store.createProject({
        name: "Dashboard v1",
        description: "Mission control",
        departmentId: "eng",
        status: "active",
      });
      expect(project.name).toBe("Dashboard v1");
    });
  });

  describe("memories", () => {
    it("creates and lists memories by day", () => {
      store.createMemory({
        agentId: "agent-1",
        departmentId: null,
        content: "Had a productive meeting about the roadmap.",
        summary: "Roadmap meeting",
        day: "2026-03-16",
        tags: ["planning"],
      });
      const memories = store.listMemories({ day: "2026-03-16" });
      expect(memories).toHaveLength(1);
      expect(memories[0].summary).toBe("Roadmap meeting");
    });
  });

  describe("docs", () => {
    it("creates and searches docs", () => {
      store.createDepartment({ id: "eng", name: "Engineering", mission: "Ship", color: "#7c5cfc" });
      store.createDoc({
        title: "API Design",
        departmentId: "eng",
        content: "REST vs GraphQL considerations",
        docType: "draft",
      });
      const docs = store.listDocs({ departmentId: "eng", query: "GraphQL" });
      expect(docs).toHaveLength(1);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm test -- src/mission-control/mc-store.test.ts -v`
Expected: FAIL — module not found

- [ ] **Step 3: Implement McStore**

```typescript
// src/mission-control/mc-store.ts
import type Database from "better-sqlite3";
import { randomUUID } from "node:crypto";
import type {
  McTask, McProject, McMemoryEntry, McDepartment,
  McDoc, McCalendarEvent, McTeamMember, McOfficeAgent,
  TaskStatus, TaskPriority,
} from "./mc-types.js";

export class McStore {
  constructor(private db: Database.Database) {}

  init(): void {
    this.db.exec(`
      CREATE TABLE IF NOT EXISTS mc_departments (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        mission TEXT NOT NULL DEFAULT '',
        color TEXT NOT NULL DEFAULT '#7c5cfc',
        created_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS mc_tasks (
        id TEXT PRIMARY KEY,
        title TEXT NOT NULL,
        description TEXT NOT NULL DEFAULT '',
        status TEXT NOT NULL DEFAULT 'backlog',
        priority TEXT NOT NULL DEFAULT 'none',
        assignee_agent_id TEXT,
        department_id TEXT NOT NULL REFERENCES mc_departments(id),
        project_id TEXT,
        due_date TEXT,
        labels TEXT NOT NULL DEFAULT '[]',
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        updated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS mc_projects (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        description TEXT NOT NULL DEFAULT '',
        department_id TEXT NOT NULL REFERENCES mc_departments(id),
        status TEXT NOT NULL DEFAULT 'active',
        linked_task_ids TEXT NOT NULL DEFAULT '[]',
        linked_memory_ids TEXT NOT NULL DEFAULT '[]',
        linked_doc_ids TEXT NOT NULL DEFAULT '[]',
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        updated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS mc_memories (
        id TEXT PRIMARY KEY,
        agent_id TEXT NOT NULL,
        department_id TEXT,
        content TEXT NOT NULL,
        summary TEXT NOT NULL DEFAULT '',
        day TEXT NOT NULL,
        tags TEXT NOT NULL DEFAULT '[]',
        created_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS mc_docs (
        id TEXT PRIMARY KEY,
        title TEXT NOT NULL,
        department_id TEXT NOT NULL REFERENCES mc_departments(id),
        content TEXT NOT NULL DEFAULT '',
        doc_type TEXT NOT NULL DEFAULT 'note',
        created_at TEXT NOT NULL DEFAULT (datetime('now')),
        updated_at TEXT NOT NULL DEFAULT (datetime('now'))
      );

      CREATE TABLE IF NOT EXISTS mc_calendar_events (
        id TEXT PRIMARY KEY,
        title TEXT NOT NULL,
        task_id TEXT,
        cron_job_id TEXT,
        start_time TEXT NOT NULL,
        end_time TEXT,
        department_id TEXT,
        recurring INTEGER NOT NULL DEFAULT 0
      );

      CREATE TABLE IF NOT EXISTS mc_team_members (
        id TEXT PRIMARY KEY,
        agent_id TEXT NOT NULL,
        department_id TEXT NOT NULL REFERENCES mc_departments(id),
        role TEXT NOT NULL DEFAULT '',
        title TEXT NOT NULL DEFAULT ''
      );

      CREATE TABLE IF NOT EXISTS mc_office_agents (
        agent_id TEXT PRIMARY KEY,
        x INTEGER NOT NULL DEFAULT 0,
        y INTEGER NOT NULL DEFAULT 0,
        sprite TEXT NOT NULL DEFAULT 'default',
        activity TEXT NOT NULL DEFAULT 'idle'
      );
    `);
  }

  // --- Departments ---
  createDepartment(input: { id: string; name: string; mission: string; color: string }): McDepartment {
    const now = new Date().toISOString();
    this.db.prepare(
      "INSERT INTO mc_departments (id, name, mission, color, created_at) VALUES (?, ?, ?, ?, ?)"
    ).run(input.id, input.name, input.mission, input.color, now);
    return { ...input, createdAt: now };
  }

  listDepartments(): McDepartment[] {
    return this.db.prepare("SELECT * FROM mc_departments ORDER BY name").all().map(rowToDepartment);
  }

  // --- Tasks ---
  createTask(input: {
    title: string; description: string; status: TaskStatus; priority: TaskPriority;
    departmentId: string; assigneeAgentId: string | null; projectId: string | null;
    dueDate: string | null; labels: string[];
  }): McTask {
    const id = randomUUID();
    const now = new Date().toISOString();
    this.db.prepare(
      `INSERT INTO mc_tasks (id, title, description, status, priority, assignee_agent_id,
       department_id, project_id, due_date, labels, created_at, updated_at)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`
    ).run(id, input.title, input.description, input.status, input.priority,
      input.assigneeAgentId, input.departmentId, input.projectId,
      input.dueDate, JSON.stringify(input.labels), now, now);
    return { id, ...input, createdAt: now, updatedAt: now };
  }

  getTask(id: string): McTask | null {
    const row = this.db.prepare("SELECT * FROM mc_tasks WHERE id = ?").get(id);
    return row ? rowToTask(row) : null;
  }

  listTasks(filter: { departmentId?: string; status?: TaskStatus; projectId?: string }): McTask[] {
    let sql = "SELECT * FROM mc_tasks WHERE 1=1";
    const params: unknown[] = [];
    if (filter.departmentId) { sql += " AND department_id = ?"; params.push(filter.departmentId); }
    if (filter.status) { sql += " AND status = ?"; params.push(filter.status); }
    if (filter.projectId) { sql += " AND project_id = ?"; params.push(filter.projectId); }
    sql += " ORDER BY updated_at DESC";
    return this.db.prepare(sql).all(...params).map(rowToTask);
  }

  updateTask(id: string, patch: Partial<Pick<McTask, "title" | "description" | "status" | "priority" | "assigneeAgentId" | "dueDate" | "labels">>): void {
    const sets: string[] = [];
    const params: unknown[] = [];
    if (patch.title !== undefined) { sets.push("title = ?"); params.push(patch.title); }
    if (patch.description !== undefined) { sets.push("description = ?"); params.push(patch.description); }
    if (patch.status !== undefined) { sets.push("status = ?"); params.push(patch.status); }
    if (patch.priority !== undefined) { sets.push("priority = ?"); params.push(patch.priority); }
    if (patch.assigneeAgentId !== undefined) { sets.push("assignee_agent_id = ?"); params.push(patch.assigneeAgentId); }
    if (patch.dueDate !== undefined) { sets.push("due_date = ?"); params.push(patch.dueDate); }
    if (patch.labels !== undefined) { sets.push("labels = ?"); params.push(JSON.stringify(patch.labels)); }
    if (sets.length === 0) return;
    sets.push("updated_at = ?"); params.push(new Date().toISOString());
    params.push(id);
    this.db.prepare(`UPDATE mc_tasks SET ${sets.join(", ")} WHERE id = ?`).run(...params);
  }

  deleteTask(id: string): void {
    this.db.prepare("DELETE FROM mc_tasks WHERE id = ?").run(id);
  }

  // --- Projects ---
  createProject(input: { name: string; description: string; departmentId: string; status: string }): McProject {
    const id = randomUUID();
    const now = new Date().toISOString();
    this.db.prepare(
      `INSERT INTO mc_projects (id, name, description, department_id, status, created_at, updated_at)
       VALUES (?, ?, ?, ?, ?, ?, ?)`
    ).run(id, input.name, input.description, input.departmentId, input.status, now, now);
    return {
      id, name: input.name, description: input.description, departmentId: input.departmentId,
      status: input.status as McProject["status"],
      linkedTaskIds: [], linkedMemoryIds: [], linkedDocIds: [],
      createdAt: now, updatedAt: now,
    };
  }

  listProjects(filter: { departmentId?: string }): McProject[] {
    let sql = "SELECT * FROM mc_projects WHERE 1=1";
    const params: unknown[] = [];
    if (filter.departmentId) { sql += " AND department_id = ?"; params.push(filter.departmentId); }
    sql += " ORDER BY updated_at DESC";
    return this.db.prepare(sql).all(...params).map(rowToProject);
  }

  // --- Memories ---
  createMemory(input: {
    agentId: string; departmentId: string | null; content: string;
    summary: string; day: string; tags: string[];
  }): McMemoryEntry {
    const id = randomUUID();
    const now = new Date().toISOString();
    this.db.prepare(
      `INSERT INTO mc_memories (id, agent_id, department_id, content, summary, day, tags, created_at)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?)`
    ).run(id, input.agentId, input.departmentId, input.content, input.summary, input.day, JSON.stringify(input.tags), now);
    return { id, ...input, createdAt: now };
  }

  listMemories(filter: { day?: string; agentId?: string; departmentId?: string }): McMemoryEntry[] {
    let sql = "SELECT * FROM mc_memories WHERE 1=1";
    const params: unknown[] = [];
    if (filter.day) { sql += " AND day = ?"; params.push(filter.day); }
    if (filter.agentId) { sql += " AND agent_id = ?"; params.push(filter.agentId); }
    if (filter.departmentId) { sql += " AND department_id = ?"; params.push(filter.departmentId); }
    sql += " ORDER BY created_at DESC";
    return this.db.prepare(sql).all(...params).map(rowToMemory);
  }

  // --- Docs ---
  createDoc(input: { title: string; departmentId: string; content: string; docType: string }): McDoc {
    const id = randomUUID();
    const now = new Date().toISOString();
    this.db.prepare(
      `INSERT INTO mc_docs (id, title, department_id, content, doc_type, created_at, updated_at)
       VALUES (?, ?, ?, ?, ?, ?, ?)`
    ).run(id, input.title, input.departmentId, input.content, input.docType, now, now);
    return {
      id, title: input.title, departmentId: input.departmentId, content: input.content,
      docType: input.docType as McDoc["docType"], createdAt: now, updatedAt: now,
    };
  }

  listDocs(filter: { departmentId?: string; query?: string }): McDoc[] {
    let sql = "SELECT * FROM mc_docs WHERE 1=1";
    const params: unknown[] = [];
    if (filter.departmentId) { sql += " AND department_id = ?"; params.push(filter.departmentId); }
    if (filter.query) { sql += " AND (title LIKE ? OR content LIKE ?)"; params.push(`%${filter.query}%`, `%${filter.query}%`); }
    sql += " ORDER BY updated_at DESC";
    return this.db.prepare(sql).all(...params).map(rowToDoc);
  }

  // --- Calendar ---
  createCalendarEvent(input: {
    title: string; taskId: string | null; cronJobId: string | null;
    startTime: string; endTime: string | null; departmentId: string | null; recurring: boolean;
  }): McCalendarEvent {
    const id = randomUUID();
    this.db.prepare(
      `INSERT INTO mc_calendar_events (id, title, task_id, cron_job_id, start_time, end_time, department_id, recurring)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?)`
    ).run(id, input.title, input.taskId, input.cronJobId, input.startTime, input.endTime, input.departmentId, input.recurring ? 1 : 0);
    return { id, ...input };
  }

  listCalendarEvents(filter: { startAfter?: string; startBefore?: string; departmentId?: string }): McCalendarEvent[] {
    let sql = "SELECT * FROM mc_calendar_events WHERE 1=1";
    const params: unknown[] = [];
    if (filter.startAfter) { sql += " AND start_time >= ?"; params.push(filter.startAfter); }
    if (filter.startBefore) { sql += " AND start_time <= ?"; params.push(filter.startBefore); }
    if (filter.departmentId) { sql += " AND department_id = ?"; params.push(filter.departmentId); }
    sql += " ORDER BY start_time ASC";
    return this.db.prepare(sql).all(...params).map(rowToCalendarEvent);
  }

  // --- Team ---
  addTeamMember(input: { agentId: string; departmentId: string; role: string; title: string }): McTeamMember {
    const id = randomUUID();
    this.db.prepare(
      "INSERT INTO mc_team_members (id, agent_id, department_id, role, title) VALUES (?, ?, ?, ?, ?)"
    ).run(id, input.agentId, input.departmentId, input.role, input.title);
    return { id, ...input };
  }

  listTeamMembers(filter: { departmentId?: string }): McTeamMember[] {
    let sql = "SELECT * FROM mc_team_members WHERE 1=1";
    const params: unknown[] = [];
    if (filter.departmentId) { sql += " AND department_id = ?"; params.push(filter.departmentId); }
    return this.db.prepare(sql).all(...params).map(rowToTeamMember);
  }

  // --- Office ---
  setOfficeAgent(input: McOfficeAgent): void {
    this.db.prepare(
      `INSERT OR REPLACE INTO mc_office_agents (agent_id, x, y, sprite, activity)
       VALUES (?, ?, ?, ?, ?)`
    ).run(input.agentId, input.x, input.y, input.sprite, input.activity);
  }

  listOfficeAgents(): McOfficeAgent[] {
    return this.db.prepare("SELECT * FROM mc_office_agents").all().map(rowToOfficeAgent);
  }
}

// --- Row mappers ---
function rowToDepartment(r: any): McDepartment {
  return { id: r.id, name: r.name, mission: r.mission, color: r.color, createdAt: r.created_at };
}
function rowToTask(r: any): McTask {
  return {
    id: r.id, title: r.title, description: r.description, status: r.status, priority: r.priority,
    assigneeAgentId: r.assignee_agent_id, departmentId: r.department_id, projectId: r.project_id,
    dueDate: r.due_date, labels: JSON.parse(r.labels), createdAt: r.created_at, updatedAt: r.updated_at,
  };
}
function rowToProject(r: any): McProject {
  return {
    id: r.id, name: r.name, description: r.description, departmentId: r.department_id,
    status: r.status, linkedTaskIds: JSON.parse(r.linked_task_ids),
    linkedMemoryIds: JSON.parse(r.linked_memory_ids), linkedDocIds: JSON.parse(r.linked_doc_ids),
    createdAt: r.created_at, updatedAt: r.updated_at,
  };
}
function rowToMemory(r: any): McMemoryEntry {
  return {
    id: r.id, agentId: r.agent_id, departmentId: r.department_id, content: r.content,
    summary: r.summary, day: r.day, tags: JSON.parse(r.tags), createdAt: r.created_at,
  };
}
function rowToDoc(r: any): McDoc {
  return {
    id: r.id, title: r.title, departmentId: r.department_id, content: r.content,
    docType: r.doc_type, createdAt: r.created_at, updatedAt: r.updated_at,
  };
}
function rowToCalendarEvent(r: any): McCalendarEvent {
  return {
    id: r.id, title: r.title, taskId: r.task_id, cronJobId: r.cron_job_id,
    startTime: r.start_time, endTime: r.end_time, departmentId: r.department_id,
    recurring: !!r.recurring,
  };
}
function rowToTeamMember(r: any): McTeamMember {
  return { id: r.id, agentId: r.agent_id, departmentId: r.department_id, role: r.role, title: r.title };
}
function rowToOfficeAgent(r: any): McOfficeAgent {
  return { agentId: r.agent_id, x: r.x, y: r.y, sprite: r.sprite, activity: r.activity };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm test -- src/mission-control/mc-store.test.ts -v`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
scripts/committer "feat(mission-control): add SQLite store with CRUD for all entities" src/mission-control/mc-store.ts src/mission-control/mc-store.test.ts
```

---

### Task 3: Gateway protocol schemas

**Files:**
- Create: `src/gateway/protocol/mission-control.ts`

- [ ] **Step 1: Define TypeBox schemas for all mission control RPC methods**

Define request/response schemas for:
- `mc.tasks.list`, `mc.tasks.create`, `mc.tasks.update`, `mc.tasks.delete`
- `mc.projects.list`, `mc.projects.create`
- `mc.memories.list`, `mc.memories.create`
- `mc.departments.list`, `mc.departments.create`
- `mc.docs.list`, `mc.docs.create`
- `mc.calendar.list`, `mc.calendar.create`
- `mc.team.list`, `mc.team.add`
- `mc.office.list`, `mc.office.update`

Follow the existing pattern in `src/gateway/protocol/index.ts` — use `Type.Object` with `Type.Optional` for optional fields. No `Type.Union`.

- [ ] **Step 2: Register schemas in the protocol index**

Add imports and register the new method schemas in `src/gateway/protocol/index.ts`.

- [ ] **Step 3: Commit**

```bash
scripts/committer "feat(mission-control): add gateway protocol schemas" src/gateway/protocol/mission-control.ts src/gateway/protocol/index.ts
```

---

### Task 4: Gateway RPC method handlers

**Files:**
- Create: `src/gateway/server-methods/mission-control-tasks.ts`
- Create: `src/gateway/server-methods/mission-control-projects.ts`
- Create: `src/gateway/server-methods/mission-control-departments.ts`
- Create: `src/gateway/server-methods/mission-control-calendar.ts`
- Create: `src/gateway/server-methods/mission-control-office.ts`

- [ ] **Step 1: Implement task methods handler**

Wire `mc.tasks.list`, `mc.tasks.create`, `mc.tasks.update`, `mc.tasks.delete` to `McStore` methods. Follow the handler registration pattern used in existing server-methods files (e.g., `src/gateway/server-methods/agents.ts`).

- [ ] **Step 2: Implement remaining handlers**

Repeat the pattern for projects, departments (including docs, team), calendar, and office.

- [ ] **Step 3: Register handlers in the gateway server**

Add handler registration calls in the gateway boot sequence (look at how existing method files are registered in `src/gateway/server.impl.ts` or the method registry).

- [ ] **Step 4: Commit**

```bash
scripts/committer "feat(mission-control): add gateway RPC handlers for all entities" \
  src/gateway/server-methods/mission-control-tasks.ts \
  src/gateway/server-methods/mission-control-projects.ts \
  src/gateway/server-methods/mission-control-departments.ts \
  src/gateway/server-methods/mission-control-calendar.ts \
  src/gateway/server-methods/mission-control-office.ts
```

---

### Task 5: Mission control shell and Linear-dark design tokens

**Files:**
- Create: `ui/src/ui/views/mission-control/mission-control-styles.ts`
- Create: `ui/src/ui/views/mission-control/mission-control-shell.ts`

- [ ] **Step 1: Create shared Linear-dark CSS tokens**

```typescript
// ui/src/ui/views/mission-control/mission-control-styles.ts
import { css } from "lit";

/** Linear-inspired dark-mode design tokens for Mission Control */
export const mcStyles = css`
  :host {
    /* Backgrounds */
    --mc-bg-primary: #16161a;
    --mc-bg-secondary: #1a1a2e;
    --mc-bg-elevated: #232326;
    --mc-bg-hover: #2a2a2e;
    --mc-bg-active: #333338;

    /* Text */
    --mc-text-primary: #efefef;
    --mc-text-secondary: #8b8b8f;
    --mc-text-tertiary: #5c5c60;

    /* Accent */
    --mc-accent: #7c5cfc;
    --mc-accent-hover: #6b4ce0;
    --mc-accent-subtle: rgba(124, 92, 252, 0.12);

    /* Borders */
    --mc-border: #2a2a2e;
    --mc-border-subtle: #222226;

    /* Status colors */
    --mc-status-backlog: #5c5c60;
    --mc-status-todo: #8b8b8f;
    --mc-status-in-progress: #f5a623;
    --mc-status-review: #7c5cfc;
    --mc-status-done: #4caf50;

    /* Priority colors */
    --mc-priority-urgent: #ef4444;
    --mc-priority-high: #f59e0b;
    --mc-priority-medium: #3b82f6;
    --mc-priority-low: #6b7280;

    /* Layout */
    --mc-sidebar-width: 240px;
    --mc-radius: 6px;
    --mc-radius-lg: 8px;

    /* Typography */
    --mc-font: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", system-ui, sans-serif;
    --mc-font-mono: "JetBrains Mono", "Fira Code", ui-monospace, monospace;

    font-family: var(--mc-font);
    color: var(--mc-text-primary);
    background: var(--mc-bg-primary);
  }

  /* Shared utility classes */
  .mc-card {
    background: var(--mc-bg-elevated);
    border: 1px solid var(--mc-border);
    border-radius: var(--mc-radius-lg);
    padding: 16px;
  }

  .mc-card:hover {
    border-color: var(--mc-border);
    background: var(--mc-bg-hover);
  }

  .mc-badge {
    display: inline-flex;
    align-items: center;
    padding: 2px 8px;
    border-radius: 10px;
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.02em;
  }

  .mc-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border-radius: var(--mc-radius);
    border: 1px solid var(--mc-border);
    background: var(--mc-bg-elevated);
    color: var(--mc-text-primary);
    font-size: 13px;
    cursor: pointer;
    transition: background 0.15s, border-color 0.15s;
  }

  .mc-btn:hover {
    background: var(--mc-bg-hover);
  }

  .mc-btn-primary {
    background: var(--mc-accent);
    border-color: var(--mc-accent);
    color: white;
  }

  .mc-btn-primary:hover {
    background: var(--mc-accent-hover);
  }

  .mc-input {
    background: var(--mc-bg-secondary);
    border: 1px solid var(--mc-border);
    border-radius: var(--mc-radius);
    padding: 8px 12px;
    color: var(--mc-text-primary);
    font-size: 13px;
    outline: none;
    transition: border-color 0.15s;
  }

  .mc-input:focus {
    border-color: var(--mc-accent);
  }

  .mc-section-title {
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--mc-text-tertiary);
    margin-bottom: 8px;
  }
`;
```

- [ ] **Step 2: Create the mission control shell with sidebar**

```typescript
// ui/src/ui/views/mission-control/mission-control-shell.ts
import { LitElement, html, css } from "lit";
import { customElement, state } from "lit/decorators.js";
import { mcStyles } from "./mission-control-styles.js";

type McPage = "tasks" | "calendar" | "projects" | "memories" | "departments";

@customElement("mc-shell")
export class McShell extends LitElement {
  static styles = [mcStyles, css`
    :host {
      display: flex;
      width: 100%;
      height: 100vh;
      overflow: hidden;
    }

    nav {
      width: var(--mc-sidebar-width);
      background: var(--mc-bg-secondary);
      border-right: 1px solid var(--mc-border);
      display: flex;
      flex-direction: column;
      padding: 16px 0;
      flex-shrink: 0;
    }

    .nav-header {
      padding: 0 16px 16px;
      font-size: 14px;
      font-weight: 600;
      color: var(--mc-text-primary);
      letter-spacing: -0.01em;
    }

    .nav-item {
      display: flex;
      align-items: center;
      gap: 10px;
      padding: 7px 16px;
      font-size: 13px;
      color: var(--mc-text-secondary);
      cursor: pointer;
      transition: color 0.1s, background 0.1s;
      border: none;
      background: none;
      width: 100%;
      text-align: left;
    }

    .nav-item:hover {
      color: var(--mc-text-primary);
      background: var(--mc-bg-hover);
    }

    .nav-item[data-active] {
      color: var(--mc-text-primary);
      background: var(--mc-accent-subtle);
    }

    .nav-icon {
      width: 16px;
      height: 16px;
      opacity: 0.7;
    }

    main {
      flex: 1;
      overflow-y: auto;
      background: var(--mc-bg-primary);
    }
  `];

  @state() activePage: McPage = "tasks";

  private navItems: Array<{ key: McPage; label: string; icon: string }> = [
    { key: "tasks", label: "Task Board", icon: "▣" },
    { key: "calendar", label: "Calendar", icon: "◫" },
    { key: "projects", label: "Projects", icon: "◎" },
    { key: "memories", label: "Memories", icon: "◉" },
    { key: "departments", label: "Departments", icon: "◈" },
  ];

  render() {
    return html`
      <nav>
        <div class="nav-header">Mission Control</div>
        ${this.navItems.map(item => html`
          <button
            class="nav-item"
            ?data-active=${this.activePage === item.key}
            @click=${() => { this.activePage = item.key; }}
          >
            <span class="nav-icon">${item.icon}</span>
            ${item.label}
          </button>
        `)}
      </nav>
      <main>
        ${this.renderPage()}
      </main>
    `;
  }

  private renderPage() {
    switch (this.activePage) {
      case "tasks": return html`<mc-task-board></mc-task-board>`;
      case "calendar": return html`<mc-calendar-view></mc-calendar-view>`;
      case "projects": return html`<mc-projects-view></mc-projects-view>`;
      case "memories": return html`<mc-memories-view></mc-memories-view>`;
      case "departments": return html`<mc-departments-view></mc-departments-view>`;
    }
  }
}
```

- [ ] **Step 3: Commit**

```bash
scripts/committer "feat(mission-control): add shell layout with Linear-dark design tokens" \
  ui/src/ui/views/mission-control/mission-control-styles.ts \
  ui/src/ui/views/mission-control/mission-control-shell.ts
```

---

## Chunk 2: Task Board (Kanban)

### Task 6: Task board view

**Files:**
- Create: `ui/src/ui/views/mission-control/task-board/task-board-view.ts`
- Create: `ui/src/ui/views/mission-control/task-board/task-board-column.ts`
- Create: `ui/src/ui/views/mission-control/task-board/task-board-card.ts`
- Create: `ui/src/ui/views/mission-control/task-board/task-board-filters.ts`

- [ ] **Step 1: Create the filter bar component**

`task-board-filters.ts` — a horizontal bar with department and agent dropdown filters. Uses `<select>` styled with `mc-input` class. Emits `filter-change` custom events with `{ departmentId, agentId }`.

- [ ] **Step 2: Create the task card component**

`task-board-card.ts` — displays task title, priority badge (colored dot), assignee avatar/name, due date, department badge. Card styled with `mc-card` class. Supports `draggable="true"` with `dragstart`/`dragend` events that set `dataTransfer` with the task ID.

- [ ] **Step 3: Create the kanban column component**

`task-board-column.ts` — a vertical column with a status header (Backlog, Todo, In Progress, Review, Done), task count badge, and a droppable zone. Listens for `dragover`/`drop` events and emits `task-moved` custom event with `{ taskId, newStatus }`.

- [ ] **Step 4: Create the board view**

`task-board-view.ts` — fetches tasks from `mc.tasks.list` via gateway WebSocket. Renders 5 `mc-task-board-column` components in a horizontal flex row. Handles `task-moved` by calling `mc.tasks.update`. Includes `mc-task-board-filters` above the columns. Supports creating new tasks via a modal form.

- [ ] **Step 5: Run the UI dev server and verify board renders**

Run: `cd ui && pnpm dev`
Expected: Navigate to Mission Control > Task Board, see 5 empty columns.

- [ ] **Step 6: Commit**

```bash
scripts/committer "feat(mission-control): add kanban task board with drag-and-drop" \
  ui/src/ui/views/mission-control/task-board/task-board-view.ts \
  ui/src/ui/views/mission-control/task-board/task-board-column.ts \
  ui/src/ui/views/mission-control/task-board/task-board-card.ts \
  ui/src/ui/views/mission-control/task-board/task-board-filters.ts
```

---

### Task 7: Calendar view

**Files:**
- Create: `ui/src/ui/views/mission-control/calendar/calendar-view.ts`
- Create: `ui/src/ui/views/mission-control/calendar/calendar-grid.ts`
- Create: `ui/src/ui/views/mission-control/calendar/calendar-event-pill.ts`

- [ ] **Step 1: Create the event pill component**

`calendar-event-pill.ts` — small colored pill showing event title, time, and an icon indicating if it's a cron job (repeat icon) or a task (checkmark). Color matches department color.

- [ ] **Step 2: Create the calendar grid**

`calendar-grid.ts` — renders a month grid (7 columns x ~5 rows) or a week view (7 columns x 24 hour rows). Each cell is a day/hour slot. Accepts an array of `McCalendarEvent` and renders pills in the correct slots. Clicking a day emits `day-selected`.

Properties:
- `mode: "month" | "week"`
- `currentDate: string` (ISO date)
- `events: McCalendarEvent[]`

- [ ] **Step 3: Create the calendar view**

`calendar-view.ts` — top-level view with month/week toggle, prev/next navigation arrows, "Today" button. Fetches events from `mc.calendar.list` with the visible date range. Also pulls cron jobs from the existing `cron` gateway method and maps them to calendar events for display.

- [ ] **Step 4: Commit**

```bash
scripts/committer "feat(mission-control): add calendar view with month/week modes" \
  ui/src/ui/views/mission-control/calendar/calendar-view.ts \
  ui/src/ui/views/mission-control/calendar/calendar-grid.ts \
  ui/src/ui/views/mission-control/calendar/calendar-event-pill.ts
```

---

## Chunk 3: Projects, Memories, Departments

### Task 8: Projects view

**Files:**
- Create: `ui/src/ui/views/mission-control/projects/projects-view.ts`
- Create: `ui/src/ui/views/mission-control/projects/project-detail.ts`
- Create: `ui/src/ui/views/mission-control/projects/project-link-chip.ts`

- [ ] **Step 1: Create the link chip component**

`project-link-chip.ts` — a small inline chip that represents a linked item (task, memory, or doc). Shows an icon + title. Clicking emits `link-navigate` with the item type and ID.

- [ ] **Step 2: Create the project detail pane**

`project-detail.ts` — shows project name, description, status badge, department, and three sections: "Linked Tasks", "Linked Memories", "Linked Docs" — each rendered as a list of `mc-project-link-chip` components. Includes an "Add Link" button that opens a search dropdown.

- [ ] **Step 3: Create the projects list view**

`projects-view.ts` — master-detail layout. Left panel: list of projects grouped by department, filterable. Right panel: `mc-project-detail` for the selected project. Includes a "New Project" form (title, description, department picker).

- [ ] **Step 4: Commit**

```bash
scripts/committer "feat(mission-control): add projects view with linking" \
  ui/src/ui/views/mission-control/projects/projects-view.ts \
  ui/src/ui/views/mission-control/projects/project-detail.ts \
  ui/src/ui/views/mission-control/projects/project-link-chip.ts
```

---

### Task 9: Memories view (digital journal)

**Files:**
- Create: `ui/src/ui/views/mission-control/memories/memories-view.ts`
- Create: `ui/src/ui/views/mission-control/memories/memory-day-group.ts`
- Create: `ui/src/ui/views/mission-control/memories/memory-entry-card.ts`

- [ ] **Step 1: Create the memory entry card**

`memory-entry-card.ts` — displays a single memory entry: agent name, timestamp, summary (bold), content (body), tags as small badges. Styled as a minimal card with a left accent border colored by department.

- [ ] **Step 2: Create the day group component**

`memory-day-group.ts` — groups memory entries under a date heading ("Today", "Yesterday", or "March 14, 2026"). Renders a vertical timeline with `mc-memory-entry-card` components.

- [ ] **Step 3: Create the memories view**

`memories-view.ts` — infinite-scroll timeline. Fetches memories from `mc.memories.list` grouped by day. Includes a top filter bar for agent and department. Includes a date picker for jumping to a specific day. Loads more entries on scroll.

- [ ] **Step 4: Commit**

```bash
scripts/committer "feat(mission-control): add memories journal timeline view" \
  ui/src/ui/views/mission-control/memories/memories-view.ts \
  ui/src/ui/views/mission-control/memories/memory-day-group.ts \
  ui/src/ui/views/mission-control/memories/memory-entry-card.ts
```

---

### Task 10: Departments view (with Docs, Team, Office tabs)

**Files:**
- Create: `ui/src/ui/views/mission-control/departments/departments-view.ts`
- Create: `ui/src/ui/views/mission-control/departments/department-detail.ts`
- Create: `ui/src/ui/views/mission-control/departments/docs-repository.ts`
- Create: `ui/src/ui/views/mission-control/departments/team-org-chart.ts`
- Create: `ui/src/ui/views/mission-control/departments/office-pixel-view.ts`

- [ ] **Step 1: Create the docs repository component**

`docs-repository.ts` — searchable list of department docs. Top search input (`mc-input`), filterable by doc type (draft, plan, report, note). Each doc row shows title, type badge, last updated date. Clicking a doc opens an inline markdown viewer (use existing `marked` dependency for rendering).

- [ ] **Step 2: Create the team org chart**

`team-org-chart.ts` — displays department mission statement at top in a highlighted card. Below, shows a tree/grid of team members. Each member shows agent name, role, title. Uses CSS grid for layout. Includes an "Add Member" button.

- [ ] **Step 3: Create the pixel art office**

`office-pixel-view.ts` — a `<canvas>` element that renders a simple 2D pixel art office scene. The office is a grid-based room with:
- Desks (colored rectangles)
- Agent sprites (8x8 or 16x16 pixel art characters) positioned at their desk
- Activity indicators (working: typing animation, idle: standing, meeting: grouped, away: grayed)
- Department-colored name labels below each agent

Uses `requestAnimationFrame` for gentle idle animations. Data comes from `mc.office.list`. Canvas size ~640x400. Pixel-perfect rendering with `image-rendering: pixelated`.

The pixel art assets are drawn programmatically (no external sprite sheets):
- Simple humanoid sprites: 16x16 px
- Desk: 24x16 rectangle with monitor
- Floor: tiled grid pattern
- Walls: darker borders

- [ ] **Step 4: Create the department detail pane**

`department-detail.ts` — tabbed view with 3 tabs: "Docs", "Team", "Office". Renders the corresponding sub-component based on active tab. Shows department name, mission, and color indicator at top.

- [ ] **Step 5: Create the departments list view**

`departments-view.ts` — master-detail layout. Left panel: department list with colored dots and names. Right panel: `mc-department-detail`. Includes "New Department" form.

- [ ] **Step 6: Commit**

```bash
scripts/committer "feat(mission-control): add departments view with docs, team, and pixel office" \
  ui/src/ui/views/mission-control/departments/departments-view.ts \
  ui/src/ui/views/mission-control/departments/department-detail.ts \
  ui/src/ui/views/mission-control/departments/docs-repository.ts \
  ui/src/ui/views/mission-control/departments/team-org-chart.ts \
  ui/src/ui/views/mission-control/departments/office-pixel-view.ts
```

---

## Chunk 4: Integration, Navigation, and Testing

### Task 11: Wire Mission Control into the existing UI app

**Files:**
- Modify: `ui/src/ui/app.ts` (add MC route/nav entry)
- Modify: `ui/src/main.ts` (import MC components)

- [ ] **Step 1: Add Mission Control as a top-level nav item**

In `app.ts`, add a "Mission Control" entry to the app's navigation/sidebar. Follow the existing pattern for how other views (Chat, Config, Channels, etc.) are registered.

- [ ] **Step 2: Add component imports**

In `main.ts` (or wherever components are bulk-imported), add imports for all `mission-control/**` components so they're registered as custom elements.

- [ ] **Step 3: Verify navigation works**

Run: `cd ui && pnpm dev`
Expected: "Mission Control" appears in sidebar. Clicking it renders the MC shell with sub-navigation.

- [ ] **Step 4: Commit**

```bash
scripts/committer "feat(mission-control): wire into main app navigation" \
  ui/src/ui/app.ts ui/src/main.ts
```

---

### Task 12: Component tests

**Files:**
- Create: `ui/src/ui/views/mission-control/mission-control-shell.test.ts`
- Create: `ui/src/ui/views/mission-control/task-board/task-board-card.test.ts`
- Create: `ui/src/ui/views/mission-control/calendar/calendar-grid.test.ts`
- Create: `ui/src/ui/views/mission-control/departments/office-pixel-view.test.ts`

- [ ] **Step 1: Test the shell component**

Test that the shell renders all 5 nav items, that clicking a nav item switches the active page, and that the correct child component tag is rendered.

- [ ] **Step 2: Test the task card**

Test that the card renders title, priority, assignee. Test that `draggable` attribute is set. Test that `dragstart` event sets the task ID in `dataTransfer`.

- [ ] **Step 3: Test the calendar grid**

Test that a month grid renders 7 columns. Test that events appear in the correct day cell. Test that clicking a day emits `day-selected`.

- [ ] **Step 4: Test the pixel office**

Test that the canvas is created with the correct dimensions. Test that `listOfficeAgents` data is rendered (mock the gateway call).

- [ ] **Step 5: Run all tests**

Run: `pnpm test -- ui/src/ui/views/mission-control/ -v`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
scripts/committer "test(mission-control): add component tests" \
  ui/src/ui/views/mission-control/mission-control-shell.test.ts \
  ui/src/ui/views/mission-control/task-board/task-board-card.test.ts \
  ui/src/ui/views/mission-control/calendar/calendar-grid.test.ts \
  ui/src/ui/views/mission-control/departments/office-pixel-view.test.ts
```

---

### Task 13: Build verification and type check

- [ ] **Step 1: Run type check**

Run: `pnpm tsgo`
Expected: No type errors in mission-control files.

- [ ] **Step 2: Run lint and format**

Run: `pnpm check`
Expected: No lint errors. Fix any formatting issues with `pnpm format:fix`.

- [ ] **Step 3: Run full build**

Run: `pnpm build`
Expected: Build succeeds. No `[INEFFECTIVE_DYNAMIC_IMPORT]` warnings.

- [ ] **Step 4: Commit any fixes**

```bash
scripts/committer "chore(mission-control): fix lint and type issues" <files...>
```

---

## Summary

| Chunk | Tasks | What it delivers |
|-------|-------|-----------------|
| 1: Foundation | Tasks 1-5 | Types, SQLite store, protocol schemas, gateway handlers, shell + design tokens |
| 2: Task Board & Calendar | Tasks 6-7 | Kanban drag-and-drop board, month/week calendar with cron integration |
| 3: Projects, Memories, Departments | Tasks 8-10 | Project linking, journal timeline, docs repo, org chart, pixel art office |
| 4: Integration & Testing | Tasks 11-13 | App navigation wiring, component tests, build verification |

**Total new files:** ~25 source + ~5 test files
**Estimated implementation chunks:** 4, each independently testable
