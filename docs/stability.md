# Stability Policy

## Versioning

`hgitignore` follows [Semantic Versioning](https://semver.org/) strictly:

- **Major** (`X.0.0`): any breaking change to a public, semver-covered
  module listed below — signature changes, removed exports, changed
  behavior of an existing function for previously-defined inputs.
- **Minor** (`0.X.0`): additive, backward-compatible changes — new exports,
  new optional behavior, support for a new Git version added to
  [`compatibility.md`](compatibility.md).
- **Patch** (`0.0.X`): bug fixes that don't change any public signature,
  including conformance fixes (a fixed matcher that now correctly handles a
  case it previously got wrong is a patch release, even though the
  *behavior* changes — the public *type signatures* don't).

A conformance fix that changes behavior is, by definition, fixing a bug,
not introducing a new feature — see the discussion in
[`conformance.md`](conformance.md) on how disagreements between
implementation and fixtures get resolved. This is recorded here explicitly
because it's a case where "behavior changed" and "this is a patch release"
both being true can look contradictory without this note.

## Public, Semver-Covered Modules

```text
HGitIgnore.Compile
HGitIgnore.Match
HGitIgnore.Pattern
HGitIgnore.Types
HGitIgnore.Error
```

Every exported name from these five modules is covered by the guarantees
above. Internal types these modules might re-export are themselves
considered part of the public surface at that point — re-exporting
something does not exempt it.

## `HGitIgnore.Internal.*`

Everything under this namespace is explicitly **not** covered by semver. It
may change in any release, including patch releases, without a changelog
entry beyond noting internal refactoring occurred. Code outside the
`hgitignore`/`hgitignore-core` packages should not depend on
`HGitIgnore.Internal.*` — doing so is unsupported.

## Error Codes Are API

Per [`SPEC.md`](../SPEC.md#stable-error-codes): `ErrorCode` constructors
(`HG001`, `HG002`, ...) are covered by the same stability guarantee as the
five modules above, even though `HGitIgnore.Error` is one specific module in
that list — called out separately here because error codes have an extra
rule beyond normal semver: they are never renumbered or reused even across
major versions, since downstream tooling may match on the numeric code
itself (e.g. `HG003` always means "Invalid Path," forever, even if a future
major version restructures the `CompileError` type around it).

## What Triggers a Major Version Bump — Worked Examples

| Change | Version bump |
|---|---|
| Add a new function to `HGitIgnore.Match` | Minor |
| Change `matches`'s argument order | Major |
| Fix a bug where a directory-exclusion edge case was mishandled (see [`semantics.md` §3.1](semantics.md#31-the-directory-exclusion-exception)) | Patch |
| Add a new `ErrorCode` constructor for a newly-identified invalid input | Minor |
| Remove a deprecated export | Major |
| Add the `MatchTrace` type from [Future Work](../SPEC.md#future-work), purely additive | Minor |
| Change `IgnoreMatcher`'s internal representation without changing any exported signature | Patch (or no release at all, if bundled with something else) |
