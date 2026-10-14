# Demo scope: the quality bar for the game roadmap changes

The changes in `docs/roadmap.md` (0 to 11) exist to make an AI-workshop **demo** work.
They are not a production release. The artifacts were written thoroughly, with
benchmarks, golden masters, currency checks and accessibility requirements, so the
agent that applies them must know which of that is **required** and which is
**optional ballast** when time is short.

> The test for every task: does it help the demo moment of this change in
> `docs/roadmap.md` work reliably? If yes, do it. If it only hardens something
> beyond the demo, it may be deferred.

## How to defer

Deferring is allowed, silent dropping is not. When you skip a task because of demo
scope, leave its checkbox unchecked and append `(deferred: demo scope)` to the line.
List every deferral in the pull request description under "Deferred for demo scope".
Stretch tiers (for example the coaster editor's loops and inversions) are skipped by
default; MVP tiers are not.

## May be relaxed

- **Performance budgets and benchmarks.** Treat numbers as sanity targets. Do not add
  BenchmarkDotNet projects or gate on timings. Make it fast enough that the demo runs
  smoothly with the intended crowd size; measure only if it visibly stutters.
- **Coverage floor.** The 80 % line-coverage rule from `CLAUDE.md` does not apply to
  new game modules in these changes. Still write tests for the core invariants (ECS
  lifecycle, scene-graph ordering, determinism of the simulation, golden masters where
  the change says the old engine is the oracle) because they catch real breakage.
- **Equivalence and tolerance tiers.** One comparison that proves old and new engines
  agree is enough; skip the extra tiers.
- **Currency checks.** Do a quick check that a chosen package is not deprecated and
  builds on the pinned SDK. Skip the deeper upstream-release-date research.
- **Accessibility.** Keep keyboard operation and labels on new panels where it is cheap.
  Axe specs for every new panel are optional.
- **Documentation derivations, authoring guides, validation scripts, visual regression,
  git-lfs, diagnostics endpoints, extra dashboards.** Optional unless the demo moment
  needs them.
- **Edge cases.** Code must be neat and readable and handle the normal flow and the
  obvious failures (bad input from the UI, a missing file, an empty park). It does not
  need to cover every rare combination, overflow, race or recovery path. Where a spec
  lists an exotic scenario, implement it only if it is cheap or the demo can hit it.
- **Security hardening.** No threat modelling, authentication or authorisation work, rate
  limiting, input-size caps, tamper-proof saves or checksum/signature schemes beyond what
  already exists. The demo runs on a trusted machine. Tasks that exist only for
  hardening (for example save-file integrity, hostile-input validation, replay-attack
  guards) may be deferred.
- **Plan size.** Each change may stop at the first task group that delivers its demo
  moment, then continue only if time allows.

## Must not be relaxed

- **A green build at every commit** and the existing ride tests keep passing. The
  working Octopus is the foundation of the demo; do not break it.
- **No destructive actions without confirmation,** no secrets in files or logs, no
  merging: the agent opens the pull request and never merges. Relaxed security above
  does not extend to these: never commit credentials, even for a demo.
- **Clean code.** Short-cuts are in coverage of rare cases, not in readability: no dead
  code, no leftover debug output, honest names, and the module and folder conventions of
  `CLAUDE.md` still hold.
- **Legal and originality:** original names and art, our own numbers, no copied code or
  assets from RollerCoaster Tycoon or OpenRCT2.
- **Reproducible demo:** seeded randomness and the fixed simulation step stay, because a
  demo that behaves differently on every run cannot be rehearsed.
- **Honesty in text:** commit messages and pull requests say what the diff does and
  describe the result as demo-grade, not production-ready.
- **The preconditions gate (`doctor`)** from change 0, because a missing tool on the
  demo machine is the most likely way for the demo to fail.

## Where this document is read

`docs/vision.md` links here. Each change's `proposal.md` carries a one-line pointer to
this file. If `CLAUDE.md` and this file disagree, `CLAUDE.md` is the repo-wide rule and
this file is the owner's stated intent for the game roadmap: stop and ask the owner
before applying the stricter or the looser reading when it affects more than a few lines.
