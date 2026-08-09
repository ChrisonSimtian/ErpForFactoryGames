# 0029. Per-game libraries as standalone repos publishing `ErpForFactoryGames.*` packages

- Status: Accepted
- Date: 2026-08-09
- Deciders: Chris
- Amends: [0014](0014-pure-csharp-save-ingestion-via-fork.md) (resolves its open
  "permanent fork" question), [0021](0021-migrate-from-nuke-fork-to-fallout.md)
  (completes its `GITHUB_TOKEN` retirement follow-up, with one scoped exception)

## Context

ERP depends on two out-of-repo bodies of game-parsing code, and their situations
are opposites — a distinction that was not obvious until measured.

**`SatisfactorySaveNet`** is a runtime dependency (`PackageReference` 4.1.6 in
`Satisfactory.Infrastructure.csproj`). We consume a fork at
`ChrisonSimtian/SatisfactorySaveNet`, which as of 2026-08-09 is **21 commits
ahead of and 0 commits behind** upstream `R3dByt3/SatisfactorySaveNet`. Upstream's
last substantive commit was 2025-12-18 (~8 months ago; 18 stars, 4 open issues).
Every save-version fix since — v1.2 TOC/Data Blob, deep-parse
`ArrayProperty<StructProperty>`, chain-actor ExtraData fallback, the save v60
resilience fix — originated in the fork. [ADR 0014](0014-pure-csharp-save-ingestion-via-fork.md)
left the endgame explicitly open: the fork is source of truth "until either
upstream merges (cutoff: 4 weeks of no maintainer response) or we accept a
permanent fork". The upstream PR (#33) is closed without a merge. The cutoff has
long passed. In practice we are no longer a fork of a maintained project; we are
the maintained line, wearing a fork's label.

**`CUE4Parse`** is the reverse. It is build-time-only — a `ProjectReference` from
`tools/SatisfactoryPakExtractor` to the `vendor/CUE4Parse` submodule, run offline
to emit committed datasets (`known-resource-nodes.json`, `known-flora.json`). The
fork at `ChrisonSimtian/CUE4Parse` is **0 ahead, 0 behind** upstream
`FabianFG/CUE4Parse` — a pristine mirror whose recent commits are upstream
authors', not ours. `vendor/CUE4Parse` is not even checked out in a normal
working copy. We vendor it for exactly the reason the tool's csproj states:
upstream's published NuGet (1.2.2) cannot parse Satisfactory 1.x's IoStore
container header, so we build from `master` to pick up fixes promptly. **We do
not modify CUE4Parse.** The problem is a stale published package, not missing
patches.

Two further forces:

- Both forks sit under a personal account (`ChrisonSimtian`) while the
  `erp-for-factory-games` org holds exactly one repo. Ownership does not match
  where the work actually lives.
- This is about to stop being a two-game question. Captain of Industry already
  needs its own out-of-band extraction (`tools/CaptainOfIndustryExtractor`,
  loading `Mafi.*.dll` in an `AssemblyLoadContext`), and
  [Outworld Station](0022-captain-of-industry-support.md) — another Unreal title
  — will need pak extraction too. Whatever shape we choose is a template, not a
  one-off.

## Decision

**Each game's parsing and extraction code becomes a standalone repository in the
`erp-for-factory-games` org, publishing versioned `ErpForFactoryGames.*` packages
to nuget.org. ERP consumes released packages only — no submodules, no
`ProjectReference` into vendored source.**

Concretely:

1. **Naming.** Repo per game, packages per concern under one reservable prefix:

   | Repo | Packages |
   |---|---|
   | `erp-for-factory-games/Satisfactory` | `ErpForFactoryGames.Satisfactory.Saves`, `…​.Saves.Abstracts`, `…​.Catalogue` |
   | `erp-for-factory-games/CaptainOfIndustry` | `ErpForFactoryGames.CaptainOfIndustry.Catalogue` |
   | `erp-for-factory-games/OutworldStation` | `ErpForFactoryGames.OutworldStation.Catalogue` (pending the #309 spike) |

2. **SatisfactorySaveNet becomes ours.** The fork is transferred to the org and
   rebranded as a hard-fork successor, following the same playbook ADR 0021
   documents for NUKE → Fallout: 1:1 namespace mapping, README stating the
   lineage, upstream MIT copyright preserved. Version numbering **continues the
   4.x line** rather than resetting — the package-id change already
   disambiguates, and the existing correspondence between release numbers and
   game save versions is worth keeping (Fallout set the same precedent by
   continuing NUKE's `10.x`).

3. **CUE4Parse is not forked in any meaningful sense.** We keep a mirror and
   publish **unmodified rebuilds of pinned upstream commits** as
   `ErpForFactoryGames.CUE4Parse` to the org's **GitHub Packages** feed. The
   `vendor/CUE4Parse` submodule and the `ProjectReference` are removed;
   `SatisfactoryPakExtractor` takes a `PackageReference`, which also retires its
   `net8.0` pin rationale. If we ever genuinely need to patch CUE4Parse, the
   fix goes upstream first — see [#313](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/313).

4. **Two feeds, deliberately.** Our own libraries go to **nuget.org** (public,
   tokenless). The CUE4Parse rebuild goes to **GitHub Packages** under the org.
   This is affordable because `tools/` is *outside* `ErpForFactoryGames.slnx` (31
   projects, none under `tools/` or `vendor/`): the main solution restores
   without any token, and `GITHUB_TOKEN` becomes a prerequisite only for the
   out-of-band extractor tools.

## Alternatives considered

- **Keep vendoring both as submodules.** Rejected. The submodules are already
  `update = none, ignore = all` — off the build graph, invisible to `git status`,
  and in CUE4Parse's case not even checked out. They provide the *illusion* of
  vendoring without its benefit (reproducibility), while imposing its cost
  (a second source of truth). A pinned package version does the job honestly.
- **One monorepo holding every game library.** Rejected. It re-couples release
  cadences that have nothing to do with each other — a Satisfactory save-version
  hotfix should not need a CoI catalogue release — and it reintroduces the
  cross-repo coordination the split is meant to remove.
- **Publish everything to GitHub Packages.** Rejected on the evidence of
  ADR 0021: GitHub Packages NuGet requires authentication even for public
  packages, which is what forces `GITHUB_TOKEN`, `packageSourceCredentials`, and
  the `packageSourceMapping` workaround in `nuget.config` today.
- **Publish the CUE4Parse rebuild to nuget.org too.** Rejected. Upstream owns the
  `CUE4Parse` name there; shipping a near-identical public package under a
  different id invites confusion about which is canonical, and puts our name on
  601-star Apache-2.0 code we contributed nothing to. Internal is the honest
  scope for a convenience rebuild.
- **Keep the `SatisfactorySaveNet` name.** Rejected. The nuget.org id is taken by
  upstream, so a suffix would be needed regardless; and the name does not
  generalise to the per-game pattern this ADR is establishing.
- **Reset the Satisfactory package to 1.0.0.** Rejected — see Decision §2.

## Consequences

What becomes easier:

- `dotnet restore` on `ErpForFactoryGames.slnx` needs **no token at all** once
  the Satisfactory packages land on nuget.org. `nuget.config` loses its
  `packageSourceCredentials` block and its `packageSourceMapping` workaround, and
  CI drops the `packages: read` grant. This is the follow-up ADR 0021 anticipated.
- Adding game number four is a repo-creation template, not an architecture
  discussion.
- The libraries become independently useful. Given upstream Satisfactory parsing
  is dormant, a public, maintained package is a real contribution to that
  community rather than a private convenience.

What becomes harder:

- We are now unambiguously on the hook for maintaining a save parser across game
  patches, with external consumers who may file issues. That was already true in
  practice; publishing makes it explicit and visible.
- Iterating on a library now costs a package release rather than a submodule
  edit. Mitigation: keep the `vendor/` checkout habit available for local
  iteration, but never on the build graph.
- Two feeds means `nuget.config` does not become trivial — it stays non-empty for
  the extractor tools, just no longer on the main build's critical path.

Licence obligations that follow:

- **`SatisfactorySaveNet` (MIT).** Preserve upstream copyright notice; state the
  lineage to `R3dByt3/SatisfactorySaveNet` in the README and package metadata.
- **`CUE4Parse` (Apache-2.0).** Preserve `NOTICE`; because we publish a build,
  the package must state that it is an *unmodified rebuild of upstream `master`
  at `<sha>`*. If that ever stops being true, the "state significant changes"
  clause applies and the package description must change with it.

Follow-up work (tracked as
[#310](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/310)–[#313](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/313)):

- Reserve the `ErpForFactoryGames.*` prefix on nuget.org — needed before the
  first push, and it is the one step with an external dependency.
- Sequence the nuget.org publish **before** the org transfer, so we do not
  republish to a GitHub Packages feed we then abandon (feed URLs do not redirect
  on transfer; repo URLs do).
- The Satisfactory catalogue/pak extraction currently in `tools/` moves into the
  new repo as `…​.Catalogue` — that is a later step than the save library, and
  should not block it.
