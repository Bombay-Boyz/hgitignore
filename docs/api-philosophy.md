# API Philosophy

This document governs what is allowed to become part of `hgitignore`'s
public, semver-covered API (see [`stability.md`](stability.md) for the
exact module list this applies to).

## Core Rule

**The public API remains minimal.** Every additional exported name is a
permanent commitment under semantic versioning — removing or changing it
later is a breaking change. The bar for adding something is "this is
needed," not "this might be convenient."

## Process for Adding to the Public API

1. **Demonstrate the need.** A real call site (in `hgitignore`'s own
   filesystem layer, in a test, or in a linked external issue/use case) that
   cannot be reasonably served by composing existing exports.
2. **Check whether it belongs in `Internal`.** If the need is narrow,
   internal-only, or experimental, it goes under `HGitIgnore.Internal.*`
   first — unstable, not semver-covered, and promotable to public later if
   demand persists.
3. **Document it in the same PR** that adds it, including in
   [`SPEC.md`](../SPEC.md#public-api) if it changes the core signatures
   list there.
4. **Get sign-off in API review** — see the
   [Release Checklist](../SPEC.md#release-checklist) item; for additions
   between releases, the same scrutiny applies at PR-review time rather
   than waiting for a release.

## Specific Rules

1. **Public API remains minimal.** Prefer one well-designed function over
   three convenience overloads.
2. **Additions require demonstrated need.** Speculative "someone might want
   this" additions are rejected. This includes speculative configuration
   options, flags, or `Maybe`-wrapped behavior toggles.
3. **Internal modules are unstable by design**, not by accident — this is
   what lets the implementation evolve (e.g. swapping the DFA
   representation in Phase 9, or adding the `MatchTrace` type from
   [Future Work](../SPEC.md#future-work)) without breaking users.
4. **Convenience wrappers are discouraged.** If a caller needs
   `compileFile` followed by a default-error-handling pattern, that
   composition belongs in the caller's code, not as a new exported
   function, unless it's needed so broadly that *not* providing it would be
   the surprising choice (rare — treat this as a high bar, not a low one).
5. **Fewer exports are preferred over more**, even when an additional
   export would save a few lines at typical call sites. The long-term cost
   of an extra permanent API surface generally outweighs the short-term
   convenience.

## What This Means Concretely

- No `compileOrThrow` partial-function convenience export — callers pattern
  match on `Either CompileError IgnoreMatcher` themselves, per the project's
  total-functions stance (see [`SPEC.md`](../SPEC.md#type-safety-requirements)).
- No exposed mutable matcher-building API (e.g. an incremental "add this one
  pattern" function) ahead of actual demonstrated need — `compile` taking
  the full pattern text is the v1 contract. Incremental updates are tracked
  explicitly as v3 [Future Work](../SPEC.md#future-work), not added early
  "just in case."
- `ErrorCode` constructors are added only when a genuinely new error
  condition is identified — not pre-allocated "for future use."

## Reviewing This Document Itself

Changes to this philosophy document are rare and should be treated with the
same scrutiny as a public API change, since it governs all future API
decisions. Propose changes via PR with explicit rationale, not as a drive-by
edit alongside an unrelated feature.
