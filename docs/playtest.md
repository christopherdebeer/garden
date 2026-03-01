---
title: Playtest is a game-agnostic agentic playtesting framework
status: seedling
created: 2026-03-01
---

[[project]]

A framework where AI agents play board games against each other. Define a game's rules in a markdown file, spin up a gamemaster and some player agents, and let them play. The engine handles state, randomization, and turn management. The agents handle decision-making and rule interpretation.

Three design principles:

1. **Engine owns state** — All game state, randomization, and deck management lives in a TypeScript engine. Agents never touch state directly.
2. **Agents make decisions** — The gamemaster interprets rules and adjudicates disputes. Players choose actions and compete to win.
3. **Games are data, not code** — A new game is a `RULES.md` file with YAML configuration and natural-language rules. No TypeScript changes required.

18 games — from UNO to hidden-role social deduction — run on the same engine with the same agent architecture, differentiated only by their rules files and which mechanics they enable. 176 composable mechanics across 19 categories.

## The mechanic registry

The mechanic registry is where it gets interesting architecturally. Each mechanic self-registers with:

- **Requirements**: what other mechanics must be active
- **Conflicts**: what mechanics can't coexist
- **Hooks**: ~50 composition points where mechanics contribute behaviour
- **Resolution strategies**: when multiple mechanics respond to the same hook — merge all responses? Take the first? Block until all agree?

This isn't really a game engine pattern. It's a [[recurring-architecture|capability negotiation protocol]] wearing a board game costume. Replace "mechanic" with "service" or "agent capability" and nothing changes structurally.

## The contest system

When a player believes another's action violated the rules, they file a contest. The game blocks. The gamemaster adjudicates — allow or reject. If rejected, the mechanic's `reverseAction()` hook unwinds the state. A 60-second timeout auto-allows to prevent deadlock.

This is dispute resolution as a first-class primitive. It reappears in [[sync-parc-land]] as action preconditions and version conflicts.

## Evolutionary history

The git history tells the story of a system that grew through playtesting:

1. **Mechanics expansion** — building the 192-mechanic library
2. **First contact with reality** — first full playtest, every game stalled
3. **Bug fix sprint** — 118 issues, 6 critical systemic bugs
4. **The great extraction** — strangler fig pattern, monolith to composable hooks
5. **Specific game hardening** — making complex games (social deduction, simultaneous selection) actually work
6. **Architecture reflection** — recognizing the patterns generalize beyond games

Phase 4 is the most instructive. Each extraction moved logic from a monolithic core into mechanic hooks, broke the test suite, repaired it, and validated with manual playtests. [[knowledge-work-should-accrete|Growth through accretion]], not design-from-above.

## Connection to other projects

Playtest's architecture converges with [[sync-parc-land]] and [[Contextual]]. See [[the-dark-room-problem]] for the full argument: three independently-developed systems are the same system at different scales. See [[recurring-architecture]] for the named principles.

The distributed agent CLI proposal in the playtest repo explores extracting the engine into a general multi-agent coordination framework — recognizing that file-based state, blocking waits, hook-based extensibility, and dispute resolution are domain-invariant patterns.

See also: [[dygram]], [[agent-skills-extend-ai-coding-assistants|agent skills]]
