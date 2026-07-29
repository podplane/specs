# Podplane Specs — Agent Guide

## Purpose

This repository contains Podplane technical specifications. Specs define intended
behavior, contracts, boundaries, and implementation plans. Production code belongs
in its owning repository, not here.

## Before Editing

- Read the entire spec and directly related specs.
- Inspect relevant code, configuration, schemas, and tests in each owning repository.
- Read each owning repository's `AGENTS.md`. Repositories are normally under
  `~/Workspace/<github-org>/<repo>`; `podplane/workspace` defines the managed list.
- Verify external standards against authoritative sources.
- Distinguish current behavior from proposals and resolve conflicts with existing
  specs explicitly.

## Specification Status

The first content after every spec's top-level heading must be a status line:

```markdown
# Specification Title

> **STATUS**: Draft
```

Allowed statuses:

- `Draft` — material design decisions remain unresolved.
- `Ready for implementation` — contracts and decisions are settled enough to build,
  but material implementation has not started.
- `In progress` — material implementation has started, but substantial required
  scope remains.
- `In review` — the majority of the required scope is implemented, but final review
  and verification have not yet established that the spec is complete.
- `Implemented` — the full required scope is implemented and verified.

Split independently deliverable parts into separate specs when their statuses differ.

Change status only with evidence. Verify code and tests before using `Implemented`;
do not rely on plans or checklists. New specs start as `Draft`. Ask the user when an
existing spec's status cannot be established.

Support files such as `README.md` do not require a status.

## Quality Bar

Cover these subjects where relevant:

- goals, non-goals, terminology, and ownership;
- user-visible behavior and configuration, API, protocol, schema, and storage
  contracts;
- state transitions, concurrency, failure handling, recovery, and security;
- compatibility, migration, rollout, and operations; and
- an ordered implementation plan and verification strategy.

Include enough detail that implementers do not need to make material product or
architecture decisions. Otherwise, keep the spec in `Draft`.

Clearly separate current behavior, requirements, future work, and non-goals. Keep
examples consistent with the contract and label illustrative examples.

For cross-repository work, name each `github.com/<org>/<repo>` owner, define the
shared contract first, order work by dependency, and include integrated validation.

## Writing and Change Conventions

- Use one level-one title and a logical heading hierarchy.
- Write direct, precise prose and use terminology consistently.
- Use requirement language deliberately; prefer contracts and invariants over
  speculative implementation detail.
- Use language-tagged code fences and relative links such as `[ZERO.md](./ZERO.md)`.
- Link dependencies after the status line instead of duplicating their contracts.
- Preserve established decisions unless intentionally revising them. Update affected
  examples, plans, validation, and linked specs when contracts change.
- Keep changes focused and surface assumptions or unresolved questions explicitly.
- Never include secrets, credentials, private endpoints, or environment-specific
  values.

## Validation

- Confirm the title, status, headings, links, examples, and implementation claims.
- Check consistency with related specs and owning repositories.
- Run `git diff --check`.
- Before setting `Implemented`, verify relevant repository tests and integrated paths.
  Report anything that could not be verified.
