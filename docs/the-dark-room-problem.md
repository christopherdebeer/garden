---
title: The Dark Room Problem
status: seedling
created: 2026-02-16
---

# The Dark Room Problem

*On interfaces that don’t exist yet, protocols that discover themselves, and the architecture of not-knowing-in-advance*

-----

## 1.

You’re in a dark room. The fire is dead. You can light it.

That’s the entire game. One button. One resource. One verb.

You light the fire. The room warms. A stranger stumbles in. She can build things, she says — but she needs wood. A new tab appears: the forest. You didn’t ask for it. You didn’t configure it. The interface *grew* because you did something, and the system decided you were ready for more.

[If you haven’t played A Dark Room, play it before reading this. Not because I’ll spoil it — I will — but because the experience of the interface expanding under your hands is the thesis of this essay, and describing it is not the same as feeling it.]

-----

## 2.

Here’s what the source code looks like. Each module — Room, Outside, Path, World, Ship, Space — is an independent object with an `init()` method that creates its own DOM panel and tab. The engine doesn’t orchestrate their appearance. Instead, each module’s activation is gated by a predicate against shared state:

```
if (typeof $SM.get('stores.wood') != 'undefined') {
    Outside.init();
}
if ($SM.get('stores.compass', true) > 0) {
    Path.init();
}
```

No sequence table. No phase manager. No `currentLevel` counter. The topology of the game is *latent in the data*, expressed as a set of independent conditions that happen to form a progression. Each module knows its own emergence conditions and self-activates when they’re met.

The events work the same way. Every random event has an `isAvailable()` function:

```
isAvailable: function() {
    return Engine.activeModule == Room && $SM.get('stores.fur', true) > 0;
}
```

The nomad can’t arrive until you have fur. You can’t have fur until you have traps. You can’t have traps until the builder is awake. The builder can’t wake up until the room is warm. The room can’t be warm until you light the fire. The fire is where you start. The entire game is encoded as a dependency graph of predicates, but the player experiences it as *discovery*.

This is the thing. The player doesn’t know the graph exists. They don’t know there are six locations, or that the game eventually goes to space. They experience each expansion as surprising-but-inevitable — *of course* there’s a forest, you need wood. *Of course* there’s a path, you have a compass now. The architecture produces the feeling of organic emergence precisely because it isn’t scripted. It’s conditional.

-----

## 3.

I’ve been building a game testing framework called Playtest. AI agents play board games against each other via CLI commands, with a TypeScript engine managing state, turn order, and rules validation. A gamemaster agent adjudicates disputes. Player agents wait for their turn, read the game state, and submit actions.

The architecture is simple: the engine owns a JSON file, agents interact through `./playtest act`, `./playtest wait`, `./playtest status`. File locks handle concurrency. The CLI is the contract — any process that can shell out is a valid participant. An LLM agent and a bash script are mechanically identical.

[Standard multi-agent stuff. Nothing remarkable yet. Stay with me.]

What got interesting was the mechanic system. Games are composed of reusable mechanics — `card-matching`, `dice-rolling`, `trick-taking`, `worker-placement` — and each mechanic self-registers with:

- **Requirements**: “I need the `cards` mechanic to be active”
- **Conflicts**: “I can’t coexist with `trick-taking`”
- **Hooks**: 20+ composition points where mechanics contribute behaviour
- **Resolution strategies**: when multiple mechanics respond to the same hook, do we merge all responses? Take the first? Block until all agree?

This isn’t really a game engine pattern. It’s a capability negotiation protocol wearing a board game costume. Replace “mechanic” with “service” or “agent capability” and nothing changes structurally.

And the engine itself doesn’t care who calls it. Any process with filesystem access can register, wait for a turn, submit an action. The CLI interface is the most universal IPC boundary there is. The coordinator that spawns agents and assigns roles is a convenience, not a requirement. The engine already implies a distributed, loosely-coupled system that the orchestration layer constrains back into a centralised one.

[I wrote a proposal exploring what happens if you remove the constraint. If agents discover sessions rather than being spawned into them. If they bring capabilities rather than being assigned roles. If the protocol evolves rather than being authored in advance. The full document is in the playtest repo. The short version: you end up with a CLI-native, shared-state, multi-agent coordination system with dispute resolution that happens to have been prototyped as a board game engine.]

-----

## 4.

Separately — or so I thought at the time — I’ve been building Contextual.

[ctxl. Formerly prompt studio. The name keeps changing but the idea hasn’t moved since I wrote it down in April 2024: a tool for making tools that do stuff, without requiring the user to learn to program in the traditional sense.]

The architecture I arrived at — after a long detour through prompt templates, evaluation frameworks, and chain interfaces — is described in the companion essay “You Are The Component.” The short version: a React component where the LLM isn’t a service the component calls but the reconciliation logic itself. The component IS the agent. State is memory. Effects are perception. Render is expression. And the component can rewrite its own source code, recompile via esbuild-wasm in the browser, and keep running with state preserved through React Refresh.

A component that doesn’t have source code yet. You tell it what you want. It authors itself into existence. You use it. It perceives how you use it — which buttons you press, which data you provide, where you hesitate. It reasons about whether its current form serves your objective. If it doesn’t, it rewrites itself. You didn’t program it. You *used* it into being.

That essay ended with an admission: the mechanism works but the product doesn’t exist yet. The architecture can support adaptive interfaces. But I hadn’t articulated *how* the adaptation should feel — what the experience of a tool that reveals itself should actually be like.

-----

## 5.

Then I looked at A Dark Room’s source code and the three projects collapsed into one.

-----

## 6.

Here’s the connection, and I need to build it in layers because the full picture only makes sense once you have all the pieces. Which is, I suppose, the point.

**Layer one: disclosure as predicate.**

ADR’s modules activate when state conditions are met. The predicates are simple — `stores.wood != undefined`, `stores.compass > 0`. But they produce the experience of progressive revelation because the player’s actions are what change the state, and the player doesn’t know what states will trigger what revelations.

ctxl needs the same pattern. When a user starts with “I want to track my reading habits,” the interface should begin minimal — maybe just a text input and a list. As they add entries, the system accumulates state. When enough entries exist, a visualisation component could emerge — not because a timer fired, but because the predicate `entries.length > 10 && entries.some(e => e.date)` became true. The user created the conditions for the interface to expand, by using it.

This is not “progressive onboarding.” Onboarding reveals features that already exist. ADR reveals features that *didn’t exist until the conditions were met*. The Path module doesn’t hide behind a locked tab. It literally doesn’t exist in the DOM until `Path.init()` fires. The distinction matters because it means the system isn’t withholding — it’s growing. And ctxl’s agent-components, which can author themselves into existence, make this literal rather than metaphorical. The visualisation component doesn’t appear from behind a curtain. It gets written, compiled, and mounted, because the parent agent-component reasoned that now is the time.

**Layer two: coordination as protocol.**

ADR’s modules don’t coordinate with each other directly. They all read from and write to a shared state object (`$SM`), and a pub/sub system (`$.Dispatch('stateUpdate')`) notifies everyone when anything changes. Each module reacts independently. There’s no central game loop that says “first Room, then Outside, then Path.” There’s just shared state and independent reactors.

This is exactly the playtest engine pattern: shared state (JSON file), independent agents (CLI processes), event notification (file watchers). And it’s exactly how ctxl’s agent ecology should work. Multiple agent-components in a React tree, sharing state through a common store, each independently deciding whether to mount, unmount, or rewrite themselves based on state predicates. No orchestrator. No sequence. Just conditions and responses.

But ctxl introduces something ADR and the current playtest engine don’t have: *negotiation*. In ADR, modules can’t conflict — the designer ensured they compose cleanly. In a user-driven system where agent-components author themselves into existence, conflicts are inevitable. Two siblings might both want to render a full-width chart. A child might request capabilities its parent can’t provide. An agent-component might rewrite itself into something that breaks its contract with its neighbors.

This is where playtest’s capability registry becomes essential. The `requires` / `conflicts` / `defines` system, the hook resolution strategies (merge, first, blocking), the contest-based dispute resolution — these are the mechanisms for agents to coexist without a central authority deciding the layout. The registry doesn’t tell agents what to do. It tells them what they *can* do, given what everyone else is doing.

**Layer three: the experience of emergence.**

Here’s what ADR gets right that most adaptive systems get wrong: the player never feels the system deciding. There’s no “loading new module” message. No “you’ve unlocked a new feature!” popup. No tutorial. The tab just appears. The button just shows up. The world just turns out to be bigger than you thought.

This is a design principle, not a technical one, and it’s the hardest part to implement in ctxl. When a new agent-component mounts because the user’s accumulated state crossed a threshold, how do you make that feel like discovery rather than interruption? How do you make the user feel “of course, that’s what I needed” rather than “what just changed?”

ADR’s answer: the new capability is always *implied by what you already have*. You have a compass → of course you can now explore the world. You have fur → of course a trader might show up. The connection between the enabling state and the enabled capability is intuitive even if the player didn’t predict it.

For ctxl, this means the disclosure predicates can’t be arbitrary. A visualisation component shouldn’t appear because `entries.length > 10` — that’s a mechanical threshold. It should appear because the agent *reasoned* that the user’s data has temporal structure, and temporal structure benefits from visual representation, and the user has demonstrated engagement with the data (editing entries, returning to the app, adding dates). The predicate should encode a *narrative* about readiness, not a count.

[This is where the LLM earns its place in the architecture. A static predicate can check `entries.length > 10`. Only a reasoning system can check “does this person’s behaviour suggest they’d benefit from a chart right now?” The `useReasoning` hook, fired when the parent’s dependencies change, is the mechanism for narrative predicates. The model looks at the user’s trajectory and decides whether a new capability should emerge. The decision is expensive — an API call — but it doesn’t happen on every render. It happens when the dependency array changes, same as `useEffect`. Perception through React’s lifecycle. Reasoning through the model. Disclosure through component mounting.]

-----

## 7.

So the system has three layers and I can now name them.

**The Disclosure Layer** decides what exists. It evaluates predicates — some simple (does this data have dates?), some reasoned (is this user ready for complexity?) — and determines which agent-components should be alive at any given moment. It is the parent agent’s responsibility. It is ADR’s `isAvailable()` generalised.

**The Protocol Layer** manages coexistence. When multiple agent-components are alive, they need to negotiate space, capabilities, and state access. This is the playtest mechanic registry extracted: self-registration, dependency/conflict resolution, hook-based composition, dispute adjudication. It doesn’t tell agents what to render. It tells them what resources they can claim, what hooks they can participate in, and how conflicts get resolved.

**The Agency Layer** is the agent-component itself — the living, reasoning, self-modifying unit from “You Are The Component.” It perceives through effects, reasons through the LLM, acts through state mutations, and expresses through render. And it can rewrite its own source when its current form doesn’t serve the user’s evolving objective.

These layers compose:

```
Disclosure: "The user has enough temporal data. Mount a timeline."
Protocol:   "Timeline requires 'visualisation' capability. Chart component
             already claims full-width. Resolve via merge: split the viewport."
Agency:     [Timeline agent authors itself into existence. Reasons about the
             user's date patterns. Renders a visualisation. Observes whether
             the user engages. If not, signals to the disclosure layer that
             it should unmount.]
```

The user sees: a timeline appeared. It looked at my data. It showed me something I hadn’t noticed. I didn’t ask for it, but I’m glad it’s here.

-----

## 8.

There’s a fourth layer that I’ve been circling without naming. ADR has it. Playtest has it in a limited form. ctxl needs it and doesn’t have it yet.

**The Memory Layer.**

In ADR, the state manager persists everything to localStorage. When you return, the fire is where you left it. The builder remembers you. The world map is still explored. The game has *continuity*.

But more than that: the state accumulates a *history* that shapes future possibilities. The number of huts you’ve built determines the population. The population determines which events can fire. Your inventory of teeth and scales determines what the nomad will sell you. The state isn’t just “where you are” — it’s the sediment of every decision, and the system reads that sediment to decide what happens next.

In playtest, this is the transition log — every action, every dispute, every adjudication. The log is append-only and forms the ground truth for the session’s history. Agents can read it to understand trajectory, not just current state.

ctxl needs both. The agent-component’s external state store preserves current state across self-modifications. But the system also needs to accumulate interaction history — what the user did, what the agent showed them, what they engaged with, what they ignored, what the agent tried and rolled back. This history is the substrate for the disclosure layer’s narrative predicates. “Is this user ready for a chart?” is a question about trajectory, not snapshot.

[The evaluation loop from the original ctxl design fits here. Measures — quantitative assessments of whether the tool is serving its objective — aren’t a separate system. They’re the disclosure layer reading the memory layer. Did the user engage with the last capability that was revealed? Did their behaviour change after the interface expanded? Are they using the tools the system gave them, or working around them? These signals feed back into the predicates that gate future disclosure.]

-----

## 9.

I want to be precise about what I’m proposing and what I’m not.

I’m *not* proposing a fully autonomous system that decides what the user needs without their input. The user starts with “I want to…” and that objective anchors everything. The disclosure layer reveals capabilities that serve the stated objective. The user can always override, dismiss, request. The system adapts; the user directs.

I’m *not* proposing that every interface should be generated from scratch by an LLM. Most interfaces should be designed by humans and implemented normally. The adaptive system is for a specific class of problem: tasks where the user doesn’t know the shape of the tool they need because they don’t yet understand the shape of their problem. Exploratory work. Data sense-making. Creative projects. Learning. Situations where the tool and the understanding co-evolve.

What I *am* proposing is that three independently-developed systems — a web game’s progressive disclosure architecture, a game testing framework’s agent coordination protocol, and a self-modifying component architecture — are the same system at different scales, and their convergence describes something that doesn’t exist yet:

An interface that grows with you.

Not one that was designed in advance and gradually unlocked. Not one that was generated once and sits inert. An interface that perceives what you’re doing, reasons about what you might need next, authors new capabilities into existence at the moment you’re ready for them, negotiates their composition without central coordination, and remembers everything so that each emergence is informed by the full history of your interaction.

The fire is where you start. What comes after depends on what you do.

-----

## 10. The Parts

For the record, and for my own future reference, the constituent architecture:

**Agent-Component** (from ctxl / “You Are The Component”)
The unit of intelligence. A React component embodying an LLM agent. Perceives through effects. Reasons through a `useReasoning` hook. Acts through state mutations. Expresses through render. Can rewrite its own source via esbuild-wasm and survive through React Refresh. Externalised state ensures continuity across self-modifications.

**Capability Registry** (from Playtest / MechanicRegistry)
The coordination substrate. Agent-components self-register with capability declarations: what they provide, what they require, what they conflict with. Hook-based composition with resolution strategies (merge, first, blocking) governs how multiple agents contribute to shared concerns. Dispute resolution handles conflicts that static rules can’t predict.

**Disclosure Predicates** (from A Dark Room / `isAvailable()`)
The emergence conditions. Each potential agent-component has a predicate — ranging from simple state checks to LLM-reasoned narrative assessments — that determines whether it should exist. Predicates are evaluated by parent agents when their dependencies change. Components that pass their predicate mount. Components whose predicate is no longer met unmount. The interface breathes.

**Interaction Memory** (from ADR’s state accumulation + Playtest’s transition log + ctxl’s evaluation loop)
The history substrate. Every user action, every agent decision, every disclosure event, every self-modification, every engagement signal — appended to an immutable log. The disclosure layer reads this log to make trajectory-aware decisions. The evaluation system reads it to assess whether recent disclosures served the objective. The agent-components read it for self-awareness about their own evolution.

**Session Protocol** (from Playtest’s Distributed Agent CLI proposal)
The participation contract. Defines what roles exist, what transitions are valid, what capabilities each role can access, and how disputes are resolved. For ctxl, the session is the user’s working context. The protocol is authored by the initial objective and evolves as capabilities are disclosed. Agent-components join the session by satisfying capability requirements. They leave when their predicate fails.

-----

## 11.

[I don’t have this built. I have the component architecture as a proof of concept. I have the mechanic registry in production for game testing. I have the progressive disclosure pattern understood from reading someone else’s source code. The convergence is clear in my head and now in this essay. The implementation is not.

The next move is probably the smallest possible integration: a ctxl prototype where a single parent agent-component can reason about user state and spawn a child agent-component based on a disclosure predicate. Not the full registry. Not the full memory layer. Just: you do something, and the interface grows. One tab appearing. The A Dark Room experience, produced by the ctxl mechanism, without the game.

If that works — if it *feels* right, if the disclosure feels like discovery rather than automation — then the protocol layer and the memory layer have a substrate to build on.

If it doesn’t, I’ll know something about the gap between architecture and experience that I don’t know yet.

Either way: the room is dark, and the fire is dead, and you can light it. What happens after that is the interesting part.]