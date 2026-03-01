---
title: The same architecture keeps emerging across every project
status: seedling
created: 2026-03-01
---

[[the-dark-room-problem]] named the convergence: three independently-developed systems — a web game's progressive disclosure, a game testing framework's agent coordination protocol, and a self-modifying component architecture — are the same system at different scales.

[[playtest|Playtest]]'s full technical design makes the evidence overwhelming. The framework is 51,000 lines of TypeScript, 176 composable mechanics, 18 playable games, 6 evolutionary phases — and every structural decision it made, independently, recurs in [[sync-parc-land]], [[Contextual]], [[dygram]], and this [[digital-gardens|garden]] itself.

Here are the principles. They weren't designed. They were discovered — by building the same thing repeatedly and noticing.

-----

## 1. State is the universal substrate

Every system centers on a single shared state store. Not message passing, not RPC, not orchestrated calls. Raw observable state that all participants read from and write to.

**Playtest**: `game.json` on the filesystem. The engine is the authoritative source of truth. Agents interact exclusively through CLI commands that read and mutate this file. File locks handle concurrency. Lock-free reads via atomic write-then-rename guarantee consistent snapshots.

**sync.parc.land**: SQLite key-value table. Messages, audit logs, agent presence, game state, task queues — all scoped entries in the same table. "This uniformity is the key design insight." Timers, enabled expressions, and version-checking work identically across all data types.

**A Dark Room**: `$SM` StateManager. Every module reads from it. Every action writes to it. `$.Dispatch('stateUpdate')` notifies everyone. The entire game topology is latent in the data.

**This garden**: the corpus of markdown files. Each note is a state entry. [[bi-directional-links]] are the query mechanism. The build system reads all files and produces a navigable site. The files ARE the database.

The conviction: the simplest coordination mechanism is a shared data structure that everyone can see. Everything more complex is built on top of that, never instead of it.

-----

## 2. Two operations, everything else is wiring

Every participant — agent, player, component, gardener — reduces to: **perceive context**, then **take action**. This loop is the irreducible primitive.

**sync.parc.land** names it explicitly: `GET /wait?condition=<CEL>` (block until something is true, receive full context) + `POST /actions/:id/invoke` (act on what you see). "Every multi-agent system, regardless of domain, reduces to two operations — read context and invoke actions. Everything else is wiring."

**Playtest**: `player:turn` (block until it's your turn + receive available actions) → `player:act` (submit action). The `player:turn` optimization — combining the wait and the perception into a single command — is the two-op loop compressed for performance.

**Contextual/YATC**: `useEffect` (perceive through React's lifecycle) → state mutations (act) → render (express). The `useReasoning` hook is the perception step that invokes the LLM.

**This garden**: read (follow links, discover connections) → write (create or edit a note). The log is the capture loop; the garden is the accrual.

-----

## 3. Self-activation through predicates

Components don't get orchestrated into existence. They declare their own emergence conditions and self-activate when conditions are met.

**A Dark Room**: `isAvailable()` on every module and event. `Path.init()` fires when `stores.compass > 0`. No sequence table. No phase manager. "The topology of the game is latent in the data, expressed as a set of independent conditions that happen to form a progression."

**sync.parc.land surfaces**: every surface has an `enabled` CEL expression. Sections appear when their condition is true. Disappear when it's not. "A surface is a component that knows when it's relevant."

**Playtest mechanics**: `requires` and `conflicts` declarations plus auto-enable logic. Infrastructure mechanics activate when their dependencies are satisfied. No game designer explicitly lists them. The registry self-assembles.

**This garden**: [[links-signal-intent-before-content-exists]]. A broken link is a predicate that hasn't been satisfied yet. When the note is written, the link activates. Backlinks accumulate before pages exist, creating pressure to fill the gap. The [[missing]] virtual page is literally a list of unsatisfied predicates.

-----

## 4. Composition through observation, not orchestration

No central controller sequences participants. Each observes shared state independently and responds. New participants don't modify existing ones.

**Playtest**: agents never communicate directly. The gamemaster doesn't run every turn — it enters a blocking wait and only wakes for disputes. Players observe game state via filtered views and submit actions independently. The mechanic registry's hook system lets multiple mechanics respond to the same event — merge all responses, take the first, or block until agreement. No mechanic knows about any other mechanic. They all respond to the same hooks.

**sync.parc.land**: "No routing layer, no controller. The room's structure IS the protocol." Surfaces observe state independently. Actions gate on state conditions. Adding a new surface or action requires changing nothing that already exists.

**A Dark Room**: modules don't coordinate directly. They all read from and write to `$SM`. Each reacts independently. "There's no central game loop that says 'first Room, then Outside, then Path.' There's just shared state and independent reactors."

**Surfaces as substrate** names it: the "coral reef pattern." Individual organisms respond to local chemical signals without awareness of the whole structure. The reef emerges.

This is also how [[evergreen-notes]] work. Each note is self-sufficient. Notes reinforce each other through links, not through explicit orchestration. The network structure is emergent.

-----

## 5. Atomic units that accrete

Small, self-contained units that compound through connection. One idea. One mechanic. One action. One surface.

**This garden**: [[atomicity-forces-clarity]]. One idea per note. "If you can't isolate a concept, you don't understand it." Each note is durable, findable, linkable. [[knowledge-work-should-accrete|Knowledge work should accrete]]: time spent writing compounds because the output remains useful.

**Playtest**: one mechanic per file. 138 leaf mechanics, each composable through hooks. The strangler fig extraction (Phase 4) moved logic from a monolithic `game.ts` into atomic mechanic files — each extraction broke the test suite and was repaired, proving the unit was correctly isolated. The test harness replays logged games action by action.

**sync.parc.land**: one action = one coherent state transition. "Actions don't sequence — they gate." Write to the grain you gate on.

**DyGram**: one node per concept. Node types (Task, State, Context, Input, Output) compose through typed arrows. The graph accretes as the design evolves.

The accrual mechanism is always the same: write a small unit → link it to existing units → the network becomes more valuable → future work gets easier. Same mechanism in notes, mechanics, surfaces, state entries.

-----

## 6. Absence is generative

Missing things aren't errors. They're invitations. The system surfaces what doesn't exist yet and creates pressure to fill valuable gaps.

**This garden**: broken links are fine — they signal intent. [[links-signal-intent-before-content-exists]]: "Creating a link to a non-existent page is a creative act." The [[missing]] page lists all broken links. Frequently-linked stubs reveal important gaps.

**Surfaces as substrate**: "Absence is Signal." A key that doesn't exist yet is information, not an error. `state._shared.discovered == true` evaluates false when `discovered` has never been written. Design for accretion — new keys appear as the experience unfolds.

**A Dark Room**: capabilities don't hide behind locked tabs. They literally don't exist in the DOM until their predicate fires. "The system isn't withholding — it's growing."

**Playtest**: the 16 design proposals include open investigations and speculative futures — named gaps the system hasn't filled yet. The mechanic reference library (192 entries) includes mechanics not yet implemented. The gap is visible and inviting.

-----

## 7. Descriptions that execute

The declarative description IS the executable artifact. The gap between specification and implementation collapses.

**Playtest**: "Games are data, not code." A new game is a `RULES.md` file with YAML configuration and natural-language rules. No TypeScript changes required. The engine reads the config, enables appropriate mechanics, and runs the game. 18 games, zero game-specific code.

**DyGram**: "Designs should execute immediately, not after implementation." Sketch a state machine, run it. Deterministic transitions fire instantly; complex decisions invoke AI reasoning.

**sync.parc.land**: the README IS the SKILL.md. Any Claude instance can fetch the skill file and immediately know how to use the API. The documentation doesn't describe the system — it teaches the system to agents.

**Agent skills**: [[agent-skills-extend-ai-coding-assistants|SKILL.md files]] package procedural knowledge into portable, version-controlled units. The skill description is the skill.

**Literate programming**: [[literate-programming|code written primarily for humans to read]]; execution is secondary. Explanation drives structure.

**This garden**: the markdown files are simultaneously the source, the database, and (after build) the site. `docs/**/*.md → dist/**/*.html`. The notes don't describe the knowledge — they ARE the knowledge.

-----

## 8. Evolution through use, not specification

Things grow through use. Start rough, refine through feedback. The final form can't be designed in advance because you don't yet know what you'll learn.

**This garden**: seedling → budding → evergreen. "Evolved. Notes grow. Seedlings become evergreen. Update > append > new file." [[epistemic-status]] labels give permission to publish rough work and make growth visible.

**Playtest**: six evolutionary phases — breadth, first contact, bug sprint, extraction, hardening, reflection. The strangler fig extraction wasn't planned; it was forced by the weight of a monolith that couldn't adapt. Each phase was a response to what the previous phase revealed.

**Contextual**: "You didn't program it. You used it into being." The component perceives how you use it, reasons about whether its current form serves your objective, and rewrites itself.

**The vibes**: "keep iterating / developing / growing / evolving how it works and what it does as I use it... so that we can transcend."

**A Dark Room / the-dark-room-problem**: "Not one that was designed in advance and gradually unlocked. Not one that was generated once and sits inert. An interface that perceives what you're doing, reasons about what you might need next, authors new capabilities into existence at the moment you're ready for them."

-----

## The convergence

These principles weren't extracted from one system and applied to others. They were independently discovered in a digital garden, a game testing framework, a multi-agent coordination substrate, a self-modifying component architecture, and a DSL for executable state machines. Each system was built to solve its own problem. Each arrived at the same structural decisions.

[[the-dark-room-problem]] saw this first and named three layers: Disclosure (predicates), Protocol (capability negotiation), Agency (self-modifying components). Playtest adds the fourth: Memory (the append-only log that makes trajectory-aware decisions possible, which the garden has as [[log-captures-fleeting-thoughts|the log]], sync has as the `_audit` scope, and A Dark Room has as accumulated `$SM` state).

What Playtest contributes that wasn't visible before:

- **The hook resolution strategies** (merge, first, blocking) are a concrete answer to "how do independent observers compose when they disagree." The garden doesn't have this problem because notes don't conflict. Games do. Agents do. This is the [[Contextual|ctxl]] problem — what happens when two agent-components want the same space — and Playtest has a working solution in production.

- **The strangler fig evolution** is the most detailed evidence for principle 8. The commit history shows each extraction breaking tests, being repaired, and being validated through manual playtesting. Growth through accretion documented at the commit level.

- **The contest system** is dispute resolution as a first-class primitive. Neither the garden nor A Dark Room need it — they're single-participant systems. But multi-agent coordination requires a way to challenge and reverse actions. sync.parc.land has CAS version conflicts and action preconditions. Playtest has the full pattern: contest → block → adjudicate → allow-or-reverse.

- **"Games are data, not code"** is the most complete instance of principle 7. RULES.md files with YAML frontmatter and markdown rules — the same format as garden notes, agent skills, and sync.parc.land room configs. The convention is fractal.

-----

## What you're circling

The [[vibes-2026-02-28|vibes note]] captures it in one compressed sentence:

> Hey chat, [as a human] I want to [ambition], you do the [hard part], establish the [interface] between us, so I can use it to [accomplish] my goal [undefined]. It should [automatically] know how I want to use it, keep [iterating/developing/growing/evolving] how it works as I use it, so that we can [transcend].

Every project is an attempt to build one piece of this. The garden builds the accrual substrate (principles 1, 5, 6). Playtest builds the coordination protocol (principles 2, 3, 4, and the hook resolution strategies). Contextual builds the self-modification loop (principles 7, 8). sync.parc.land tries to be the minimal viable intersection.

The thing that doesn't exist yet — the thing all of these are orbiting — is the integration: a system where the state substrate, the self-activation predicates, the compositional accrual, and the evolutionary self-modification all operate together. Where the interface grows with you not because someone designed the growth, but because the architecture makes growth the natural consequence of use.

The fire is where you start. What comes after depends on what you do.
