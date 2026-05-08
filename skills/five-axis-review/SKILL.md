---
name: five-axis-review
description: "Phase 4 self-review for the Molecule AI dev SOP. Walks the diff through five axes — Correctness / Readability / Architecture / Security / Performance — each producing a finding with severity (Critical / Required / Optional / Nit / FYI) or an explicit 'no finding because X'. Empty axes need justification — that's what catches blind spots. Replaces unstructured 'list 3 weakest spots' self-review."
origin: molecule-skill-five-axis-review
---

# Five-axis self-review (Phase 4 of the dev SOP)

You finished implementing. Tests pass. Before opening the PR, walk the diff through five axes. **For each axis, produce either a finding (with severity) OR an explicit "no finding because X."** Empty axes without justification are the failure mode this skill is designed to prevent — they're where blind spots hide.

## When to invoke

- Just before opening a PR (after Phase 3 implementation, before Phase 4 verification publishes the change).
- When asked by the user / a peer to "do a hostile self-review" or "five-axis review."
- After any refactor of code you wrote, before declaring done.
- NOT for trivial single-line changes or pure docs (overhead exceeds value).

## The five axes

Walk them in this order. Resist the urge to merge axes — each catches a distinct class of issue.

### 1. Correctness

> Does the code do what it claims to do, including in conditions you didn't write the test for?

Check:
- Edge cases — null/empty/zero inputs, boundary values (0, 1, max, max+1), unicode, very long strings.
- Error paths — what happens when each downstream call fails? Are the failures distinguishable?
- Race conditions — concurrent reads/writes, TOCTOU windows, cache invalidation timing.
- State consistency — invariants the type system doesn't enforce (e.g., "cache is either complete or absent, never partial").
- Off-by-one — loop bounds, slice indices, time-window edges.
- Idempotency — running twice in a row should be safe (or explicitly documented why not).

Do NOT just check that tests pass. Tests pass in conditions tests were written for.

### 2. Readability & simplicity

> Can another engineer (or your future self in 6 months) understand this without the original context?

Check:
- Names — descriptive, consistent with codebase conventions, free of `temp` / `data` / `result` / `helper` / `_unused`.
- Function names match what the function does NOW (not what it did before a mid-development refactor).
- Control flow — straight-line where possible; deeply nested ternaries / early-return spaghetti / unclear short-circuits flagged.
- Comments explain WHY, not WHAT. Self-explanatory code doesn't need comments.
- Dead code — `_ = unused_import`, backwards-compat shims with no consumer, `// removed` comments, no-op variables.
- Misleading docstrings — function used to do X, was refactored to do Y, docstring still says X.
- Could this be done in fewer lines without sacrificing clarity?

### 3. Architecture

> Does this fit the system's design or fight it?

Check:
- Existing patterns followed (look at sibling files, not just the diff).
- Module boundaries — does the change leak concerns across layers?
- Dependency direction — no circular imports; high-level modules don't reach into low-level internals.
- Abstractions earning their complexity — interfaces with one implementation are smell unless test-injection is the reason.
- Single source of truth — same logic in two places after this change?
- Coupling — does this change make later refactors harder than they were before?

### 4. Security

> What untrusted input does this code touch, and what's the trust boundary?

Check:
- Input validation at boundaries — yaml, env vars, HTTP params, file contents, network responses.
- Secret persistence — does any code path write tokens / passwords / keys to disk, logs, or process arguments?
- Auth/authz — every privileged operation has the appropriate check; failures fail closed.
- Path traversal — `..`, absolute paths, symlinks crossing trust boundaries.
- Injection — string concat into SQL / shell / yaml / template / URL; should be parameterized.
- Defense-in-depth — even if downstream tools validate, validate at YOUR boundary too.
- Output encoding — anything user-controlled rendered to HTML / JSON / shell context?
- Dependencies — new ones from trusted sources, no known CVEs?

### 5. Performance

> What's the cost in the hot path, under load, with realistic data sizes?

Check:
- N+1 patterns — loops doing per-item DB queries / network calls.
- Unbounded operations — `os.walk` of arbitrary depth, ungated recursion, accumulating slices that may grow without limit.
- Cache effectiveness — when does cache hit vs miss? Does miss mean re-doing 10x the work, or re-doing the failed step?
- Hot-path allocations — large objects created per request, regex compiled per call, file I/O in render loop.
- Async — sync operations that should be async, or the reverse.
- Pagination — list endpoints / queries / loops without bound.

## Severity labels (use exact prefix in findings)

| Prefix | Meaning | Action |
|---|---|---|
| **Critical:** | Blocks merge | Security vuln, data loss, broken core functionality |
| **Required:** | Must fix before merge | Real bug, real correctness gap, real security weakness |
| **Optional:** / **Consider:** | Worth addressing | Sub-critical improvement, may defer with reasoning |
| **Nit:** | Cosmetic | Author may ignore |
| **FYI:** | Informational | Context for future readers |

## Output shape

After walking all five axes, produce a structured report. Use this exact format so PR reviewers can scan it quickly:

```markdown
## Phase 4 self-review (five-axis)

**Correctness:**
- Required: <finding> — <one-line proposed fix>
- (or) No finding because <explicit reason — not "I checked" but WHY there's no concern>

**Readability:**
- Nit: <finding>
- (or) No finding because <reason>

**Architecture:**
- (axis-specific findings or no-finding-because)

**Security:**
- Critical: <finding> — <one-line fix>
- (or) No finding because <reason>

**Performance:**
- (axis-specific findings or no-finding-because)
```

Then ADDRESS the Critical and Required items before opening the PR. Optional/Nit/FYI can ship as follow-ups (file as parked tasks if not addressed in this PR).

## What this replaces

The previous SOP Phase 4 said: "Review the diff yourself as if you were a hostile reviewer. List the three weakest spots."

Failure mode that drove the change: "three weakest" lets the reviewer pattern-match on whatever surfaces visibly — naming, comments, easy stuff — and skip the systematic axes (security boundaries, architectural fit, performance under load). Real cases retrofit-caught (internal#77, 2026-05-08): cache validity, token persistence in `.git/config`, misleading function name post-refactor — all missed by the same author's "three weakest" pass on the same code.

## Anti-patterns

- **Rubber-stamping yourself.** "All five axes look fine" without per-axis specifics is a fail. Each axis needs either a finding or a justified absence.
- **Conflating axes.** "Readability AND architecture: name is bad" — pick one, the issue is structurally one or the other.
- **Marking real issues as Optional to ship faster.** If the issue is real, fix it or document the deferral as a parked task with severity.
- **Treating tests passing as sufficient.** Tests pass in conditions tests were written for. The Correctness axis specifically asks about conditions tests didn't cover.
