# Surfaces as Substrate

*Design principles for composable experiences in [[/sync]]*

-----

## The Observation

A surface is a declarative view over mutable shared state. An action is a state transition with preconditions. Together they form something more interesting than either: **an experience that assembles itself from the state it observes.**

This is not a state machine. State machines require you to enumerate transitions upfront. This is closer to how a coral reef works — individual organisms (surfaces, actions) respond to local chemical signals (state) without awareness of the whole structure. The reef *emerges*.

The dark room game revealed what happens when you fight this. Compound boolean flags (`found_cabin && !has_map && outside`), coupled write sequences, surfaces whose visibility depends on the precise history of state changes — these are symptoms of trying to build a state machine using coral.

The following principles describe how to work *with* the substrate rather than against it.

-----

## I. Absence is Signal

A key that doesn’t exist yet is not an error. It’s information.

The substrate must handle absence gracefully — `state._shared.discovered == true` should be false when `discovered` has never been written, without requiring someone to have written `discovered: false` first. This is a property of the evaluator, not the experience designer.

**Why this matters:** If you require all state to exist before it’s referenced, you’ve created a coupling between setup and every future addition. You can’t add a new surface that gates on `state._shared.found_secret_door` without going back and initializing `found_secret_door: false` in the room setup. That’s not composable — it’s a manifest.

Design for accretion. State grows over time. New keys appear as the experience unfolds. A well-designed surface handles the moment *before* its key exists exactly as gracefully as the moment after.

-----

## II. Locality of Reasoning

A surface should be understandable by looking at the surface and the state it references. Nothing else.

If understanding why a section is visible requires tracing through three actions, two other surfaces, and the ordering of writes — the coupling has won. Each surface’s `enabled` expression should read as a simple, local question:

- *“Has the player reached the cabin?”* → `state._shared.at == "cabin"`
- *“Is there something to pick up?”* → `state._shared.item_available == true`
- *“Has the door been opened?”* → `state._shared.door_open == true`

Not:

- *“Has the player found the cabin AND not yet taken the map AND still has the key AND didn’t go back inside?”*

When you find yourself writing compound conditions with more than two clauses, you’re encoding a *narrative dependency graph* into a visibility expression. That graph belongs in the actions (which perform transitions), not the surfaces (which observe results).

**The test:** Can someone unfamiliar with the experience read a single surface config and understand when it appears? If the answer requires knowledge of action ordering or write side-effects, refactor.

-----

## III. Actions as Boundaries

An action is a *narrative boundary* — the moment between one state of the world and another. It should produce a coherent, self-contained state change.

This means:

**One action, one conceptual transition.** “Search the cabin” should produce *one* atomic result: the cabin has been searched. Whether that means setting a flag, updating inventory, and writing narrative text — all of it happens in one action’s write batch, and the post-state is internally consistent.

**Actions don’t sequence — they gate.** An action’s `if` expression asks “is this transition valid right now?” It doesn’t ask “have the previous three transitions happened in order?” If you need ordering, it should emerge from state: action A writes a key, action B’s precondition checks for that key. But B doesn’t know about A — it only knows about the key.

**Write to the grain you gate on.** If surfaces gate on `at == "cabin"`, then the action that moves you there should write `at: "cabin"`. If surfaces gate on `door_open == true`, the action should write `door_open: true`. The vocabulary of action writes and surface conditions should be the *same vocabulary*. When they diverge — actions writing booleans that surfaces must recombine into compound expressions — you’ve split the concept across two representations.

-----

## IV. The Grain of State

Not boolean soup. Not a monolithic enum. Something in between.

The dark room’s mistake was using many fine-grained booleans (`tried_door`, `looked_shelf`, `found_key`, `door_open`, `outside`, `found_cabin`, `has_map`, `has_rope`) that individually meant little but whose *combinations* encoded the actual game state. This is the boolean soup problem: each flag is easy to write but impossible to reason about in aggregate.

The opposite extreme — a single `phase` enum — is also wrong. It forces linear progression, prevents parallel threads, and requires that every possible state be a named member of one enum. You can’t have “player is outside AND a storm is brewing AND a merchant has arrived” without an enum entry for every permutation.

**The right grain is the conceptual entity.** State keys should map to things the experience cares about as first-class concepts:

- **Location:** `at: "cabin"` — not five booleans about movement history
- **Discovery:** `door_open: true` — a single fact about the world
- **Accumulation:** `inventory: ["rope", "map"]` — a collection, displayed but not gated on
- **Atmosphere:** `narrative: "Firelight flickers..."` — display state, decoupled from logic

The heuristic: **if removing a key would make two or more surfaces wrong, it’s the right grain.** If removing a key only affects one surface, it might be too fine. If it affects everything, it’s too coarse.

-----

## V. Additive Composition

You should be able to add a new surface, action, or state key to an existing experience without modifying anything that already exists.

This is the composability test. In the dark room, adding a “secret passage in the well” would require:

1. A new action with writes
1. A new surface section
1. *Nothing else changes*

If adding the passage requires modifying the well section’s enabled condition, or changing what `descend_well` writes, or adding initialization to the room setup — the design isn’t composable.

**Surfaces compose by observing, not by being orchestrated.** A new surface that gates on `state._shared.found_secret == true` can be added to the config at any time. The state key doesn’t need to exist yet (Principle I). No other surface needs to know about it. The action that writes `found_secret: true` can be added independently.

This is the coral reef pattern. New organisms attach to the existing structure and respond to local chemistry. The reef’s architecture isn’t planned — it accretes.

**Implications for config structure:** The `_dashboard` surfaces array should read as a flat collection of independent pieces, not as a nested dependency tree. Sections can be nested for *visual grouping*, but their enabled conditions should be independent. A section nested inside another section shouldn’t rely on the parent’s condition — it should restate its own. Redundancy in enabled expressions is a feature, not a bug; it preserves locality (Principle II).

-----

## VI. Display State vs. Gate State

Not all state exists for the same reason. Conflating display state with gating state is a source of coupling bugs.

**Gate state** determines what’s visible and available: `at: "cabin"`, `door_open: true`, `quest_complete: false`. Surfaces and actions reference these in `enabled` and `if` expressions.

**Display state** provides content for what’s already visible: `narrative: "The cabin smells of pine..."`, `inventory: ["rope", "map"]`, `wood_count: 48`. Surfaces render these as markdown, watches, metrics.

**The rule:** Gate on gate state. Display display state. Don’t gate on display state.

Gating on `inventory.length > 2` means the player’s ability to progress depends on a display concern. If you later decide to change how inventory works (combining items, dropping things), you’ve broken progression gates. Instead, have the action that gives the player the critical item also write the gate key: `has_supplies: true`.

-----

## VII. Surfaces as Self-Describing Components

Each surface carries its own activation contract. It says: *“I appear when X. I show Y. I offer Z.”*

This is the connection to YATC thinking. A surface is a component that *knows when it’s relevant*. It doesn’t need an orchestrator to show or hide it. It doesn’t need to be imperatively mounted. It evaluates its own `enabled` expression against the shared state and either renders or doesn’t.

The _dashboard config is not a controller — it’s a **manifest of available components**, each with self-declared activation conditions. The runtime’s job is to evaluate those conditions and render what’s active. There is no routing logic, no transition function, no switch statement. Just components and conditions.

This means the config is also *introspectable*. You can look at the surfaces array and immediately understand the experience’s possibility space: what states the experience can be in, what the player sees in each state, what actions are available. The config *is* the experience design, not a thin UI wrapper over hidden logic.

-----

## VIII. Failure Modes and How to Smell Them

When an experience built on surfaces goes wrong, it’s usually one of these:

**The Compound Gate:** `enabled: "a == true && b == false && c == true && d != 'done'"`. This means the surface’s visibility encodes a *history of transitions* rather than a *current state*. Refactor: find the single concept this combination represents and make it a key.

**The Invisible Dependency:** Action A writes `x: true`. Surface S gates on `x == true`. But S only makes sense if action B has also been invoked, and nothing enforces that ordering. Refactor: either action A should also write the state that S actually needs, or S’s condition should include what it actually requires.

**The Write Spray:** An action writes to 5 different keys across 3 scopes. Some are gate state, some are display state, some are side-effects for other systems. Refactor: one action, one conceptual transition. Display updates can be separate writes in the batch, but the *gating* change should be one key.

**The Dead Reference:** A surface references a view or action that was renamed or removed. The surface silently disappears or shows empty. This is a tooling problem — the substrate should warn about dangling references.

**The Absence Surprise:** A surface gates on `x == false` but `x` was never initialized. Whether this works depends entirely on evaluator semantics. After our fix, absence is falsy, so this works. But it’s still worth being explicit in action descriptions about what “default” means.

-----

## Worked Example: The Dark Room, Revisited

Applying these principles to the same game:

**Gate state:** `at` (location enum: room, outside, cabin, well, descent, ending), `door_open`, `cabin_searched`, `has_compass`

**Display state:** `narrative` (markdown text for current scene), `inventory` (array), `wood_count`

**Surfaces:**

```
section "The Room" enabled="at == 'room'"
  → markdown (narrative view)
  → action-bar [look_around, try_door, search_shelf, open_door]

section "Outside" enabled="at == 'outside'"  
  → markdown (narrative view)
  → action-bar [gather_wood, search_forest]
  → watch [wood_count, inventory]

section "The Cabin" enabled="at == 'cabin'"
  → markdown (narrative view)  
  → action-bar [search_cabin, go_to_well]

section "The Well" enabled="at == 'well'"
  → ...
```

Each section: one gate condition, one location. Actions write `at` to move between locations. No compound conditions needed.

`search_cabin` writes: `cabin_searched: true`, appends to inventory (one batch write), updates narrative. It does NOT write `at` — you stay at the cabin. `go_to_well` is a separate action that writes `at: "well"`, gated on `cabin_searched == true`.

The experience is now a collection of independent scene components, each self-activating based on one piece of state. Adding a new location means adding a new section and the actions that write the new `at` value. Nothing existing changes.

-----

## Summary

|Principle                 |One-liner                                   |
|--------------------------|--------------------------------------------|
|Absence is Signal         |Unset keys are meaningful, not errors       |
|Locality of Reasoning     |Each surface understandable in isolation    |
|Actions as Boundaries     |One action = one coherent transition        |
|The Grain of State        |Concepts, not booleans or monoliths         |
|Additive Composition      |New pieces don’t modify existing pieces     |
|Display vs. Gate          |Don’t gate visibility on display concerns   |
|Self-Describing Components|Surfaces carry their own activation contract|

The substrate is the state. Surfaces observe it. Actions transform it. The experience is what emerges when you get the grain right.