# 0030. Game repositories are onion-layered, with CLIs as presentation-layer tools

- Status: Accepted
- Date: 2026-08-09
- Deciders: Chris
- Extends: [0029](0029-standalone-game-libraries-as-nuget-packages.md)
- Applies: [0004](0004-use-onion-architecture.md) to the game repositories

## Context

[ADR 0029](0029-standalone-game-libraries-as-nuget-packages.md) established one
repository per game, publishing `ErpForFactoryGames.*` packages that the planner
consumes. It did not say what goes *inside* one, and the first attempt got that
wrong in an instructive way.

Captain of Industry was moved out as a console app with its extraction logic in
`Program.cs`. That put every piece of game-specific knowledge inside a
presentation binary: the planner could not consume any of it, the only output
was a file on disk, and the packaging question — library or `dotnet tool`? —
had no good answer because the thing was both and neither.

Two further forces:

- **A game needs more than one kind of game-specific code.** Every supported game
  has a *catalogue* (what the game can make) and *saves* (what this player has
  built). They have opposite cadences: the catalogue is read once per game patch,
  as part of setting a player up; the save is read repeatedly while they play, to
  produce a snapshot of the factory. Anything else game-specific belongs with
  them rather than leaking into the planner.
- **The expensive part is narrow.** Reading a Captain of Industry catalogue means
  loading the game's assemblies and executing its registration code. Reading an
  Outworld Station save means parsing a UE 5.4 binary format. In both cases that
  is one adapter, and everything around it — the model, the use case, the
  persisted output — has no such requirement.

## Decision

**Each game repository is onion-layered exactly as [ADR 0004](0004-use-onion-architecture.md)
describes for this repository, holds every game-specific concern for that game,
and ships its CLIs as presentation-layer .NET global tools released alongside the
libraries.**

Concretely:

1. **Layers.** `src/Domain`, `src/Application`, `src/Infrastructure`,
   `src/Presentation`, with `test/` mirroring `src/` layer for layer. Dependencies
   point inward only; Domain has no references at all.

2. **Scope.** A game repository owns the catalogue reader, the save parser, and
   anything else specific to that game. The planner consumes packages; it never
   learns a game's file formats.

3. **CLIs are presentation.** Each repo publishes a global tool
   (`ToolCommandName`: `erp-coi`, `erp-outworld`) whose commands are thin wrappers
   over Application use cases. The game agent runs the catalogue command once when
   setting a player up, and the snapshot command frequently while they play. Every
   command also has a `--json` path, so the agent and a human share one binary.

4. **A package per layer**, not one per repo. A consumer that only wants the
   model shouldn't drag in an adapter that loads game assemblies. The tool is its
   own package again.

5. **Domain types carry a game prefix** (`CoiRecipe`, `StationModule`). The
   planner consumes several game packages at once, so bare `Recipe` or `Module`
   would collide across them and force using-aliases at every call site.

6. **Serialisation contracts live in Infrastructure DTOs, never in Domain.** The
   Captain of Industry catalogue is the cautionary case: its on-disk shape calls
   products `items` and the game version `coiVersion`. Keeping that at the edge
   lets the domain be named sensibly while old files keep loading, with a test
   pinning the wire names.

7. **Publish on a version tag**, via nuget.org **trusted publishing** (OIDC), with
   no long-lived API key in any repository. Not on every merge to main: a
   published version is permanent and cannot be replaced, so publishing should be
   a decision rather than a side effect. The inherited SatisfactorySaveNet
   workflow published on every main commit, and would have fired on the first
   push into the org — see [#311](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/311).

## Alternatives considered

- **Keep the CLI-with-logic-inside shape.** Rejected — it was the starting point
  and it fails the basic requirement: the planner cannot consume a console app.
  It also made the packaging question unanswerable.
- **One rolled-up package per game.** Rejected. A consumer wanting only the domain
  model would pull the adapter that loads game assemblies, along with its
  transitive dependencies, for nothing.
- **Keep tools outside the architecture, per [ADR 0026](0026-onion-layered-src-with-product-split.md)'s
  `tools/` rule.** Rejected for game repositories. That rule governs *this* repo,
  where tools are one-shot local utilities. In a game repository the CLI is the
  product's user interface and is shipped to players' machines, which is what a
  presentation layer is.
- **A single tool binary across all games.** Rejected. It would have to depend on
  every game package at once, so installing it for one game pulls the others, and
  a release of any game forces a release of the tool.

## Consequences

What becomes easier:

- The agent can `dotnet tool install` per game and drive setup and snapshots
  through a documented CLI rather than bespoke plumbing.
- Use cases are testable with no game installed. Both `ExtractCatalogue` and
  `CaptureSnapshot` are unit-tested against fakes; only the adapters need the
  real thing, and those tests skip when it is absent.
- Adding a game is a template rather than a design exercise.

What becomes harder:

- Four projects and several packages per game, all needing version coordination.
  Acceptable for the isolation it buys; revisit if it becomes a chore.
- A save-format or game-data change now touches an adapter *and* possibly the
  domain projection. That is the intended trade: the blast radius is inside one
  repository instead of across the planner.

Follow-up implied:

- **Satisfactory has not been restructured yet.** It is a hard fork with its own
  build and layout, so it is the largest of the three; until then the three game
  repositories are not uniform.
- The `tools/SatisfactoryPakExtractor` and `tools/CaptainOfIndustryExtractor`
  copies in this repository are superseded and should be retired
  ([#319](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/319)).
- Captain of Industry still has **no save parser**, so its repo is catalogue-only;
  Outworld Station's catalogue is blocked behind a `.usmap`
  ([#318](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/318)).
  Neither gap is architectural — both are work.
