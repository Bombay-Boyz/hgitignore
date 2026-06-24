# Internal Invariants

Properties that must hold at all times, across all phases. Each invariant
below should have a corresponding property test by the time the phase that
introduces it is complete — see the cross-references into
[`ROADMAP.md`](ROADMAP.md).

| Invariant | Introduced in | Enforced by |
|---|---|---|
| `RelativePath` values are always normalized — no code path produces an unnormalized value with this type. | Phase 3 (type), Phase 4 (normalize fn) | Smart constructor + `normalize . normalize == normalize` property test. |
| `IgnoreMatcher` is immutable once compiled — no API exists to mutate one in place. | Phase 10 | Type design (no exposed mutation functions); enforced by code review, since Haskell purity makes accidental mutation structurally difficult but not impossible via, e.g., an `IORef` smuggled into an opaque type. |
| The compiled DFA is deterministic — no `(state, input)` pair maps to more than one next state. | Phase 9 | Property test comparing DFA walk results against the Phase 8 reference NFA simulator across generated inputs. |
| No DFA transition table entry references a nonexistent state. | Phase 9 | Property test validating the transition table's codomain against its own state set after every compilation. |
| The `AST` is always validated (Phase 7) before reaching the compiler (Phase 8); the compiler assumes a valid `AST` and does not re-validate. | Phase 7/8 boundary | Type-level separation if practical (e.g. a `ValidatedAST` newtype distinct from `AST`) — decide and document the chosen approach during Phase 7 implementation; if a type-level distinction isn't used, this must instead be enforced by the module boundary (compiler module only imports the validation module's output type) plus an explicit comment at the boundary. |
| Negation cannot re-include a path under an already-excluded ancestor directory. | Phase 1 (spec), Phase 10 (matcher), Phase 11 (corpus) | Dedicated property test (see [`SPEC.md`](../SPEC.md#required-property-tests)) plus ≥50 dedicated corpus fixtures. |
| The parser never crashes on any `Text` input, valid pattern syntax or not. | Phase 6 | Totality property test + Phase 13 fuzzing. |
| The matcher never crashes on any `(IgnoreMatcher, RelativePath)` pair producible through the public API. | Phase 10 | Totality property test + Phase 13 fuzzing. |
| Compiling is a pure, deterministic function of its `Text` input — `compile t == compile t`. | Phase 10 | Property test, per [`SPEC.md`](../SPEC.md#required-property-tests). |
| No forbidden function (`head`, `tail`, `fromJust`, `error`, `undefined`, `unsafePerformIO`, `unsafeCoerce`, etc.) appears in production code. | Phase 0 onward | `hlint` rule configuration (Phase 0) flagging each forbidden name; CI fails the build on any match. |

## Test-Code Exception List

Production code (`hgitignore-core/src`, `hgitignore/src`) never uses the
forbidden functions listed in
[`SPEC.md`](../SPEC.md#type-safety-requirements), with no exceptions.

Test code (`test/`, `fuzz/`) may use a narrow set, **only** where the
alternative would meaningfully obscure what's being tested:

- `fromJust` / `error` in test setup code, when the precondition is
  guaranteed by the test's own construction (e.g. "this `Maybe` is `Just`
  because we just constructed it three lines above") — and even then,
  prefer pattern-matching with a descriptive failure message over a bare
  `fromJust`.
- `head`/`tail` in test assertions over lists that are known non-empty by
  construction within the same test.

Any use of this exception list should be reviewed skeptically — "it's just
a test" is not a blanket exemption, since flaky or unclear tests undermine
the entire conformance story this project depends on.
