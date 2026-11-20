# 1. Record architecture decisions

- Status: Accepted
- Date: 2026-11-19

## Context

As the project evolves, architectural choices need a durable, discoverable, and versioned record so future contributors understand why the system is shaped the way it is.

## Decision

We will record architecture decisions as ADRs in `docs/adr/`, numbered sequentially, using the format Title / Status / Context / Decision / Consequences. New decisions are added as new records rather than editing old ones; superseded records are marked Deprecated or Superseded.

## Consequences

- Decisions are discoverable and versioned alongside the code.
- Reviewers and future contributors can trace the rationale for each boundary.
- Adds a small maintenance obligation to write an ADR for significant decisions.
