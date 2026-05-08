# molecule-skill-five-axis-review

Five-axis self-review skill for Phase 4 of the Molecule AI dev SOP. Replaces the unstructured "list 3 weakest spots" self-review with a per-axis pass that catches blind spots.

## What it does

When invoked (typically before opening a PR), walks the diff through five axes and produces structured findings with severity:

1. **Correctness** — edge cases, error paths, race conditions
2. **Readability** — name clarity, control flow, dead code, misleading docstrings
3. **Architecture** — abstractions, module boundaries, consistency with existing patterns
4. **Security** — untrusted input, secret persistence, defense-in-depth
5. **Performance** — hot-path costs, N+1, cache effectiveness

Empty axes need an explicit "no finding because X" — that's what catches blind spots.

## Install

Add to a workspace's `plugins:` list in `workspace.yaml`:

```yaml
plugins:
  - molecule-skill-five-axis-review
```

Or add to org-level defaults in `org.yaml` / `dev-department.yaml`:

```yaml
defaults:
  plugins:
    - molecule-skill-five-axis-review
    - ... other plugins ...
```

The plugin UNIONs with workspace-level plugins per the platform's `mergePlugins` semantics.

## How agents use it

The skill is invoked when an agent finishes a substantive change and is preparing to open a PR. The agent walks the five axes, writes findings, addresses Critical/Required items, then opens the PR with the self-review section in the description.

## Why this exists

The original SOP Phase 4 said "list the three weakest spots." That biases toward visible surface issues (names, comments) and skips systematic ones (security boundaries, architectural fit). Structured five-axis review forces explicit attention to each category.

Real cases caught after retrofitting this skill (internal#77 dev-department extraction, 2026-05-08):

- **Cache-validity gap** — partial cache write looked valid; would have hit production silently with no self-heal. Surfaced as Correctness finding.
- **Auth token persistence in `.git/config`** — token in URL userinfo persisted on disk. Surfaced as Security finding.
- **Misleading function name** after a mid-development refactor. Surfaced as Readability finding.

All three were missed by the same author's "three weakest" review pass on the same code. The structured pass caught them on the next reviewer-perspective sweep.

## Refs

- `skills/five-axis-review/SKILL.md` — the skill body
- Molecule AI dev SOP — Phase 4 (canonical: `molecule-ai/internal/runbooks/dev-sop.md`)
- `molecule-skill-code-review` — sibling plugin for reviewing OTHER people's PRs (this one is for self-review before opening yours)
