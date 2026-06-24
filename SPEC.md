# hgitignore — Gold Standard Specification

### Version 2.0 (Implementation Charter)

> Supersedes v1.0. Changes from v1.0 are summarized in
> [Changelog from v1.0](#changelog-from-v10) at the end of this document.

---

## Mission

Build the canonical Haskell implementation of Git ignore semantics.

Primary goals, in priority order:

1. Exact Git compatibility.
2. Small, maintainable codebase.
3. Strong type safety.
4. Explicit error handling.
5. Stable API.
6. Long-term maintainability.
7. Conformance-driven development.

These goals are ordered deliberately. When two goals conflict — for example,
"small codebase" vs. "exact Git compatibility" on some baroque edge case —
goal #1 wins. The smallest codebase that is *wrong* is not acceptable.

---

## Guiding Principle

**Correctness > Simplicity > Performance.**

Optimization work is prohibited until conformance is complete (Phase 14 only,
after Phase 12 parity passes). This is enforced procedurally, not just
culturally: CI rejects any PR touching `benchmark/` before
`docs/ROADMAP.md` shows Phase 12 checked off.

---

## Architecture Overview

```text
hgitignore/
│
├── README.md
├── SPEC.md
├── stack.yaml
├── package.yaml
│
├── hgitignore-core/
│   ├── src/
│   └── test/
│
├── hgitignore/
│   ├── src/
│   └── test/
│
├── hgitignore-testkit/
│   ├── src/
│   └── test/
│
├── benchmark/
├── fuzz/
│
├── docs/
│   ├── semantics.md
│   ├── conformance.md
│   ├── compatibility.md
│   ├── invariants.md
│   ├── api-philosophy.md
│   ├── architecture.md
│   ├── stability.md
│   └── ROADMAP.md
│
└── test-corpus/
```

---

## Package Responsibilities

### `hgitignore-core`

```text
AST
Parser
Compiler
Matcher
Types
Errors
Normalization
```

Properties — all mandatory, enforced by module boundary and CI:

- Pure.
- No `IO`.
- No filesystem access.
- No Git process execution.

### `hgitignore`

```text
compileFile
Filesystem helpers
User-facing API
```

Depends on `hgitignore-core`. This is the only package permitted to touch
`IO`, the filesystem, or spawn processes.

### `hgitignore-testkit`

```text
Generators
Parity helpers
Fixture helpers
Corpus helpers
```

Depended on by test-suites in both other packages, and by `fuzz/` and
`benchmark/`. Never depended on by production code.

Full data-flow and dependency diagram: [`docs/architecture.md`](docs/architecture.md).

---

## Development Sequence

Implementation follows a strict phase order. The full breakdown — goals,
deliverables, and **checkable exit criteria** for every phase — lives in
[`docs/ROADMAP.md`](docs/ROADMAP.md). This section gives the index only; treat
`docs/ROADMAP.md` as authoritative for what "done" means.

| Phase | Name |
|---|---|
| 0  | Infrastructure |
| 1  | Formal Semantics |
| 2  | Conformance Specification |
| 3  | Domain Types |
| 4  | Path Normalization |
| 5  | Lexer |
| 6  | Parser |
| 7  | Validation |
| 8  | NFA Compiler |
| 9  | DFA Compiler |
| 10 | Matcher |
| 11 | Conformance Corpus |
| 12 | Git Parity |
| 13 | Fuzzing |
| 14 | Benchmarking |
| 15 | Release |

**Note on Phase 4 (renamed from "Normalization" to "Path Normalization"):**
this phase normalizes *paths being matched* (the `RelativePath` smart
constructor's job — trimming, separator canonicalization, redundant-segment
collapse), not patterns being parsed. It has no dependency on the lexer or
parser and could in principle run in parallel with Phase 5, but is ordered
first because `RelativePath` is a Phase 3 type that Phase 10's `Matcher`
consumes directly. Pattern-level normalization (e.g. handling of trailing
spaces and escape sequences in patterns) is part of Phase 6 (Parser) /
Phase 7 (Validation) instead, since it is a parsing concern, not a path
concern. This split is intentional and documented to prevent the ambiguity
present in v1.0 of this spec.

---

## Public API

```haskell
compile
  :: Text
  -> Either CompileError IgnoreMatcher
```

```haskell
compileFile
  :: FilePath
  -> IO (Either CompileError IgnoreMatcher)
```

```haskell
matches
  :: IgnoreMatcher
  -> RelativePath
  -> MatchResult
```

```haskell
matchesMany
  :: IgnoreMatcher
  -> Vector RelativePath
  -> Vector MatchResult
```

These signatures are the target public surface. They are written here ahead
of Phases 3–10 as the contract those phases build toward — not as an
implication that the API is implemented before its dependencies. See
[`docs/api-philosophy.md`](docs/api-philosophy.md) for what governs *changes* to
this surface once it exists.

---

## Type Safety Requirements

Mandatory:

- ADTs.
- Newtypes.
- Opaque constructors.
- Smart constructors.
- Total functions.
- Exhaustive pattern matching (`-Wincomplete-patterns -Werror`, enforced from
  Phase 0 onward).

Forbidden, anywhere in `hgitignore-core` or `hgitignore` production code
(test code under `test/` and `fuzz/` may use a narrow, documented exception
list — see [`docs/invariants.md`](docs/invariants.md)):

```text
head
tail
init
last
read
fromJust
error
undefined
unsafePerformIO
unsafeCoerce
```

---

## Error Architecture

Centralized in:

```haskell
module HGitIgnore.Error
```

### Stable Error Codes

```haskell
data ErrorCode
  = HG001
  | HG002
  | HG003
  | HG004
```

Examples:

```text
HG001  Invalid Escape Sequence
HG002  Invalid Pattern
HG003  Invalid Path
HG004  Validation Failure
```

Rules:

- Error codes are public API. They are covered by the same stability
  guarantees as anything in [`docs/stability.md`](docs/stability.md).
- Never reuse a retired code.
- Never renumber an existing code.
- New error conditions get the next unused number — never a gap-fill.

---

## `RelativePath`

```haskell
newtype RelativePath
```

Guarantees, enforced only through the smart constructor (no other
construction path exists):

- Non-empty.
- Relative (never absolute; constructor rejects leading `/` on POSIX or a
  drive letter / UNC prefix on Windows).
- Normalized per [`docs/semantics.md`](docs/semantics.md#path-normalization) —
  separators canonicalized, no `.`/`..` segments, no trailing separator
  except where semantically meaningful for directory-only patterns.

---

## `MatchResult`

```haskell
data MatchResult
  = Ignored
  | Included
```

No booleans — booleans don't self-document at call sites and don't leave
room to add a third state without a breaking change.

**Known limitation, tracked explicitly rather than silently accepted:** real
`git check-ignore --verbose` exposes a third practical state — *no pattern
matched at all* vs. *explicitly re-included by negation* — which matters for
some diagnostic tooling but not for the ignore/don't-ignore decision itself.
`hgitignore` v1's `MatchResult` deliberately collapses these into `Included`,
because the binary decision is what goal #1 (exact compatibility) requires
for the actual ignore behavior. A `MatchTrace` type exposing the
distinguishing detail is tracked as a candidate, additive, backward-compatible
post-v1 API addition — see [Future Work](#future-work) — and must not block
v1 release.

---

## Formal Matching Rules

This is a summary; [`docs/semantics.md`](docs/semantics.md) is authoritative and
must be written (Phase 1) before any parser or matcher code exists.

1. Normalize the candidate path.
2. Evaluate patterns in source order, across all applicable ignore files,
   in Git's own precedence order (see
   [`docs/semantics.md`](docs/semantics.md#file-precedence) — `.git/info/exclude`,
   then global excludes file, then `.gitignore` files from the repository
   root downward, with deeper files overriding shallower ones for paths
   under them).
3. Track the latest matching state per path.
4. Last matching pattern wins.
5. **Directory-exclusion exception (the part v1.0 of this spec omitted):**
   if a path is ignored because one of its *parent directories* matches an
   exclude pattern, a later negation pattern matching the path itself
   **cannot** re-include it. Git does not descend into an already-excluded
   directory to evaluate further patterns against its contents. A negation
   can only re-include a path if none of its ancestor directories are
   themselves excluded. This is the single most commonly-misimplemented
   piece of Git ignore semantics and is the reason a fixture corpus
   (Phase 11) and real-Git parity testing (Phase 12) are non-negotiable
   rather than "nice to have."
6. Return the final state as `MatchResult`.

Full algorithm, including precedence across multiple `.gitignore` files,
`**` semantics, anchoring rules (`/` at start/end), character class and
escape handling, and the directory-exclusion exception above with worked
examples: [`docs/semantics.md`](docs/semantics.md).

---

## Internal Invariants

Authoritative list: [`docs/invariants.md`](docs/invariants.md). Highlights:

- `RelativePath` is always normalized; there is no code path that produces
  an unnormalized value with that type.
- `IgnoreMatcher` is immutable once compiled.
- The compiled DFA is deterministic — no transition function ever has more
  than one possible next state for a given (state, input) pair.
- No transition table entry points to an invalid/nonexistent state.
- The `AST` is always validated (Phase 7) before it reaches the compiler
  (Phase 8); the compiler does not re-validate and is permitted to assume a
  valid `AST`.

---

## API Philosophy

Authoritative: [`docs/api-philosophy.md`](docs/api-philosophy.md). Rules:

1. Public API remains minimal.
2. Additions require demonstrated need — a real call site or a documented
   user request, not speculative convenience.
3. Internal modules (`HGitIgnore.Internal.*`) are unstable and exempt from
   semver guarantees.
4. Convenience wrappers are discouraged; prefer composition by the caller.
5. Fewer exports are preferred over more, even at minor ergonomic cost.

---

## Compatibility Policy

Authoritative: [`docs/compatibility.md`](docs/compatibility.md). Summary:

- Supported Git versions: latest stable Git release at time of each
  `hgitignore` release, plus the two prior minor versions.
- Supported OS: latest major Linux distributions (current Ubuntu LTS,
  current Debian stable, current Fedora) at minimum; macOS and Windows
  support are tracked but not blocking for v1.
- Every release runs the full parity suite (Phase 12 pipeline) against the
  minimum supported Git version, not just the latest.

---

## Stability Policy

Authoritative: [`docs/stability.md`](docs/stability.md). Summary:

- Semantic Versioning, strictly.
- Public, semver-covered modules:

```text
HGitIgnore.Compile
HGitIgnore.Match
HGitIgnore.Pattern
HGitIgnore.Types
HGitIgnore.Error
```

- Everything under `HGitIgnore.Internal.*` is unstable and may change in any
  release, including patch releases.

---

## Test Strategy

Framework: `hedgehog`.

### Required Property Tests

```haskell
parse . render == id
```

```haskell
normalize . normalize == normalize
```

```haskell
match p x == match p x   -- determinism: matching is a pure function of (matcher, path)
```

Additional required properties (added in this revision — see
[Changelog](#changelog-from-v10)):

```haskell
-- Negation cannot escape an excluded ancestor directory.
forAll (excludedDirWithNegatedChild) $ \ (patterns, path) ->
  matches (compileOrDie patterns) path == Ignored
```

```haskell
-- Compiling is deterministic: same source text, same matcher behavior.
compile t == compile t
```

Full required-property list with rationale for each: [`docs/invariants.md`](docs/invariants.md).

---

## Conformance Corpus

Directory: `test-corpus/`.

Fixture format:

```yaml
patterns:
  - "*.log"

path: app.log

expected: ignored
```

The corpus is authoritative; the implementation is secondary. If
implementation behavior and a *correct* fixture disagree, the implementation
is wrong. (If a fixture itself is found to be wrong — i.e. it doesn't match
what real Git does — the fixture is corrected via the same parity pipeline
that validates everything else; see [`docs/conformance.md`](docs/conformance.md).)

Corpus must include explicit fixtures for:

- Basic glob and wildcard patterns.
- Anchored vs. unanchored patterns.
- `**` in all documented positions.
- Negation, including the directory-exclusion exception above.
- Escape sequences and literal special characters.
- Trailing whitespace and trailing-slash directory markers.
- Multiple `.gitignore` files at different repository depths.
- `.git/info/exclude` and global excludes file precedence.
- Malformed/edge-case input that must still produce a defined result, not a
  crash.

---

## Git Parity Pipeline

```bash
stack run parity
```

Pipeline:

1. Generate fixtures.
2. Run real Git (`git check-ignore`) against each fixture.
3. Run `hgitignore` against the same fixture.
4. Compare outputs.
5. Fail the pipeline on any difference, with a diff showing pattern set,
   path, expected (Git) result, and actual (`hgitignore`) result.

Methodology and supported-Git-version matrix: [`docs/conformance.md`](docs/conformance.md).

---

## Fuzzing

Directory: `fuzz/`.

Requirements:

- The parser never crashes on any byte sequence, valid UTF-8 or not.
- The matcher never crashes on any `(IgnoreMatcher, RelativePath)` pair
  producible through the public API.
- Malformed input always produces either a `CompileError` or a defined
  `MatchResult` — never a runtime exception, never a hang.

---

## Complexity Targets

```text
Compile: O(pattern count)
Match:   O(path length)
Memory:  O(pattern count)
```

These are algorithmic targets for the *compiled* matcher's steady-state
behavior, not promises about compile-time constant factors. See
[`docs/architecture.md`](docs/architecture.md) for the NFA→DFA tradeoff this
implies.

---

## Performance Targets

```text
Compile: 10,000 patterns < 200 ms
Match:   100,000 paths/sec minimum
```

Measured via `gauge`, on reference hardware documented in
[`docs/compatibility.md`](docs/compatibility.md). Not measured, not enforced, and
not optimized for until Phase 14 — see Guiding Principle above.

---

## Security Policy

Tracked in `SECURITY.md` at the repository root (created in Phase 0).

Guarantees:

- No catastrophic backtracking — the compiled-automaton approach (Phases 8–9)
  makes this true by construction rather than by careful regex authoring.
- No regex engine dependency.
- Bounded memory growth as a function of pattern count and path length only.
- Resistance to malicious input: no pattern or path, however constructed,
  should produce non-linear time or unbounded memory behavior.

---

## Release Checklist

Every release requires, in this order:

```text
✓ build passes
✓ conformance passes
✓ parity passes (against every Git version in docs/compatibility.md)
✓ fuzz tests pass (minimum corpus-run duration documented in docs/ROADMAP.md)
✓ benchmarks pass (performance targets above)
✓ docs updated
✓ changelog updated
✓ API review completed (see docs/api-philosophy.md)
✓ README.md status table updated
```

---

## Explicitly Rejected

Do not use:

- GADTs.
- Liquid Haskell.
- Template Haskell (v1 — may be reconsidered for v2's quasiquoter, see
  Future Work).
- Runtime exceptions for validation — validation failures are values
  (`Either CompileError a`), never thrown.

---

## Future Work

### v2

```haskell
[gitignore|
*.log
target/
|]
```

A quasiquoter for compile-time-checked ignore patterns. Requires revisiting
the Template Haskell rejection above, scoped narrowly to this one feature.

A `MatchTrace` type (see [`MatchResult`](#matchresult) above) exposing the
distinction between "no pattern matched" and "explicitly re-included," as an
additive, non-breaking change.

### v3

- SIMD matching.
- Incremental updates (recompiling a matcher cheaply when one pattern file
  changes, without recompiling the whole automaton).
- Streaming evaluation for very large path sets.

---

## Changelog from v1.0

- Renamed Phase 4 from "Normalization" to "Path Normalization" and clarified
  it operates on `RelativePath`, not on patterns — resolves an ordering
  ambiguity with Phases 5–7.
- Added the directory-exclusion exception to Formal Matching Rules
  (negation cannot re-include a path under an already-excluded ancestor
  directory) — this was missing entirely in v1.0 and is required for goal #1
  (exact Git compatibility).
- Added an explicit corresponding property test and corpus-coverage
  requirement for the directory-exclusion exception.
- Documented `MatchResult`'s known two-state limitation explicitly rather
  than leaving it unaddressed, and tracked a non-breaking future extension.
- Added file-precedence detail (`.git/info/exclude`, global excludes,
  per-directory `.gitignore`) to Formal Matching Rules, previously
  unaddressed.
- Extracted all phase deliverables into checkable exit criteria in the new
  [`docs/ROADMAP.md`](docs/ROADMAP.md), replacing the v1.0 pattern of one-line
  phase descriptions with no definition of "done."
- Added this README-and-SPEC split per project request; `SPEC.md` is now the
  charter, `README.md` is the entry point and live status tracker.
