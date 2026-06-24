# Roadmap

Sixteen phases, strict order. A phase is **not done** until every checkbox
in its section is checked, its own tests pass in CI, and any docs it
references are written (not stubbed).

Conventions used throughout this document:

- `[ ]` — not done. `[x]` — done. Update these in PRs; this file is meant to
  be edited as a living tracker, not a write-once plan.
- **Goal** — why this phase exists, one sentence.
- **Depends on** — phases that must already be fully checked off.
- **Deliverables** — what gets built.
- **Exit criteria** — the checkable bar. If it's not on this list, it is not
  required to ship the phase, even if it seems related.
- **Non-goals** — explicitly out of scope for this phase, to prevent scope
  creep into later phases' work.

When a phase is completed, also tick its row in the
[README status table](../README.md#project-status).

---

## Phase 0 — Infrastructure

**Goal:** a buildable, lintable, CI-checked empty skeleton.

**Depends on:** nothing.

**Deliverables:**

- `stack.yaml`, `package.yaml` for all three packages
  (`hgitignore-core`, `hgitignore`, `hgitignore-testkit`).
- CI pipeline (build + test on every push).
- Formatter configuration (e.g. `ormolu` or `fourmolu` — pick one, document
  the choice in `docs/architecture.md`).
- Linter configuration (`hlint`).
- `benchmark/` and `fuzz/` directories scaffolded but empty (a placeholder
  `.gitkeep` or trivial no-op target only — no real benchmarks or fuzz
  targets yet; see Phases 13–14).
- `SECURITY.md` at repository root.
- Compiler flags applied to every package:

```yaml
-Wall
-Wcompat
-Wincomplete-patterns
-Werror
```

**Exit criteria:**

- [ ] `stack build` succeeds from a clean clone with zero warnings.
- [ ] CI runs on every push/PR and is green.
- [ ] CI fails the build if `-Wall -Wcompat -Wincomplete-patterns -Werror`
      is removed from any package's GHC options (i.e. there's a test or CI
      step that would catch a regression here, not just a one-time check).
- [ ] Formatter check runs in CI and fails on unformatted code.
- [ ] Linter runs in CI; baseline is zero `hlint` warnings.
- [ ] `SECURITY.md` exists with at minimum the guarantees listed in
      [`SPEC.md`](../SPEC.md#security-policy).
- [ ] `benchmark/` and `fuzz/` exist as directories tracked in git, with a
      one-line `README.md` each stating "intentionally empty until Phase
      14 / Phase 13."

**Non-goals:** no real code in any `src/` directory yet. No fixture format.
No actual benchmarks or fuzz targets.

---

## Phase 1 — Formal Semantics

**Goal:** write down, precisely, what "correct" means — before any code that
could embed an incorrect assumption exists.

**Depends on:** Phase 0.

**Deliverables:**

- `docs/semantics.md`, covering:
  - Pattern syntax grammar (informal or formal — at minimum unambiguous
    prose, EBNF preferred).
  - Anchoring rules (leading `/`, trailing `/`).
  - Wildcard semantics (`*`, `?`, `[...]`, `**` in leading/trailing/middle
    position).
  - Negation semantics, **including the directory-exclusion exception**
    (a negated pattern cannot re-include a path whose ancestor directory is
    already excluded) with at least three worked examples.
  - File precedence: `.git/info/exclude`, global excludes file, nested
    `.gitignore` files, and how deeper-directory files interact with
    shallower ones.
  - Path normalization rules (separator handling, trailing slash meaning,
    `.`/`..` handling).
  - Escape sequence handling (`\#`, `\!`, `\ ` trailing space, etc.).
  - Comment and blank-line handling.
  - A list of known edge cases with their resolved (documented, intentional)
    behavior — not an open question list.

**Exit criteria:**

- [ ] `docs/semantics.md` exists and covers every bullet above.
- [ ] The directory-exclusion exception has at least 3 worked examples with
      explicit pattern set, path, and expected result.
- [ ] At least one person other than the author has reviewed
      `docs/semantics.md` against real `git check-ignore` behavior for the
      worked examples (manually verified, not yet automated — that's
      Phase 12) and signed off in the PR.
- [ ] No parser, lexer, or matcher code exists anywhere in the repository
      yet (enforced by review, not tooling — this is a process gate).

**Non-goals:** no implementation. No fixture corpus yet (that's Phase 2's
format definition and Phase 11's bulk corpus). This phase is prose and
worked examples only.

---

## Phase 2 — Conformance Specification

**Goal:** define how "matches real Git" will be measured, before building
anything that needs measuring.

**Depends on:** Phase 1.

**Deliverables:**

- `docs/conformance.md`, covering:
  - Reference implementation: `git check-ignore`, with exact invocation
    pattern (flags used, working-directory assumptions, exit code meaning).
  - Supported Git version matrix (cross-reference
    [`docs/compatibility.md`](compatibility.md)).
  - Parity methodology: how a fixture's "expected" field is derived (must be
    "run real Git and record the result," never "what we believe Git does").
  - Fixture file format (the YAML shape used throughout `test-corpus/`).
  - How fixture disagreements are triaged (semantics.md is wrong vs.
    implementation is wrong vs. fixture itself is wrong).

**Exit criteria:**

- [ ] `docs/conformance.md` exists and covers every bullet above.
- [ ] The fixture format is finalized and matches the example in
      [`SPEC.md`](../SPEC.md#conformance-corpus) (or `SPEC.md` is updated to
      match — the two must not drift).
- [ ] At least 10 example fixtures exist in `test-corpus/examples/` (a
      small bootstrap set, distinct from the bulk corpus built in Phase 11)
      demonstrating the format covers every category listed in
      [`SPEC.md`](../SPEC.md#conformance-corpus).
- [ ] Each bootstrap fixture's `expected` value has been verified against a
      real `git check-ignore` run, with the command and output pasted into
      the PR description or a linked CI log.

**Non-goals:** not the bulk corpus (Phase 11). Not the automated parity
runner (Phase 12) — this phase defines the methodology the runner will
later implement.

---

## Phase 3 — Domain Types

**Goal:** establish the type vocabulary everything else is built on.

**Depends on:** Phase 2.

**Deliverables:**

- `Types.hs` — `RelativePath`, `MatchResult`, `IgnoreMatcher` (opaque at
  this stage — internals arrive in Phases 8–10), and any other core
  newtypes.
- `AST.hs` — the pattern AST shape (no parser yet, just the data type).
- `Error.hs` — `CompileError`, `ErrorCode` (`HG001`–`HG004` as a starting
  set; more added as needed per [`SPEC.md`](../SPEC.md#stable-error-codes)
  rules).

**Exit criteria:**

- [ ] `Types.hs`, `AST.hs`, `Error.hs` compile with zero warnings under the
      Phase 0 compiler flags.
- [ ] `RelativePath`'s only constructor is a smart constructor returning
      `Maybe RelativePath` or `Either SomeError RelativePath` — no
      data-constructor export.
- [ ] `MatchResult` is exactly the two-constructor type from
      [`SPEC.md`](../SPEC.md#matchresult) — no boolean stand-in anywhere in
      the codebase that should be a `MatchResult`.
- [ ] Every type has a Haddock comment explaining its invariants (not just
      its shape).
- [ ] No forbidden function (per
      [`SPEC.md`](../SPEC.md#type-safety-requirements)) appears anywhere in
      these three modules.
- [ ] Unit tests exist for the `RelativePath` smart constructor covering:
      empty string (rejected), absolute path (rejected), valid relative path
      (accepted), path with redundant separators (accepted-and-normalized
      or rejected — pick one, document which, test it).

**Non-goals:** no parser. No matcher. No normalization logic beyond what the
smart constructor needs for its own validity checks.

---

## Phase 4 — Path Normalization

**Goal:** a pure, idempotent function that normalizes `RelativePath` values.

**Depends on:** Phase 3.

**Deliverables:**

- `Normalize.hs` implementing path normalization as specified in
  `docs/semantics.md`'s path-normalization section.

**Exit criteria:**

- [ ] `normalize` is implemented and exported from `Normalize.hs`.
- [ ] Property test `normalize . normalize == normalize` passes under
      `hedgehog` with the default number of test cases (document the count
      if non-default).
- [ ] Property test confirms `normalize` never produces a value that fails
      `RelativePath`'s own smart-constructor invariants.
- [ ] At least 5 hand-written unit tests cover: redundant separators,
      trailing separator preservation/removal (per spec), `.`/`..` segment
      handling, and the empty-after-normalization edge case.
- [ ] Module has zero warnings, no forbidden functions.

**Non-goals:** this phase does not normalize *patterns* — pattern-level
normalization (escape handling, trailing-space trimming in pattern text) is
Phase 6/7's job, not this phase's. See the note in
[`SPEC.md`](../SPEC.md#development-sequence) on why this split exists.

---

## Phase 5 — Lexer

**Goal:** tokenize raw pattern-file text into a token stream, with no
semantic interpretation yet.

**Depends on:** Phase 4 (for shared `Error.hs` types; no functional
dependency on normalization itself).

**Deliverables:**

- A tokenizer (hand-written or Megaparsec-based — Megaparsec is reserved for
  Phase 6's grammar; the lexer may be a thin wrapper or a separate simple
  pass, document the choice).
- Token type covering: literal segments, `*`, `**`, `?`, `[`/`]` character
  classes, `!` negation marker, `#` comment marker, escape sequences,
  line/blank-line boundaries.

**Exit criteria:**

- [ ] Lexer produces a defined token stream for every line of every
      bootstrap fixture from Phase 2.
- [ ] Lexer never throws — malformed input produces a token representing
      "lex error" or is deferred to Phase 7 validation, not a crash. Decide
      which and document it.
- [ ] Property test: lexing is total over arbitrary `Text` input (fed via
      `hedgehog` generators) — no input causes a runtime exception.
- [ ] Unit tests for every token category listed in Deliverables, including
      at least one escape-sequence case per documented escape in
      `docs/semantics.md`.

**Non-goals:** no AST construction. No validation of whether a token
sequence forms a *valid* pattern — that's Phase 6 (grammar) and Phase 7
(validation).

---

## Phase 6 — Parser

**Goal:** turn the token stream into the `AST` from Phase 3, using
Megaparsec.

**Depends on:** Phase 5.

**Deliverables:**

- Megaparsec-based parser producing `AST` values from token streams (or
  directly from `Text`, if the lexer is folded in as a Megaparsec
  tokenizer — document the actual boundary chosen).
- Pattern-level normalization (trailing-space trimming, escape resolution)
  applied here, per the Phase 4 note above.

**Exit criteria:**

- [ ] Parser produces correct `AST` for every bootstrap fixture's pattern
      set from Phase 2.
- [ ] Property test `parse . render == id` passes, where `render` is a
      (test-only, or public — decide and document) function turning `AST`
      back into pattern text.
- [ ] Parser returns `Either CompileError AST` — never partial, never an
      exception — for arbitrary `Text` input (covered by a totality property
      test, same style as Phase 5's).
- [ ] Comment lines and blank lines are correctly excluded from the
      resulting `AST`.
- [ ] Every error path uses one of the `HG0xx` codes from
      [`SPEC.md`](../SPEC.md#stable-error-codes); new codes added here are
      appended, never inserted out of sequence.

**Non-goals:** no validation of semantic validity beyond what's needed to
produce a well-formed `AST` (e.g. parser accepts syntactically valid but
semantically questionable patterns; Phase 7 is where those get rejected or
accepted per spec).

---

## Phase 7 — Validation

**Goal:** reject `AST` values that are syntactically well-formed but
semantically invalid, so the compiler (Phase 8) can assume a valid `AST`.

**Depends on:** Phase 6.

**Deliverables:**

- A validation pass over `AST` producing `Either CompileError AST`
  (validated).

**Exit criteria:**

- [ ] Every "known edge case" listed in `docs/semantics.md` (Phase 1) that
      is classified as *invalid* has a corresponding rejection in this pass,
      with a test proving it's rejected with the correct error code.
- [ ] Every edge case classified as *valid-but-unusual* is proven to pass
      through validation unchanged, with a test.
- [ ] `docs/invariants.md` updated to state: "the compiler (Phase 8) may
      assume its `AST` input has passed this validation pass" — and this
      phase's tests are what make that assumption safe.
- [ ] No forbidden function used; all rejections are values, never thrown
      exceptions (per [`SPEC.md`](../SPEC.md#explicitly-rejected)).

**Non-goals:** no compilation to automata yet.

---

## Phase 8 — NFA Compiler

**Goal:** compile a validated `AST` into an NFA representation.

**Depends on:** Phase 7.

**Deliverables:**

- `AST -> NFA` compiler.
- NFA type definition (internal module).

**Exit criteria:**

- [ ] Compiler is total over any validated `AST` (i.e. any `AST` that passed
      Phase 7) — proven by a property test, not just example tests.
- [ ] For every bootstrap fixture's pattern set, the resulting NFA correctly
      accepts/rejects the fixture's `path` when walked by a reference NFA
      simulator (a simple, possibly inefficient, "obviously correct"
      simulator — this is the cross-check before the DFA exists).
- [ ] `docs/invariants.md` documents the NFA's structural invariants (no
      dangling transitions, etc.) and a property test enforces each one.

**Non-goals:** no DFA construction yet. No performance concerns — the
reference NFA simulator is allowed to be slow.

---

## Phase 9 — DFA Compiler

**Goal:** compile the NFA into a deterministic automaton suitable for fast,
allocation-light matching.

**Depends on:** Phase 8.

**Exit criteria:**

- [ ] `NFA -> DFA` compiler is total over any NFA producible by Phase 8.
- [ ] Property test: for every (NFA, input) pair generated by `hedgehog`,
      the DFA's accept/reject result matches the reference NFA simulator's
      result from Phase 8 — this is the determinism/correctness bridge
      between the two phases.
- [ ] `docs/invariants.md`'s "DFA deterministic" and "no invalid
      transitions" invariants each have a dedicated property test.
- [ ] Subset-construction blowup is measured (not yet optimized against —
      that's Phase 14) and recorded in `docs/architecture.md` for at least
      the bootstrap fixture pattern sets, so later regressions are visible.

**Non-goals:** no benchmarking against the Phase-14 performance targets yet
— recording a number is not the same as optimizing for one.

---

## Phase 10 — Matcher

**Goal:** the public-facing `matches` / `matchesMany` functions, operating
purely on compiled automata.

**Depends on:** Phase 9.

**Deliverables:**

- `matches :: IgnoreMatcher -> RelativePath -> MatchResult`.
- `matchesMany :: IgnoreMatcher -> Vector RelativePath -> Vector MatchResult`.
- `compile :: Text -> Either CompileError IgnoreMatcher`, wiring together
  Phases 5–9 into the single public entry point.

**Exit criteria:**

- [ ] `compile` and `matches` together reproduce the correct result for
      every bootstrap fixture from Phase 2 — this is the first point where
      "fixture in, result out" works end-to-end.
- [ ] **The directory-exclusion exception is explicitly tested here** with
      the worked examples from `docs/semantics.md` (Phase 1) — this is
      gated as a named exit criterion, not folded into "fixtures pass,"
      because it is the single highest-risk correctness gap historically
      (see [`SPEC.md`](../SPEC.md#formal-matching-rules) item 5).
- [ ] `matchesMany v == fmap (matches m) v` as a property test (consistency
      between the batch and single-item API, not just a performance
      shortcut that happens to agree on examples).
- [ ] `match p x == match p x` determinism property test passes.
- [ ] Public API signatures match exactly what's in
      [`SPEC.md`](../SPEC.md#public-api) — any deviation requires updating
      `SPEC.md` in the same PR, not after.
- [ ] `hgitignore` package's `compileFile :: FilePath -> IO (Either
      CompileError IgnoreMatcher)` is implemented, with tests covering:
      missing file, unreadable file, empty file, valid file.

**Non-goals:** no bulk corpus yet (Phase 11). No automated real-Git parity
yet (Phase 12) — this phase proves internal consistency and the bootstrap
fixtures only.

---

## Phase 11 — Conformance Corpus

**Goal:** build the large-scale fixture corpus that becomes the permanent
regression bar.

**Depends on:** Phase 10 (need a working matcher to bootstrap corpus
authoring against, even though the corpus's `expected` values come from
real Git, not from `hgitignore`).

**Deliverables:**

- `test-corpus/` populated with fixtures numbering in the thousands, per
  [`SPEC.md`](../SPEC.md#conformance-corpus)'s required-coverage list.

**Exit criteria:**

- [ ] At least 2,000 fixtures exist in `test-corpus/` (the spec's "thousands"
      — set a concrete floor so this is actually checkable; raise this
      number in this file if the project later wants more).
- [ ] Every category in [`SPEC.md`](../SPEC.md#conformance-corpus)'s
      required-coverage bullet list has at least 50 fixtures.
- [ ] The directory-exclusion exception has at least 50 dedicated fixtures
      covering varied nesting depths and pattern orderings.
- [ ] Every fixture's `expected` field is generated by running real Git
      (scripted, not hand-typed) — `docs/conformance.md`'s methodology
      section describes exactly how, and the generation script lives in
      `hgitignore-testkit`.
- [ ] Corpus generation is reproducible: running the generation script twice
      against the same Git version produces byte-identical fixture files.
- [ ] `hgitignore` is run against the full corpus as part of this phase
      (even though formal "parity" automation is Phase 12) and any failures
      are triaged and fixed before this phase is marked done — an
      unreviewed pile of red fixtures does not count as a completed corpus.

**Non-goals:** this phase is about building and validating the corpus
itself, not about building the repeatable CI pipeline around it (Phase 12).

---

## Phase 12 — Git Parity

**Goal:** an automated, CI-integrated pipeline that runs the full corpus
against real Git on every change and fails on any divergence.

**Depends on:** Phase 11.

**Deliverables:**

- `stack run parity` command, per [`SPEC.md`](../SPEC.md#git-parity-pipeline).

**Exit criteria:**

- [ ] `stack run parity` runs the entire `test-corpus/` against the minimum
      supported Git version (per `docs/compatibility.md`) and reports
      pass/fail with a readable diff on failure (pattern set, path,
      expected, actual).
- [ ] `stack run parity` is wired into CI and runs on every PR.
- [ ] CI is green with zero fixture divergences.
- [ ] Parity is also run against the *latest* Git version, separately from
      the minimum-version run, and both are green (catches Git itself
      changing behavior across versions).
- [ ] A regression test exists proving the pipeline actually fails when a
      deliberately-broken matcher is substituted (a meta-test: the parity
      tool's own correctness is checked once, not assumed).

**Non-goals:** this phase does not add new fixtures — it automates running
the ones Phase 11 built.

---

## Phase 13 — Fuzzing

**Goal:** confirm the parser and matcher never crash, on any input.

**Depends on:** Phase 12.

**Deliverables:**

- Fuzz targets in `fuzz/` for the parser and for the matcher (given
  arbitrary compiled matchers and arbitrary path strings).

**Exit criteria:**

- [ ] Parser fuzz target runs for a documented minimum duration (state the
      number here once decided, e.g. "4 CPU-hours") with zero crashes,
      zero hangs, zero unbounded-memory-growth findings.
- [ ] Matcher fuzz target, same bar.
- [ ] Any crash found during development of this phase has a corresponding
      regression fixture added to `test-corpus/` (or a dedicated regression
      suite) before this phase is marked done — fuzzing findings must
      become permanent tests, not just be fixed and forgotten.
- [ ] Fuzzing is wired into CI on a schedule (not necessarily every PR, but
      at minimum nightly) and a failed fuzz run blocks release per the
      [Release Checklist](../SPEC.md#release-checklist).

**Non-goals:** not a substitute for the property tests in earlier phases —
fuzzing here is specifically about crash/hang/memory safety, not output
correctness (that's what the corpus and parity pipeline check).

---

## Phase 14 — Benchmarking

**Goal:** measure performance against the stated targets, and optimize only
now that correctness is established.

**Depends on:** Phase 13. (Per the Guiding Principle, no benchmark-driven
optimization PR is accepted before this phase, i.e. before Phase 12 is
green — Phase 13 is also required complete since optimization work
shouldn't be layered on top of unverified crash-safety.)

**Deliverables:**

- `gauge`-based benchmarks in `benchmark/` for `compile` and `matches` /
  `matchesMany`.

**Exit criteria:**

- [ ] Benchmark suite measures: compile time for 10,000 patterns; match
      throughput in paths/sec.
- [ ] Reference hardware is documented in `docs/compatibility.md`.
- [ ] Compile target met: 10,000 patterns compile in under 200ms on
      reference hardware, verified by the benchmark suite, not by manual
      timing.
- [ ] Match target met: 100,000 paths/sec minimum sustained throughput,
      verified by the benchmark suite.
- [ ] If either target is not met, this phase is not done — optimization
      work happens here, and **every optimization PR in this phase must
      keep Phase 12's parity suite green** (no exceptions for "obviously
      safe" optimizations).
- [ ] Benchmarks run in CI (a non-blocking informational run is acceptable
      if full benchmark runs are too slow for every PR — document the
      chosen CI cadence here once decided).

**Non-goals:** no new functionality. Any optimization that would require
relaxing a Phase 1–12 correctness guarantee is rejected, full stop, even if
it would meet performance targets.

---

## Phase 15 — Release

**Goal:** ship v1.0.0.

**Depends on:** Phase 14.

**Exit criteria:** every item in
[`SPEC.md`](../SPEC.md#release-checklist), specifically:

- [ ] Build passes.
- [ ] Conformance passes (Phase 11's corpus, full run).
- [ ] Parity passes (Phase 12, both Git version targets).
- [ ] Fuzz tests pass (Phase 13's documented duration, latest scheduled run).
- [ ] Benchmarks pass (Phase 14's targets).
- [ ] All `docs/*.md` files reviewed for staleness against the actual
      shipped implementation (not just existence — re-read them).
- [ ] `CHANGELOG.md` created/updated with the full history from Phase 0.
- [ ] API review completed and signed off per
      [`docs/api-philosophy.md`](api-philosophy.md) — confirm the public
      module list in [`SPEC.md`](../SPEC.md#stability-policy) matches what's
      actually exported.
- [ ] `README.md` status table shows all 16 phases as Done.
- [ ] License finalized (currently TBD in `README.md`).
- [ ] Version tagged `v1.0.0`, published per the chosen distribution
      channel (Hackage, documented here once decided).

**Non-goals:** none — this is the finish line for v1. Future Work items
(quasiquoter, `MatchTrace`, SIMD, incremental updates, streaming) are
explicitly out of scope and tracked in
[`SPEC.md`](../SPEC.md#future-work) instead.
