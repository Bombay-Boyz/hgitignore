# Formal Semantics

This document is authoritative for what `hgitignore` must compute. It is
written and reviewed (Phase 1) before any parser, lexer, or matcher code
exists. If implementation and this document disagree, **this document is not
automatically right** — but the disagreement must be resolved by updating
this document deliberately (with the review process described in
[`ROADMAP.md`](ROADMAP.md#phase-1--formal-semantics)), never by silently
matching whatever the code happens to do.

The ultimate authority is real Git behavior (see
[`conformance.md`](conformance.md)). This document is our best current
written account of that behavior. Where this document and real Git disagree,
real Git wins, and this document gets corrected.

---

## 1. Pattern Syntax

A pattern file is a sequence of lines. Each line is one of:

- **Blank** — ignored entirely.
- **Comment** — starts with `#` (unless escaped: `\#` is a literal `#` at
  line start). Ignored entirely.
- **Pattern** — everything else.

A pattern line, after trimming trailing whitespace (unless escaped — see
§5), consists of:

```text
["!"] [anchor] segment ("/" segment)* [trailing-slash]
```

Where:

- An optional leading `!` negates the pattern (see §3).
- An optional leading `/` anchors the pattern to the directory containing the
  pattern file (see §2).
- `segment` may contain literal characters, `*`, `?`, `[...]` character
  classes, and `**` as a whole segment.
- An optional trailing `/` restricts the pattern to match directories only.

---

## 2. Anchoring

- A pattern **without** a leading `/` and **without** any other `/` in its
  body matches at any depth — e.g. `*.log` matches `app.log`,
  `src/app.log`, `a/b/c/app.log`.
- A pattern **with** a leading `/` is anchored to the directory containing
  the pattern file — e.g. `/build` matches only `build` at that directory's
  top level, not `src/build`.
- A pattern with a `/` in the middle but not at the start (e.g. `src/*.log`)
  is anchored relative to the pattern file's directory, matching only paths
  with that exact relative structure from that point.

---

## 3. Negation

A pattern prefixed with `!` re-includes a path that would otherwise be
excluded by an earlier pattern.

**Basic example:**

```text
*.log
!important.log
```

| Path | Result | Why |
|---|---|---|
| `app.log` | Ignored | Matches `*.log`, nothing re-includes it. |
| `important.log` | Included | Matches `*.log`, then `!important.log` re-includes it. |

### 3.1 The Directory-Exclusion Exception

**This is the single most commonly-misimplemented Git ignore behavior, and
the reason this implementation treats it as a first-class, separately
tested concern (see [`ROADMAP.md`](ROADMAP.md), Phases 1, 10, and 11).**

If a path's containing directory is itself excluded by a pattern, Git does
not descend into that directory to evaluate further patterns against the
directory's contents. This means a negation pattern that would otherwise
re-include a file **cannot** do so if any ancestor directory of that file is
excluded.

**Worked Example 1 — negation defeated by directory exclusion:**

```text
build/
!build/keep.txt
```

| Path | Result | Why |
|---|---|---|
| `build/keep.txt` | **Ignored** | `build/` is excluded as a directory. Git never looks inside it, so `!build/keep.txt` never gets evaluated against this path at all. |
| `build/other.txt` | Ignored | Same reasoning — never evaluated past the directory exclusion. |

This is a common source of bug reports against naive implementations: an
implementation that simply evaluates "last matching rule wins" over the
flattened pattern list, without modeling directory traversal, will
incorrectly mark `build/keep.txt` as `Included`.

**Worked Example 2 — negation succeeds because the ancestor is not excluded:**

```text
*.log
!debug/important.log
```

| Path | Result | Why |
|---|---|---|
| `debug/important.log` | **Included** | No pattern excludes the `debug/` directory itself — only the `*.log` file pattern applies, and the negation re-includes this specific file. |
| `debug/other.log` | Ignored | Matches `*.log`, no negation applies to it. |

**Worked Example 3 — negation succeeds re-including a file, but cannot
reach inside a re-excluded directory beneath it:**

```text
logs/
!logs/
logs/*.tmp
```

| Path | Result | Why |
|---|---|---|
| `logs/` (the directory itself) | Included | `logs/` excluded, then re-included by `!logs/`. The directory is no longer excluded, so Git *does* descend into it. |
| `logs/keep.txt` | Included | Directory is not excluded (per above), no pattern excludes this file. |
| `logs/debug.tmp` | Ignored | Directory is not excluded, but `logs/*.tmp` excludes this specific file, and nothing re-includes it. |

The general rule, stated precisely:

> A path `P` is `Ignored` if either (a) the last pattern matching `P` itself
> is a non-negated exclude, or (b) any ancestor directory of `P` is
> `Ignored` by this same recursive rule, **regardless of whether some
> pattern would otherwise match and negate `P` itself.** Negation of `P`
> only has effect if no ancestor of `P` is `Ignored`.

This is necessarily recursive (an ancestor's status depends on *its*
ancestors), which is why the matcher (Phase 10) must evaluate path status
top-down, directory by directory, rather than purely by examining the leaf
path's pattern matches in isolation. See
[`invariants.md`](invariants.md) for how this is enforced as a type-level
or test-level invariant.

---

## 4. File Precedence

Git ignore rules are aggregated from multiple sources, evaluated in this
order (later sources take precedence over earlier ones for any given path):

1. `$GIT_DIR/info/exclude` — repository-local, not version-controlled.
2. The file referenced by `core.excludesFile` in Git config (commonly
   `~/.gitignore_global` or similar) — user-global.
3. `.gitignore` files found in the path from the repository root down to
   the file in question, evaluated **root-first, then progressively
   deeper** — meaning a `.gitignore` in a subdirectory can override
   (including re-include via negation, subject to §3.1's exception) a rule
   set by a `.gitignore` higher up, but only for paths under that
   subdirectory.

`hgitignore`'s `compile` function (per [`SPEC.md`](../SPEC.md#public-api))
operates on a single pattern source (one `Text` value) by design — composing
multiple sources in the correct precedence order is the responsibility of
the `hgitignore` package's filesystem-aware API
(`compileFile`/directory-walking helpers), not `hgitignore-core`. This
keeps the precedence logic out of the pure core and in the IO-aware layer
where filesystem traversal naturally lives. See
[`architecture.md`](architecture.md).

---

## 5. Escape Sequences

| Sequence | Meaning |
|---|---|
| `\#` | Literal `#` at line start (otherwise starts a comment). |
| `\!` | Literal `!` at line start (otherwise negates the pattern). |
| `\ ` (backslash-space) | A literal trailing space, preserved (otherwise trailing whitespace is trimmed). |
| `\\` | Literal backslash. |
| `\*`, `\?`, `\[` | Literal `*`, `?`, `[` (suppresses wildcard meaning). |

An escape sequence that doesn't correspond to a recognized special character
(e.g. `\z`) is a validation error (`HG001`, per
[`SPEC.md`](../SPEC.md#stable-error-codes)) — it is not silently passed
through as `\z` or as `z`. This is a deliberate strictness choice: silently
accepting unknown escapes invites silent misinterpretation of patterns that
were probably typos.

---

## 6. Trailing Whitespace and Trailing Slash

- Trailing whitespace is trimmed **unless escaped** (see §5).
- A trailing unescaped `/` marks the pattern as matching directories only —
  it will not match a regular file with that name.

---

## 7. Wildcard Semantics

- `*` matches any sequence of characters **except `/`**, including the
  empty sequence, within a single path segment.
- `?` matches exactly one character except `/`.
- `[...]` matches any one character in the class (supports ranges like
  `[a-z]` and negation via `[!...]` or `[^...]`).
- `**` as a whole path segment:
  - Leading `**/` matches zero or more directories — `**/foo` matches
    `foo`, `a/foo`, `a/b/foo`.
  - Trailing `/**` matches everything inside a directory — `foo/**` matches
    `foo/a`, `foo/a/b`, but not `foo` itself.
  - `**` in the middle (`a/**/b`) matches zero or more directories between
    `a` and `b`.
  - `**` used anywhere other than as a complete path segment (e.g. `a**b`)
    is treated as two literal `*` wildcards, per Git's own documented
    behavior, not as a recursive wildcard.

---

## 8. Path Normalization

Applies to `RelativePath` values being matched (not to pattern text, which
is normalized separately during parsing per §5–6):

- All path separators are canonicalized to `/`, regardless of platform.
- Redundant separators (`a//b`) are collapsed to a single separator.
- `.` segments are removed.
- A leading `./` is removed.
- `..` segments are rejected outright (a `RelativePath` cannot escape its
  root) — this produces `HG003` (Invalid Path), not silent resolution.
- A trailing separator is preserved only when it is semantically meaningful
  (i.e. the caller is explicitly asserting "this is a directory") — the
  exact rule and its interaction with directory-only patterns (§6) is
  pinned down as part of [Phase 4](ROADMAP.md#phase-4--path-normalization)
  and recorded here once implemented, with examples.

---

## 9. Known Edge Cases

| Case | Resolution |
|---|---|
| Empty pattern file | Compiles to a matcher that includes everything. |
| Pattern file containing only comments/blank lines | Same as empty. |
| Duplicate identical patterns | No special handling needed — last one wins, same as any other pattern, with no behavioral difference from having one copy. |
| A pattern that is just `!` with nothing after it | Validation error (`HG002`, Invalid Pattern) — negating nothing is not a meaningful pattern. |
| A pattern that is just `/` | Validation error (`HG002`) — an anchor with no segment is not meaningful. |
| Pattern file not ending in a newline | Last line is still parsed as a complete pattern line; not an error. |
| Mixed `\n` and `\r\n` line endings in one file | Both are accepted as line separators; this is not treated as an error. |
| Case sensitivity | Matching is case-sensitive, matching Git's default behavior on case-sensitive filesystems. Filesystem-level case-insensitivity (e.g. `core.ignorecase` on macOS/Windows) is explicitly out of scope for `hgitignore-core` and is a concern for `hgitignore`'s filesystem layer, if addressed at all — see [`compatibility.md`](compatibility.md). |

This table is added to, never silently resolved by editing implementation
without a corresponding row here — any edge case discovered during Phase
11 corpus-building or Phase 13 fuzzing that wasn't anticipated here gets a
row added as part of fixing it.
