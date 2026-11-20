# 2. English canonical domain language

- Status: Accepted
- Date: 2026-11-19

## Context

The domain was first discussed with Spanish terms (Propietario, Inmueble, Inquilino, Contrato, Factura, Suministro). Mixed-language keys would cause drift between code, docs, and conversation. A single canonical vocabulary is needed.

## Decision

Use strictly English canonical terms as the single source of truth across the codebase and documentation. `GLOSSARY.md` is the authoritative vocabulary, defined as a table of `Canonical Term | Definition | Owner Module`. Spanish terms are not used as keys. Two legacy terms are dropped: **Factura** (superseded by Invoice) and **Suministro** (superseded by Incident/Expense).

## Consequences

- Consistent naming across frontend, backend, and docs.
- Onboarding and issue triage use the glossary.
- The glossary must be kept current when the domain changes; otherwise it becomes misleading.
