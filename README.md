# hgitignore

The canonical Haskell implementation of Git ignore semantics.

`hgitignore` parses `.gitignore`-style pattern files and tells you, for any
given path, whether Git would ignore it. It is built to be **byte-for-byte
behaviorally compatible** with `git check-ignore`, type-safe by construction,
and small enough that a single engineer can hold the whole codebase in their
head.

> **Status: Pre-implementation.** This repository currently contains the
> specification only. No code has been written yet. See
> [Project Status](#project-status) below for what that means in practice.

---

## Why this exists

Most `.gitignore` parsers in the wild are "close enough." They get the common
cases right and diverge on edge cases: trailing slashes, escaped wildcards,
double-star semantics, negation interacting with excluded directories,
anchored vs. unanchored patterns. `hgitignore` exists to be the implementation
that doesn't diverge — verified continuously against real `git check-ignore`
output across a large fixture corpus.

If you need "probably matches what Git does," many libraries already do that.
If you need "matches what Git does, provably, on every fixture we could
write," that's this project's reason to exist.

---

## Design principles

1. **Correctness > Simplicity > Performance.** In that strict order.
   Performance work is not permitted until conformance is complete (see
   [Phase 14](docs/ROADMAP.md#phase-14--benchmarking)).
2. **The corpus is authoritative, not the implementation.** If `hgitignore`
   and `git check-ignore` disagree on a fixture, `hgitignore` is wrong by
   definition, full stop — see [`docs/conformance.md`](docs/conformance.md).
3. **Total functions only.** No partial functions, no runtime exceptions for
   control flow, no `unsafePerformIO`. See
   [`docs/invariants.md`](docs/invariants.md).
4. **A small, stable public API.** Additions require demonstrated need, not
   speculative convenience. See [`docs/api-philosophy.md`](docs/api-philosophy.md).

---

## Repository structure

```text
hgitignore/
│
├── README.md                  ← you are here
├── SPEC.md                    ← the full implementation charter
│
├── stack.yaml                 ← (Phase 0)
├── package.yaml                ← (Phase 0)
│
├── hgitignore-core/            ← pure: AST, parser, compiler, matcher
│   ├── src/
│   └── test/
│
├── hgitignore/                 ← user-facing API, filesystem helpers
│   ├── src/
│   └── test/
│
├── hgitignore-testkit/         ← generators, parity helpers, fixtures
│   ├── src/
│   └── test/
│
├── benchmark/                  ← gauge benchmarks (Phase 14 only)
├── fuzz/                       ← parser/matcher fuzzing (Phase 13)
├── test-corpus/                ← the authoritative fixture corpus (Phase 11)
│
└── docs/
    ├── semantics.md            ← formal pattern-matching semantics
    ├── conformance.md          ← parity methodology against real Git
    ├── compatibility.md        ← supported Git/OS versions, deprecation policy
    ├── invariants.md           ← properties that must always hold
    ├── api-philosophy.md       ← what goes in the public API, and why
    ├── architecture.md         ← package boundaries and data flow
    ├── stability.md            ← semver policy, stable module list
    └── ROADMAP.md               ← phase-by-phase plan with checkable milestones
```

---

## Quickstart (once Phase 6 lands)

```haskell
import HGitIgnore.Compile (compile)
import HGitIgnore.Match   (matches)

main :: IO ()
main = do
  let patterns = "*.log\n!important.log\n"
  case compile patterns of
    Left err -> print err
    Right matcher -> do
      print (matches matcher "debug.log")      -- Ignored
      print (matches matcher "important.log")  -- Included
```

This snippet is aspirational until [Phase 6](docs/ROADMAP.md#phase-6--parser)
is checked off. It documents the *target* public API shape — see
[`SPEC.md`](SPEC.md#public-api) for the authoritative signatures.

---

## Project status

Development proceeds in strict phase order — see
[`docs/ROADMAP.md`](docs/ROADMAP.md) for the full breakdown. Each phase has
explicit, checkable exit criteria; a phase is not "done" until every box is
checked, including its tests and documentation.

| Phase | Name | Status |
|---|---|---|
| 0  | Infrastructure | ☐ Not started |
| 1  | Formal Semantics | ☐ Not started |
| 2  | Conformance Specification | ☐ Not started |
| 3  | Domain Types | ☐ Not started |
| 4  | Path Normalization | ☐ Not started |
| 5  | Lexer | ☐ Not started |
| 6  | Parser | ☐ Not started |
| 7  | Validation | ☐ Not started |
| 8  | NFA Compiler | ☐ Not started |
| 9  | DFA Compiler | ☐ Not started |
| 10 | Matcher | ☐ Not started |
| 11 | Conformance Corpus | ☐ Not started |
| 12 | Git Parity | ☐ Not started |
| 13 | Fuzzing | ☐ Not started |
| 14 | Benchmarking | ☐ Not started |
| 15 | Release | ☐ Not started |

Update this table as part of every PR that closes out a phase. A phase's row
only moves to "Done" when every checkbox in its `docs/ROADMAP.md` section is
checked **and** CI is green on `main`.

---

## Contributing

Before writing any code, read, in order:

1. [`SPEC.md`](SPEC.md) — the charter: mission, architecture, non-negotiables.
2. [`docs/ROADMAP.md`](docs/ROADMAP.md) — find the current open phase.
3. The specific `docs/*.md` file that phase references.

Do not start a phase out of order. Do not start implementation work (Phase 3
onward) before Phase 1 and Phase 2 are checked off — the formal semantics and
conformance methodology are the contract everything else is built against.

Pull requests that skip ahead of the current phase, that add public API
surface without a documented need (see
[`docs/api-philosophy.md`](docs/api-philosophy.md)), or that introduce any of
the [forbidden functions](SPEC.md#type-safety-requirements) will be rejected
in review regardless of test coverage.

---

## License

TBD — to be finalized before Phase 15 (Release). Do not assume a license
until this section is updated.
