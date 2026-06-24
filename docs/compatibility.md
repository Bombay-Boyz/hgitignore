# Compatibility Policy

## Supported Git Versions

- **Latest stable Git release** at the time of each `hgitignore` release.
- **The two prior minor versions** of Git, on a best-effort basis.
- The **minimum supported version** for parity-testing purposes (see
  [`conformance.md`](conformance.md)) is recorded here once decided during
  [Phase 2](ROADMAP.md#phase-2--conformance-specification) — placeholder
  until then. Both the minimum and the latest are run through the
  [Phase 12](ROADMAP.md#phase-12--git-parity) parity pipeline on every
  release, and both must pass.

If Git itself changes ignore-matching behavior in a way that breaks parity
against an older supported version, that is treated as a compatibility
incident: the supported-version floor is raised (with a note in the
changelog) rather than `hgitignore` trying to special-case both old and new
Git behavior internally, unless the divergence is trivial to support for
both simultaneously.

## Supported Operating Systems

- **Required for v1:** latest major Linux distributions — current Ubuntu
  LTS, current Debian stable, current Fedora — verified in CI.
- **Tracked, not blocking for v1:** macOS, Windows. Path-separator handling
  (see [`semantics.md` §8](semantics.md#8-path-normalization)) is written to
  accommodate both, but CI coverage and the
  [Phase 12](ROADMAP.md#phase-12--git-parity) parity pipeline running on
  these platforms is not a v1 release blocker. This is reassessed at or
  before [Phase 15](ROADMAP.md#phase-15--release) based on actual demand.
- **Filesystem case-sensitivity** (relevant on default macOS/Windows
  filesystems) is explicitly out of scope for matching semantics — see
  [`semantics.md` §9](semantics.md#9-known-edge-cases).

## Reference Hardware

Recorded during [Phase 14](ROADMAP.md#phase-14--benchmarking) — the
[performance targets](../SPEC.md#performance-targets) in `SPEC.md` are only
meaningful relative to a documented machine. Placeholder until that phase's
PR fills this in.

## Deprecation Policy

- A supported Git version is only dropped from the support matrix in a
  major `hgitignore` release, with a changelog entry explaining why.
- A supported OS is only dropped in a major release, same rule.
- Dropping support is distinct from a version simply no longer being tested
  in CI due to unavailability (e.g. an end-of-life distro) — when that
  happens, the changelog should note the gap even if no deliberate decision
  was made to drop it.
