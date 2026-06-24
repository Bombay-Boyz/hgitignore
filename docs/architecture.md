# Architecture

## Package Dependency Graph

```text
hgitignore-testkit
        ▲
        │ (test-only dependency)
        │
hgitignore-core  ◄──────────── hgitignore
   (pure)                      (IO, filesystem)
```

- `hgitignore-core` has no dependency on `hgitignore` or `hgitignore-testkit`
  in its production code. It may depend on `hgitignore-testkit` in its
  `test/` stanza only.
- `hgitignore` depends on `hgitignore-core` and, in its `test/` stanza, on
  `hgitignore-testkit`.
- `hgitignore-testkit` depends on neither of the other two packages'
  production code in a way that would create a cycle — it provides
  generators and helpers that *consume* the public types from
  `hgitignore-core` (e.g. generating arbitrary `AST` values for property
  tests), so it does depend on `hgitignore-core`, but never the reverse.
- `fuzz/` and `benchmark/` depend on `hgitignore-core`, `hgitignore`, and
  `hgitignore-testkit` as needed; nothing depends on `fuzz/` or
  `benchmark/`.

## Data Flow (Compile Path)

```text
Text
  │  (Phase 5: Lexer)
  ▼
[Token]
  │  (Phase 6: Parser, Megaparsec)
  ▼
AST
  │  (Phase 7: Validation)
  ▼
ValidatedAST  (or equivalent — see invariants.md)
  │  (Phase 8: NFA Compiler)
  ▼
NFA
  │  (Phase 9: DFA Compiler)
  ▼
DFA
  │  (wrapped as)
  ▼
IgnoreMatcher   (opaque, returned by Phase 10's `compile`)
```

## Data Flow (Match Path)

```text
FilePath / Text
  │  (RelativePath smart constructor, Phase 3)
  ▼
RelativePath
  │  (Phase 4: normalize)
  ▼
RelativePath (normalized)
  │  (Phase 10: matches, walking the DFA)
  ▼
MatchResult
```

## Why NFA Then DFA, Rather Than DFA Directly

Compiling patterns directly to a DFA via a different construction is
possible in principle, but the NFA-as-intermediate-step approach is chosen
because:

1. It mirrors the natural structure of pattern alternation/negation
   (each pattern is naturally an NFA fragment; combining patterns is
   naturally NFA union).
2. It gives a natural, independently-checkable place to insert the
   "reference NFA simulator" cross-check described in
   [Phase 8's exit criteria](ROADMAP.md#phase-8--nfa-compiler) — having a
   slow-but-obviously-correct simulator at the NFA stage, separate from the
   DFA's subset-construction optimization, is what makes
   [Phase 9's correctness property test](ROADMAP.md#phase-9--dfa-compiler)
   possible without already trusting the DFA compiler.
3. It isolates the one genuinely complexity-heavy piece of the
   compile pipeline (subset construction) to a single, dedicated phase,
   keeping Phase 8 simpler.

The cost is a compile-time intermediate representation that's discarded
after Phase 9 — this is an accepted tradeoff per the Guiding Principle
(correctness and clarity of construction over compile-time performance,
until Phase 14).

## Why the Directory-Exclusion Exception Forces Top-Down Matching

As described in [`semantics.md` §3.1](semantics.md#31-the-directory-exclusion-exception),
a path's ignored status depends recursively on its ancestors' ignored
status. This means the matcher cannot simply walk the DFA over the leaf
path's string representation in isolation — or rather, it can, but only if
the DFA's construction itself encodes the directory-exclusion semantics
(e.g. by having compiled-in "dead" states once a directory-exclusion is
hit, that no negation pattern can transition out of). The exact mechanism
(automaton-encoded vs. an explicit path-segment-by-segment walk above the
automaton) is an implementation decision made during
[Phase 8](ROADMAP.md#phase-8--nfa-compiler) and must be documented here,
concretely, once chosen — this section is intentionally left as "the
constraint" rather than "the mechanism" until that phase locks it in.

## Formatter and Linter Choice

**Formatter: `fourmolu`**

Chosen over `ormolu` because it supports per-project configuration via
`fourmolu.yaml` at the repository root, allowing consistent style
enforcement without per-developer tool flags. Configuration is in
`fourmolu.yaml`. CI runs `fourmolu --mode check` and fails on any
unformatted file.

To format all files locally:
```bash
fourmolu --mode inplace $(find . -name "*.hs" -not -path "*/.stack-work/*")
```

**Linter: `hlint`**

Configuration is in `.hlint.yaml` at the repository root. The baseline is
zero warnings. CI runs `hlint` on all source files and fails on any warning.
The `.hlint.yaml` also encodes the forbidden-function list from `SPEC.md`
as explicit `warn` hints for an extra layer of enforcement beyond GHC's
`-Werror`.

## Reference Hardware for Benchmarks

Decided and recorded during [Phase 14](ROADMAP.md#phase-14--benchmarking).
Placeholder until that phase fills in the actual machine spec the
[performance targets](../SPEC.md#performance-targets) are measured against.
