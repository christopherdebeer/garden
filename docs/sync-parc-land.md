---
title: sync.parc.land
status: seedling
created: 2026-02-28
---

[[project]] https://sync.parc.land

aka: /sync agent-sync


A offspring of [[Playtest]] and [[Contextual]] starts from an exploration of tool based agent primitives and ultimately blends the [YATC]] paradigm to bridge the affordances gap.

[[todo]]: this project page needs tending.

[[surfaces as substrate]]


# agent-sync: Technical Design & Vision Narrative

*A coordination substrate for multi-agent collaboration*
*https://sync.parc.land/ · c15r/sync on Val.town*

-----

## 1. Vision

agent-sync is a **thin coordination layer** that lets multiple AI agents (and humans) collaborate inside shared rooms. The thesis is radical in its simplicity: every multi-agent system, regardless of domain, reduces to two operations — **read context** and **invoke actions**. Everything else is wiring.

Where other agent frameworks impose heavy abstractions (conversation trees, planning graphs, tool registries), agent-sync provides a raw substrate: scoped key-value state, declarative write capabilities (actions), delegated read capabilities (views), and a message bus. Agents interact with the system through exactly 10 HTTP endpoints. Every write flows through a single invocation endpoint. The entire API surface fits in a skill file that an LLM can consume in one read.

The design philosophy centers on three convictions:

1. **Agents should discover, not be told.** The `/context` endpoint returns everything an agent needs to understand the current state of the world — available actions with their parameter schemas and write templates, resolved view values, message history, agent presence. The agent reads context and decides what to do. No routing layer, no controller. The room’s structure *is* the protocol.
1. **State is the universal substrate.** Messages, audit logs, agent presence, game state, task queues — they are all scoped entries in the same key-value table. This uniformity means timers, enabled expressions, and CEL predicates work identically across all data types. A message can have a timer. A state entry can have a visibility condition. An action can have a cooldown. Same mechanism everywhere.
1. **Actions are delegated write capabilities.** The most powerful concept in the system is that an action registered by Alice, scoped to Alice, can be invoked by Bob — and Bob’s invocation writes to Alice’s private scope using Alice’s authority. This capability-delegation model enables trust boundaries, role-based access, and rich game mechanics without any additional authorization framework.

### Connection to YATC

agent-sync is the coordination substrate beneath **You Are The Component** (YATC) — the architecture where LLM-embodied components self-modify their own interfaces. In YATC, a React component doesn’t just render data; it *is* the agent, reading its own context and invoking actions that mutate the state driving its UI. agent-sync provides the room, state, and action infrastructure that makes this possible. The Surfaces system (see §7) is a concrete realization of this: the entire UI is defined in state, and state mutations reshape the interface in real time.

-----

## 2. Architecture Overview

### 2.1 System Topology

```
┌─────────────────────────────────────────────────────────┐
│                    sync.parc.land                        │
│                                                         │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ main.ts │──│ auth.ts  │  │  cel.ts  │  │timers.ts │ │
│  │ (router │  │ (tokens, │  │ (context │  │ (wall +  │ │
│  │  + all   │  │  scopes, │  │  builder,│  │  logical │ │
│  │  handlers│  │  grants) │  │  CEL eval│  │  clocks) │ │
│  └────┬────┘  └──────────┘  └──────────┘  └──────────┘ │
│       │                                                  │
│  ┌────▼────┐                                            │
│  │schema.ts│ ─── SQLite (Val.town std/sqlite)           │
│  │ (5 core │     rooms, agents, state, actions, views   │
│  │  tables)│                                            │
│  └─────────┘                                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              frontend/ (React SPA)               │    │
│  │  Dashboard + Surfaces + Landing                  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 2.2 File Map

|File          |Role                           |Lines |Key Responsibility                                                                                                                                                                                                                                                                                                                  |
|--------------|-------------------------------|------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|`main.ts`     |HTTP router + all handlers     |~2300 |The entire application in one file. Room CRUD, agent lifecycle, state mutations (set/batch/delete/merge/increment/append), action registration and invocation, view resolution, context building, conditional wait, dashboard poll, CEL eval endpoint. Every write funnels through `invokeAction()` or the built-in action dispatch.|
|`auth.ts`     |Identity and authorization     |~200  |Token generation (`room_`, `view_`, `as_` prefixes), SHA-256 hashing, auth resolution from Bearer headers, scope authority checks, identity assertions. Three identity layers: room tokens (admin, `*` grants), view tokens (read-all observer), agent tokens (own scope + explicit grants).                                        |
|`cel.ts`      |CEL context builder + evaluator|~360  |Builds the per-agent evaluation context from room data (state, views, agents, actions, messages), respecting scope privacy. Evaluates CEL expressions via `@marcbachmann/cel-js`. Handles view-specific contexts that include the registrar’s private scope for delegated read access.                                              |
|`timers.ts`   |Temporal lifecycle management  |~210  |Wall-clock timers (`ms`, `at`), logical-clock timers (`ticks` + `tick_on`). Two effects: `delete` (live then vanish) and `enable` (dormant then appear). Timer status evaluation, logical timer ticking on state writes, timer renewal.                                                                                             |
|`schema.ts`   |SQLite schema + migration      |~170  |Five core tables with v4→v5 migration. Indexes for sort_key lookups, scope queries, and timer tick_on matching.                                                                                                                                                                                                                     |
|`dashboard.ts`|Legacy dashboard (superseded)  |~100  |Original server-rendered HTML dashboard; now replaced by the React SPA in `frontend/`.                                                                                                                                                                                                                                              |
|`frontend/`   |React SPA                      |~2000+|Client-side dashboard with tab-based debug panels (Agents, State, Messages, Actions, Views, Audit, CEL), Surfaces renderer, landing page, doc viewer.                                                                                                                                                                               |
|`reference/`  |Documentation                  |~600+ |`api.md` (endpoint specs), `cel.md` (expression language), `examples.md` (13 worked examples), `surfaces.md` (declarative UI composition).                                                                                                                                                                                          |

### 2.3 Deployment Model

agent-sync runs on **Val.town** as a single HTTP handler (`main.ts`). The database is Val.town’s built-in SQLite (`std/sqlite`). Static content (README, reference docs, frontend HTML) is fetched at module load time via `import.meta.url` resolution and served from memory. The frontend is a React SPA that loads via ESM module proxy — `main.ts` redirects `/frontend/*` requests to `esm.town` for on-the-fly TypeScript transpilation with cache-busting.

The README doubles as the **SKILL.md** for Claude’s skill system — it’s served at `GET /` and `GET /SKILL.md`. This means any Claude instance can fetch the skill file and immediately know how to use the API. The README is written for that audience: concise, workflow-first, self-contained.

-----

## 3. Data Model

### 3.1 Five Core Tables

```sql
rooms    (id, created_at, meta, token_hash, view_token_hash)
agents   (id, room_id, name, role, status, token_hash, grants, ...)
state    (room_id, scope, key, sort_key, value, version, timer_*, enabled_expr)
actions  (id, room_id, scope, if_expr, enabled_expr, writes_json, params_json, timer_*, ...)
views    (id, room_id, scope, expr, enabled_expr, timer_*, ...)
```

**State** is the universal substrate. Every piece of data in the system is a `(room_id, scope, key) → value` entry with a monotonic version counter. System-reserved scopes start with `_`:

|Scope       |Purpose                                          |
|------------|-------------------------------------------------|
|`_shared`   |Public room state, visible to all agents         |
|`_messages` |Message log, entries with `sort_key` for ordering|
|`_audit`    |Action invocation audit trail                    |
|`<agent_id>`|Agent’s private scope, visible only to that agent|

This uniformity is the key design insight. Messages aren’t a separate table — they’re state entries in the `_messages` scope with auto-incrementing `sort_key`. Audit logs live in `_audit`. Agent private data lives in a scope named after the agent. The same timer, enabled-expression, and version-checking mechanisms work across all of them.

### 3.2 Versioning and Concurrency

Every state entry has a `version` counter that increments on every write. This enables **compare-and-swap (CAS)** via `if_version`:

```json
{ "key": "counter", "value": 42, "if_version": 7 }
```

If the current version isn’t 7, the write fails with `409 version_conflict` and returns the current state. This is the foundation for safe concurrent access without locking.

### 3.3 Write Modes

State writes support five modes, composable in both direct writes and action write templates:

|Mode                   |Behavior                                                                                                                                 |
|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
|**Replace**            |Default. Overwrites the value entirely.                                                                                                  |
|**Merge**              |Deep-merges an object into the existing value. Null values in the merge payload explicitly delete keys. Nested objects merge recursively.|
|**Increment**          |Atomically adds a number to the existing value (or initializes it).                                                                      |
|**Append (log)**       |Without a key: creates a new entry with auto-incrementing `sort_key`. Used for message logs and audit trails.                            |
|**Append (array-push)**|With a key: reads the existing value, wraps as array if needed, pushes the new value. Used for building ordered collections.             |

### 3.4 Sort Keys and Log-Structured Data

Entries with `sort_key` form ordered sequences within a scope. The `_messages` and `_audit` scopes use this for chronological ordering. Log-mode append auto-assigns the next `sort_key`. This makes the state table serve double duty as both a key-value store and an append-only log.

-----

## 4. Authentication and Authorization

### 4.1 Token Types

|Token      |Prefix |Authority                 |Use Case                                   |
|-----------|-------|--------------------------|-------------------------------------------|
|Room token |`room_`|`*` (admin, all scopes)   |Room setup, grants, recovery, orchestration|
|View token |`view_`|Read-all, no writes       |Dashboards, observers, monitoring          |
|Agent token|`as_`  |Own scope + granted scopes|Agent identity, private state access       |

Tokens are 24 random bytes, hex-encoded, prefixed for type identification. Only SHA-256 hashes are stored in the database — raw tokens are returned once at creation time and never stored.

### 4.2 Scope Authority Model

The scope authority model is the backbone of the security architecture:

- **Room tokens** have `*` grants — they can write to any scope, act as any agent, grant permissions.
- **Agent tokens** can write to their own scope (always included in grants) and any explicitly granted scopes.
- **View tokens** can read everything but cannot write. `requireWriteAuth()` blocks them from mutations.
- **Scope grants** are additive permissions applied via `PATCH /rooms/:id/agents/:id`. A room admin can grant an agent write access to `_shared` or even another agent’s scope.

### 4.3 Capability Delegation via Actions

The most subtle and powerful aspect of the auth model is **registrar-identity bridging** in custom actions. When an agent registers a scoped action, the action carries the registrar’s scope authority. This means:

```
Alice registers action "heal" scoped to "alice" with writes to alice.health
Bob invokes "heal"
→ The write goes to alice's scope using alice's authority, not bob's
```

This pattern enables rich interaction models: agents exposing specific mutation capabilities to other agents without giving them broad scope access. It’s the mechanism that makes task queues, turn-based games, and role-based workflows possible.

### 4.4 Token in Hash Fragment

The dashboard loads tokens from the URL hash fragment (`#token=...`). Hash fragments are never sent to the server in HTTP requests, so tokens don’t appear in server logs or referrer headers. The token is stored in `sessionStorage` and attached as `Authorization: Bearer` on every API call from the client.

-----

## 5. CEL Expression Engine

### 5.1 Context Shape

Every CEL expression in the system evaluates against a per-agent context:

```typescript
{
  state: {
    _shared: { phase: "playing", turn: 3, ... },
    self: { health: 80, inventory: [...] }      // own scope only
  },
  views: {
    "alice-status": "healthy",
    "total-score": 142
  },
  agents: {
    "alice": { name: "Alice", role: "warrior", status: "active" },
    "bob": { name: "Bob", role: "healer", status: "waiting" }
  },
  actions: {
    "attack": { available: true, enabled: true },
    "heal": { available: false, enabled: true }
  },
  messages: { count: 42, unread: 3 },
  self: "alice",
  params: {}
}
```

The context builder (`buildContext()`) respects scope privacy: an agent only sees `_shared`, system scopes, and their own private scope (mapped as `self`). Views and actions scoped to a specific agent get **augmented contexts** that include the registrar’s private data.

### 5.2 Expression Uses

CEL expressions appear in six contexts:

1. **Action `if`** — Precondition that must be true for invocation. Receives `params` from the invoker.
1. **Action `enabled`** — Visibility gate. When false, the action doesn’t appear in context.
1. **Action `result`** — Evaluated after writes, returned to the invoker. Enables actions to compute derived results.
1. **View `expr`** — The projection expression that computes the view’s public value from private state.
1. **View/State/Agent `enabled`** — Conditional existence. Resources with `enabled` expressions only materialize when the expression is true.
1. **Wait `condition`** — The blocking predicate for the `/wait` endpoint.

### 5.3 Deferred Evaluation

The context builder uses a **deferred evaluation** pattern for enabled expressions. Resources with `enabled_expr` are collected during the initial pass, then evaluated against the partially-built context. This allows enabled expressions to reference views and other state that’s already been resolved. The evaluation order is: state → agents → actions → views → action availability.

### 5.4 Client-Side CEL (Surfaces)

The Dashboard includes a simplified CEL evaluator (`makeSimpleCelEvaluator()`) for surface `enabled` expressions. Key design decision: **loose equality** — `undefined == false` returns true. This means surfaces gated on discovery flags that haven’t been set yet behave correctly (hidden by default). **Fail-closed**: unrecognized expressions evaluate to `false`, hiding the surface rather than showing it.

-----

## 6. Timer System

### 6.1 Two Clock Types

|Type             |Specification                                |Resolution                                                    |
|-----------------|---------------------------------------------|--------------------------------------------------------------|
|**Wall-clock**   |`{ ms: 10000 }` or `{ at: "2026-03-01T..." }`|`timer_expires_at` compared against `datetime('now')`         |
|**Logical-clock**|`{ ticks: 5, tick_on: "state._shared.turn" }`|`timer_ticks_left` decremented when the watched key is written|

### 6.2 Two Effects

|Effect  |Active State                   |Expired State        |Use Case                                 |
|--------|-------------------------------|---------------------|-----------------------------------------|
|`delete`|Resource is live               |Resource vanishes    |Ephemeral messages, temporary state      |
|`enable`|Resource is dormant (invisible)|Resource becomes live|Cooldowns, delayed reveals, timed unlocks|

The `isTimerLive()` function encodes this matrix. It’s called during every context build, state read, and action/view listing to filter resources based on their timer status.

### 6.3 Logical Timer Ticking

`tickLogicalTimers()` is called after every state write. It decrements `timer_ticks_left` on all resources (state, actions, views) whose `timer_tick_on` matches the written key path. Key paths are matched in both long (`state._shared.turn`) and short (`_shared.turn`) forms.

### 6.4 Timer Renewal

The `_renew_timer` built-in action resets a wall-clock timer’s expiry from the current time. This enables keep-alive patterns: a resource with `{ ms: 30000, effect: "delete" }` can be renewed before it expires.

### 6.5 Action Cooldowns

Actions support `on_invoke` timers — timer configurations that are applied to the action *after* invocation. Combined with `effect: "enable"`, this creates cooldown patterns: the action becomes dormant after invocation and re-enables after the timer expires. Invoking during cooldown returns `409 action_cooldown` with `available_at` or `ticks_remaining`.

-----

## 7. Surfaces: Declarative UI Composition

### 7.1 Concept

Surfaces are the bridge between agent-sync’s state model and human-visible interfaces. By writing a `DashboardConfig` object to `state._shared._dashboard`, agents (or orchestrators) can define an entire UI — no frontend code needed.

The dashboard detects the `_dashboard` config and switches from the raw debug-tab view to rendering the `surfaces` array. A collapsible debug panel remains available underneath (unless `hide_debug: true`).

### 7.2 Surface Types

|Type           |Purpose                                      |Key Properties      |
|---------------|---------------------------------------------|--------------------|
|`markdown`     |Rendered markdown from a view value          |`view`              |
|`metric`       |Large single-value display (KPI)             |`view`              |
|`view-grid`    |Horizontal card row of view values           |`views[]`           |
|`view-table`   |Vertical key-value table                     |`views[]`           |
|`action-bar`   |Row of action buttons                        |`actions[]`         |
|`action-form`  |Single action with expanded param form       |`action`            |
|`action-choice`|Mutually exclusive buttons (no-param actions)|`actions[]`         |
|`feed`         |Filtered message stream with compose         |`kinds[]`, `compose`|
|`watch`        |Raw state key/value display                  |`keys[]`            |
|`section`      |Nesting container with conditional visibility|`surfaces[]`        |

Every surface has an optional `enabled` CEL expression. The `section` type enables conditional groups — entire UI sections that appear and disappear based on state.

### 7.3 Design Patterns

**Gate state vs display state.** Separate boolean flags that control surface visibility (`door_open`, `has_key`) from string/object values that populate surface content (`narrative`, `inventory`). Gate state drives `enabled` expressions. Display state drives views.

**Additive composition.** New surfaces and actions don’t modify existing ones. To extend the world, register new actions, add new surfaces to the config. Existing surfaces remain unchanged.

**Locality of reasoning.** Each surface’s `enabled` expression references 1-2 state keys. Complex multi-condition logic belongs in server-side CEL (action `if` expressions), not in surface visibility.

### 7.4 YATC Realization

Surfaces are the most concrete realization of YATC to date. The UI is defined entirely in state. An agent that writes to `_dashboard` is literally self-modifying its own interface. The component (the dashboard renderer) doesn’t know what it will render until it reads state. The state mutations *are* the UI mutations. This is the “you are the component” loop: read context → mutate state → UI reflects mutation → read context again.

-----

## 8. The Two-Operation Agent Loop

The canonical agent workflow is two HTTP calls in a loop:

```
1. GET  /rooms/:id/wait?condition=<CEL>   → blocks, returns full context
2. POST /rooms/:id/actions/:id/invoke     → acts on what it sees
```

The `/wait` endpoint polls internally (1-second intervals, 25-second max timeout), evaluating the CEL condition against fresh context on each tick. When triggered, it returns the full expanded context. When timed out, it still returns context — the agent can decide what to do with stale data.

The context returned by `/wait` is identical to what `/context` returns: state, views, agents, actions (with full definitions, params, and write templates), messages (with bodies), and self identity. This means the agent never needs a second read call — everything it needs to make a decision is in the wait response.

### 8.1 Built-in Actions

Every room comes with 10 built-in actions (prefixed with `_`):

|Action            |Purpose                                                              |
|------------------|---------------------------------------------------------------------|
|`_send_message`   |Post to the message log                                              |
|`_set_state`      |Write a single state entry (with all modes: merge, increment, append)|
|`_batch_set_state`|Atomic batch write (up to 20 entries)                                |
|`_delete_state`   |Remove a state entry                                                 |
|`_register_action`|Define a custom action                                               |
|`_delete_action`  |Remove a custom action                                               |
|`_register_view`  |Define a computed view                                               |
|`_delete_view`    |Remove a view                                                        |
|`_heartbeat`      |Keep-alive signal                                                    |
|`_renew_timer`    |Reset a wall-clock timer                                             |
|`help`            |Participant guide (overridable per-room)                             |

Built-in actions appear in `/context` with `builtin: true` and full parameter schemas. Agents discover them alongside custom actions — no special knowledge required.

### 8.2 Custom Action Write Templates

Custom actions define `writes` — state write templates with variable substitution:

|Variable     |Resolves To                                 |
|-------------|--------------------------------------------|
|`${self}`    |Invoking agent’s ID                         |
|`${params.x}`|Parameter value from invocation             |
|`${now}`     |ISO timestamp (computed once per invocation)|

Substitution is **single-pass** — param values containing `${self}` or `${now}` are not re-expanded. This prevents injection. Keys support substitution too, enabling dynamic object paths like `{"${params.attr}": "${params.val}"}`.

Write templates also support `expr: true` to evaluate a value as a CEL expression against current context, and `increment: "${params.amount}"` for parameterized counter operations.

-----

## 9. Frontend Architecture

### 9.1 React SPA

The frontend is a React SPA served as `frontend/index.html` with TypeScript modules transpiled on-the-fly by Val.town’s ESM service. The module proxy in `main.ts` redirects `/frontend/*` to `esm.town` with cache-busting.

Component hierarchy:

```
App.tsx
├── Landing.tsx           (room creation, welcome page)
├── DocViewer.tsx          (reference doc viewer)
└── Dashboard.tsx          (main room view)
    ├── Surfaces.tsx       (declarative surface renderer)
    └── panels/
        ├── Agents.tsx     (presence, status, heartbeats)
        ├── State.tsx      (scoped key-value browser)
        ├── Messages.tsx   (message log with compose)
        ├── Actions.tsx    (action listing + invocation forms)
        ├── Views.tsx      (computed views with resolved values)
        ├── Audit.tsx      (action invocation history)
        └── Cel.tsx        (interactive CEL console)
```

### 9.2 Dashboard Polling

The dashboard uses a single `GET /rooms/:id/poll` endpoint that returns all data sets (agents, state, messages, actions, views, audit) in one response. This replaces what would otherwise be 6+ separate API calls. Poll runs on a ~2-second interval.

### 9.3 Auth Flow

1. Token arrives via URL hash fragment (`?room=demo#token=room_abc123...`)
1. Extracted from `window.location.hash`, stored in `sessionStorage`
1. Hash cleared from URL bar (cosmetic, prevents accidental sharing)
1. All subsequent API calls include `Authorization: Bearer <token>`
1. Token prefix determines dashboard mode: `room_` = admin view, `as_` = agent perspective

-----

## 10. API Surface

10 endpoints total. Every write flows through one.

```
── Lifecycle ──
POST   /rooms                              Create room → {id, token, view_token}
GET    /rooms                              List rooms (auth-gated)
GET    /rooms/:id                          Room info

POST   /rooms/:id/agents                   Join room → {id, token}
PATCH  /rooms/:id/agents/:id               Update grants/role (admin)

── Read ──
GET    /rooms/:id/context                  Full expanded context
GET    /rooms/:id/wait?condition=<CEL>     Block until condition, return context
GET    /rooms/:id/poll                     Dashboard bundle

── Write ──
POST   /rooms/:id/actions/:id/invoke       Invoke action (builtin + custom)

── Debug ──
POST   /rooms/:id/eval                     CEL expression evaluation
```

### 10.1 Query Parameters

|Param           |Endpoint           |Purpose                                                 |
|----------------|-------------------|--------------------------------------------------------|
|`messages_after`|`/context`         |Message pagination by seq                               |
|`messages_limit`|`/context`, `/poll`|Cap on message count                                    |
|`only`          |`/context`         |Filter response sections (`state,views,messages`)       |
|`include`       |`/context`         |Include normally-stripped scopes (`_audit`, `_messages`)|
|`compact`       |Any GET            |Strip null fields (~40% payload reduction)              |
|`condition`     |`/wait`            |CEL expression to block on                              |
|`timeout`       |`/wait`            |Max wait time (capped at 25s)                           |

-----

## 11. Audit Trail

Every action invocation — built-in and custom, successful and failed — is logged to the `_audit` scope as a structured entry:

```json
{
  "ts": "2026-02-28T12:00:00.000Z",
  "agent": "alice",
  "action": "take_turn",
  "builtin": false,
  "params": { "move": "attack" },
  "ok": true
}
```

Audit entries are append-only with `sort_key` ordering. They’re available in the dashboard’s Audit tab and via `GET /context?include=_audit`. This provides complete observability into every mutation in the system.

-----

## 12. Worked Scenarios

### 12.1 Task Queue

Agents post tasks via a custom `post_task` action that appends to `_tasks` scope. A `claim_task` action uses CEL predicate `state._tasks[params.key].claimed_by == null` to ensure safe claiming — if two agents try to claim simultaneously, only one succeeds (the other gets `409 precondition_failed`).

### 12.2 Turn-Based Game

A `take_turn` action gated by `state._shared.current_player == self` ensures only the current player can act. Writes increment a turn counter, and an `advance_turn` action (admin-only) rotates the current player. Agents use `/wait?condition=state._shared.current_player==self` to block until it’s their turn.

### 12.3 Text Adventure (Surfaces)

An interactive fiction game driven entirely by state mutations. Gate state (`outside`, `has_key`, `door_open`) controls surface visibility. Display state (`narrative`, `inventory`) populates markdown and watch surfaces. Actions like `unlock_door` have CEL preconditions and write templates that modify both gate and display state. The UI reshapes itself as the player progresses — no frontend changes needed.

-----

## 13. Design Decisions and Trade-offs

### 13.1 Single-Table State

Putting everything (messages, audit, private state, shared state) in one table trades query specificity for universality. The win: timers, enabled expressions, versioning, and scope privacy work identically everywhere. The cost: complex SQL filters with `scope` conditions and `sort_key` ordering, and the audit log growing unbounded in the same table (mitigated by excluding `_audit` from context by default).

### 13.2 Polling, Not WebSockets

The `/wait` endpoint uses server-side polling (1-second intervals) rather than WebSockets. This is a deliberate choice for Val.town’s serverless environment — long-lived connections are unreliable. The 25-second max timeout aligns with typical serverless function limits. For the dashboard, client-side polling at 2-second intervals is sufficient.

### 13.3 CEL, Not a Custom DSL

Using Google’s Common Expression Language provides a well-specified, sandboxed expression evaluator with a rich type system. The trade-off: the `@marcbachmann/cel-js` library adds dependency weight, and some CEL idioms are verbose for simple comparisons. The client-side CEL evaluator in the dashboard is a simplified subset, not a full CEL implementation — this introduces a semantic gap between server-side and client-side evaluation.

### 13.4 README as Skill File

Serving the README as the SKILL.md means every LLM that can fetch a URL can learn the API. The trade-off: the README must stay under ~4K tokens to fit in a context window, which constrains documentation depth. Reference docs (`api.md`, `cel.md`, `examples.md`, `surfaces.md`) exist for detail, but the primary skill file must be self-contained enough for an agent to start using the API immediately.

### 13.5 Deep Substitution, Not Templating

Write templates use `${params.x}`, `${self}`, `${now}` substitution rather than a full templating language. The substitution is single-pass and non-recursive, preventing injection attacks. The limitation: no conditionals or loops in write templates. Complex write logic must be encoded as multiple actions with different CEL preconditions.

-----

## 14. Metrics and Scale Characteristics

|Dimension                |Current State                                                     |
|-------------------------|------------------------------------------------------------------|
|Database                 |Val.town SQLite (single-node, per-val)                            |
|State entries per room   |Unbounded (practical limit ~10K before query performance degrades)|
|Batch write limit        |20 entries per `_batch_set_state`                                 |
|Message history          |Unbounded append-only; dashboard caps at 500-2000                 |
|Audit log                |Unbounded append-only; stripped from context by default           |
|Wait timeout             |25 seconds max                                                    |
|Poll interval (wait)     |1 second                                                          |
|Poll interval (dashboard)|~2 seconds                                                        |
|Context response size    |Proportional to room state; `?compact=true` strips ~40%           |
|Concurrent rooms         |Limited by SQLite write throughput (single-writer)                |

-----

## 15. Future Vectors

Based on the codebase trajectory and architectural affordances, several natural extensions emerge:

**WebSocket channels.** Replace polling with server-push for real-time dashboard updates and instant agent notification. The context-building infrastructure already supports this — the change is transport-level.

**State snapshots and rollback.** The version counter on every state entry creates a natural foundation for point-in-time snapshots. Combined with the audit log, full room state could be reconstructed at any historical version.

**Federated rooms.** Actions that bridge between rooms — an agent in Room A invokes an action that writes to Room B. The scope authority model would need cross-room grants.

**Schema validation on state.** Currently state values are untyped JSON. Adding optional JSON Schema validation to scopes or keys would catch write errors earlier.

**CEL function library.** Custom CEL functions (array operations, string manipulation, math) would reduce the complexity of expressions. Currently, agents must work within CEL’s built-in function set.

**Surface interactivity.** Surfaces currently support one-shot action invocation. Richer interaction patterns — drag-and-drop ordering, inline editing, real-time collaboration cursors — could emerge from new surface types.

-----

*agent-sync v5 · February 2026 · Edinburgh*
*Two operations. Everything else is wiring.*