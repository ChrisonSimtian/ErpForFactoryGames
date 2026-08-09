# 0029. Per-game libraries as standalone repos publishing `ErpForFactoryGames.*` packages

- Status: Accepted
- Date: 2026-08-09
- Deciders: Chris
- Amends: [0014](0014-pure-csharp-save-ingestion-via-fork.md) (resolves its open
  "permanent fork" question), [0021](0021-migrate-from-nuke-fork-to-fallout.md)
  (completes its `GITHUB_TOKEN` retirement follow-up in full)

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
working copy — so the extractor does not build at all on a fresh clone. We
vendored it for exactly the reason the tool's csproj stated: upstream's published
NuGet (1.2.2) could not parse Satisfactory 1.x's IoStore container header, so we
built from `master` to pick up fixes promptly. **We do not modify CUE4Parse.**
The problem was a stale published package, not missing patches.

That premise has since expired. Upstream now publishes **dated rolling builds of
`master`** to nuget.org (`1.2.2.202607`, `1.2.2.202608`) — newer than the commit
our submodule pinned (2026-05-13). The package now *is* the master build the
submodule was hand-rolling. Verified against Satisfactory build 444486: with the
package substituted for the submodule, the extractor mounts
`FactoryGame-Windows.utoc` cleanly and regenerates `known-resource-nodes.json`
**byte-identical** to the committed dataset (647 nodes — 472 / 34 / 18 / 123 by
class, plus the 2,662 intentionally-dropped deposits).

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

3. **CUE4Parse is consumed straight from upstream — no fork, no mirror, no
   submodule.** `SatisfactoryPakExtractor` takes a `PackageReference` on
   upstream's `CUE4Parse` from nuget.org, pinned to a dated rolling build
   (`1.2.2.202608` at time of writing). The `vendor/CUE4Parse` submodule and its
   `ProjectReference` are removed, which also retires the tool's `net8.0` pin —
   it moves to `net10.0` like the rest of the repo. Bumping the datestamp after
   major Satisfactory patches replaces bumping the submodule pointer.

   **We fork only when we need to.** Today we have no patches to carry, and
   upstream ships fresher builds than our pin. If a future game patch outruns the
   published builds, fork then — and send the fix upstream first, since
   `FabianFG/CUE4Parse` is actively developed and carrying local patches against
   a fast-moving upstream is the expensive failure mode.

4. **One feed: nuget.org.** Both our own libraries and our third-party
   dependencies come from nuget.org. No GitHub Packages feed remains once the
   Satisfactory library is published, so `nuget.config` reduces to a single
   source with no credentials block and no `packageSourceMapping` workaround.

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
- **Mirror CUE4Parse into the org and republish our own rebuild of it.**
  Rejected — this was the plan until upstream's dated rolling builds were found.
  Republishing would mean maintaining a build pipeline whose entire output is a
  byte-for-byte equivalent of a package upstream already ships, under a name that
  invites confusion about which is canonical, with our name on 601-star
  Apache-2.0 code we contributed nothing to, and Apache-2.0 attribution
  obligations to discharge — all to solve a problem that no longer exists.
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
- We now depend on upstream CUE4Parse continuing to publish rolling builds. If
  that stops, or a Satisfactory patch outruns them, we fork at that point — the
  extractor is offline tooling, so the blast radius of a stale pin is a
  re-extraction we can't run yet, not a broken app.

Licence obligations that follow:

- **`SatisfactorySaveNet` (MIT).** Preserve upstream copyright notice; state the
  lineage to `R3dByt3/SatisfactorySaveNet` in the README and package metadata.
- **`CUE4Parse` (Apache-2.0).** Nothing beyond normal consumption, now that we
  neither modify nor redistribute it. The `NOTICE`/state-changes obligations only
  bite if we later fork and ship a build.

Follow-up work (tracked as
[#310](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/310)–[#313](https://github.com/erp-for-factory-games/ErpForFactoryGames/issues/313)):

- Reserve the `ErpForFactoryGames.*` prefix on nuget.org — needed before the
  first push, and it is the one step with an external dependency.
- Sequence the nuget.org publish **before** the org transfer, so we do not
  republish to a GitHub Packages feed we then abandon (feed URLs do not redirect
  on transfer; repo URLs do).
- The CUE4Parse dependency chain pulls `Microsoft.Bcl.Memory` 9.0.0, which has a
  known high-severity advisory (GHSA-73j8-2gch-69rq). Build-time-only and offline,
  so exposure is limited, but it should not sit unexamined.
- The Satisfactory catalogue/pak extraction currently in `tools/` moves into the
  new repo as `…​.Catalogue` — that is a later step than the save library, and
  should not block it.
