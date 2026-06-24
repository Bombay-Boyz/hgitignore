# AI Coding Kickoff Prompt

Copy everything in the fenced block below as your first message to whatever
coding AI/agent you're using (Claude Code, Cursor, etc.), with the repo
(this scaffold) as its working directory or attached context.

Why a prompt this specific, rather than "build hgitignore per the spec":
without explicit guardrails, most coding agents will skip straight to
writing a parser because that's the "interesting" part — and the entire
point of this project's process (see `SPEC.md`'s Guiding Principle) is that
no implementation code should exist before Phase 1 and Phase 2's documents
are written and reviewed. The prompt below encodes that constraint
explicitly so the agent can't route around it.

---

```
You are implementing "hgitignore," a Haskell project with a strict,
phase-ordered build process. Before writing any code, read these files in
this exact order and confirm you've understood each before moving to the
next:

1. README.md — project orientation and contributing rules.
2. SPEC.md — the full charter: architecture, type-safety rules, forbidden
   functions, error code policy, public API shape.
3. docs/ROADMAP.md — the 16 phases, each with exit criteria. Find the
   FIRST phase whose checkboxes are not all checked. That is the only
   phase you are allowed to work on right now.

Hard rules, no exceptions:

- Do not write code for any phase before the one currently in progress.
  If Phase 1 (Formal Semantics) isn't checked off in docs/ROADMAP.md, you
  write documentation, not Haskell.
- Do not skip a phase's exit criteria. If ROADMAP.md says a phase needs a
  property test, write the property test as part of that phase, not later.
- Never use: head, tail, init, last, read, fromJust, error, undefined,
  unsafePerformIO, unsafeCoerce — in any production code path. (Test code
  has a narrow exception list in docs/invariants.md — check before using
  any of these even there.)
- All validation failures are values (Either CompileError a), never thrown
  exceptions.
- Compile every package with -Wall -Wcompat -Wincomplete-patterns -Werror.
  Treat any warning as a build failure you must fix, not suppress.
- If you think the spec is wrong or ambiguous, stop and flag it instead of
  silently picking an interpretation — especially anything touching the
  directory-exclusion negation exception in docs/semantics.md section 3.1,
  which is the most commonly-misimplemented part of Git's ignore semantics.
- After finishing a phase's deliverables, go back to docs/ROADMAP.md and
  check off every exit-criteria box for that phase, and update the status
  table in README.md. Do not check a box for work you didn't actually do
  (e.g. don't check "property test passes" if you wrote the test but
  haven't run it).
- Stop and report back once the current phase's exit criteria are all
  genuinely met. Do not continue into the next phase without being asked,
  even if it seems like a natural continuation.

Start by telling me: which phase is currently open, and what its exit
criteria are. Then propose a short plan for that phase only before writing
anything.
```

---

## Notes on using this with different tools

- **Claude Code / agentic CLI tools**: paste as-is as the first turn. These
  tools can read the repo directly, so step 1–3 above ("read these files")
  will actually happen.
- **Chat-based tools without file access**: you'll need to paste the
  contents of `README.md`, `SPEC.md`, and `docs/ROADMAP.md` into the
  conversation first, or upload them, before sending this prompt.
- **Re-starting a session mid-project** (e.g. you're resuming after Phase 3
  is done): the prompt still works unmodified — step 3 has the agent find
  the first unchecked phase itself rather than you having to tell it which
  phase to resume at. Just make sure `docs/ROADMAP.md`'s checkboxes are
  actually up to date before resuming, since the agent trusts them.
