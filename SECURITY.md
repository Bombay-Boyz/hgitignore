# Security Policy

## Guarantees

`hgitignore` provides the following security guarantees by construction:

- **No catastrophic backtracking.** The compiled-automaton approach (NFA → DFA,
  Phases 8–9) eliminates backtracking entirely. Match time is O(path length)
  regardless of pattern complexity or input content.

- **No regex engine dependency.** `hgitignore` does not use any regex library.
  All pattern matching is performed by a purpose-built DFA compiled from the
  parsed AST. There is no way to supply a "regex bomb" through a pattern file.

- **Bounded memory growth.** Memory usage is O(pattern count) for the compiled
  matcher and O(path length) during matching. No input — however large or
  adversarially constructed — causes unbounded memory allocation.

- **Resistance to malicious input.** No pattern file or path string, however
  constructed, can produce non-linear time or unbounded memory behaviour.
  Malformed input produces a `CompileError` value; it never causes a crash,
  hang, or exception.

- **No runtime exceptions in production code paths.** All validation failures
  are returned as `Either CompileError a` values. The forbidden-function list
  in `SPEC.md` (which includes `error`, `undefined`, `unsafePerformIO`, etc.)
  is enforced by CI on every PR.

## Reporting a Vulnerability

If you believe you have found a security vulnerability in `hgitignore`, please
do **not** open a public GitHub issue. Instead, contact the maintainers directly
(contact details to be added before Phase 15 / public release).

Please include:
- A description of the vulnerability and its potential impact.
- Steps to reproduce (a minimal pattern file and/or path string that triggers
  the issue).
- The version of `hgitignore` and GHC you are using.

We aim to acknowledge reports within 72 hours and provide a fix or mitigation
plan within 14 days for confirmed vulnerabilities.
