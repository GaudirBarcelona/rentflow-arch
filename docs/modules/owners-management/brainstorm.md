# Owners Management — Architecture Brainstorm

| Field | Value |
| :--- | :--- |
| **Author** | Sacristaan (Marc Sacristán, `marc@sacristan.com.es`) |
| **Date** | 2026-11-24 |
| **Issue** | GitHub Issue #3 — Owners Management module |
| **Phase** | SDD Prompt 2 — `/egm-brainstorm` (BRAINSTORM mode) |
| **Branch** | `docs/issue-3-owners-brainstorm` |
| **Status** | Draft for review. No implementation tasks, no application code. |
| **Successor** | `/egm-draft` consumes this document as its intake source |

---

## 1. Sources read, and how they are used

Every statement below is anchored to an accepted artifact. Nothing in this document invents a canonical term that the glossary does not define or that Section 5 does not explicitly propose adding.

| Artifact | What it settles for this module |
| :--- | :--- |
| [`GLOSSARY.md`](D:/ProyectopisosyayaDeCero/rentflow-arch/GLOSSARY.md) | Canonical terms; the ownership matrix at `GLOSSARY.md:46-55`; module ownership; lifecycle states; open questions |
| [ADR-0001](D:/ProyectopisosyayaDeCero/rentflow-arch/docs/adr/0001-record-architecture-decisions.md) | New decisions are new records; existing ADRs are superseded, never edited |
| [ADR-0002](D:/ProyectopisosyayaDeCero/rentflow-arch/docs/adr/0002-english-canonical-domain-language.md) | English canonical keys; the glossary is authoritative and "must be kept current… otherwise it becomes misleading" |
| [ADR-0003](D:/ProyectopisosyayaDeCero/rentflow-arch/docs/adr/0003-module-ownership-boundaries.md) | Owners Management owns `Owner` and `Ownership Share`; cross-module access is read-only; read-only cross-module **foreign keys** are explicitly sanctioned |
| [ADR-0004](D:/ProyectopisosyayaDeCero/rentflow-arch/docs/adr/0004-contract-as-a-dedicated-aggregate.md) | Contract is its own aggregate; Invoice and Income live under it; a property may hold multiple active Contracts |
| [ADR-0005](D:/ProyectopisosyayaDeCero/rentflow-arch/docs/adr/0005-shared-document-store.md) | One Document Store; owning records are Tenant, Candidate, Contract — **Owner is absent** |
| [ADR-0006](D:/ProyectopisosyayaDeCero/rentflow-arch/docs/adr/0006-multi-tenant-saas-isolation.md) | Account-scoped isolation at the data-access layer; the primary user is a property-management **agency**; account dimensions belong in every index |
| [`AI-Coding.SDD.md`](D:/ProyectopisosyayaDeCero/rentflow-arch/AI-Coding.SDD.md) | This document's shape: ≥2 options with assumptions, trade-offs, failure modes, reversibility, operational cost, disqualifying conditions; no implementation tasks |

**Repository state.** `git ls-files` shows eleven tracked files: documentation, six ADRs, and a zero-byte `write_arch.py`. There is no schema, no migration, no Java, no TypeScript, and no OpenAPI file. This is a greenfield design. Every statement about "existing data" is therefore conditional, and every performance number is a hypothesis to be measured rather than a measurement.

**A documentation defect that affects this module directly.** [`ARCHITECTURE.md:12-18`](D:/ProyectopisosyayaDeCero/rentflow-arch/ARCHITECTURE.md) still names `Propietario`, `Inmueble`, `Inquilino`, `Contrato`, `Factura`, `Suministro` and describes a tenant-request → owner-approval → contract flow. ADR-0002 makes English canonical, the glossary drops `Factura` and `Suministro` (`GLOSSARY.md:65`) and states there is no Application entity (`GLOSSARY.md:66`). A developer reading the top-level document first learns the vocabulary the ADRs prohibit. This is not this issue's fault, but this issue is the first one where the contradiction costs real design time, so it is listed as a governance action in Section 17.

---

## 2. What the accepted artifacts already decide — and where they contradict each other

Four questions the issue leaves open are actually already answered. Three others are answered *inconsistently*, and this issue cannot proceed without resolving them.

### Already settled (do not re-open)

1. **The module owns the relation.** `Owner` and `Ownership Share` belong to Owners Management (`ADR-0003:16`, `GLOSSARY.md:48`). The writer is named.
2. **Cross-module foreign keys are permitted.** `ADR-0003:12`: "Every other module holds read-only references (**foreign keys**) and must not mutate an entity it does not own." A foreign key from the share table to the Real Estate table is sanctioned by the accepted decision, not a violation of it. This removes a debate that would otherwise consume the whole design: referential integrity across the boundary is available and expected.
3. **Ownership unknown is a documented state.** `GLOSSARY.md:18` defines `Ownership Share` as "the percentage stake a specific Owner holds in a Real Estate property (**0 when unknown**)". Partial knowledge is not an error condition in this domain; it is the normal state of a portfolio being onboarded.
4. **Isolation is account-level and schema-enforced.** `ADR-0006:18`: "migrations and indexes must include it."

### Contradictory, and blocking

**C-1. "Real Estate lists its owners" is not supported by the matrix that grants reads.**
`ADR-0003:25` says "Real Estate lists its owners and contract history", and `ADR-0003:12` says each module holds read-only references. But the ownership matrix grants Real Estate Management a read of **`Owner`** only (`GLOSSARY.md:49`) — not `Ownership Share`. A property held by three owners cannot be listed from a grant that covers the Owner record alone, and a property-level pointer to a single owner is exactly the single-owner assumption that co-ownership breaks.
*Consequence:* the property screen either reads a grant it does not have, or keeps a second, single-owner representation that will drift. This issue must extend the matrix.

**C-2. `Contract Template` requires an "owner name" from a module granted no owner read.**
`GLOSSARY.md:38` defines Contract Template as "a merge-field document (tenant name, property address, rent, dates, **owner name**)". It belongs to Settings (`GLOSSARY.md:54`), whose read list is empty — and the field is **singular**, which is already wrong for a co-owned property.
*Consequence:* the first co-owned contract either emits one arbitrary owner's name or blocks. The merge-field contract is part of this module's consumer surface whether or not the issue says so.

**C-3. The `Owner` definition excludes the owners this issue must support.**
`GLOSSARY.md:16`: an Owner "owns **one or more** rental properties". Issue #3 requires owners with zero properties. One of the two is wrong, and ADR-0002 says a stale glossary is a defect rather than a nuisance.

### Unresolved by silence (this document must decide or defer explicitly)

Who may hard-delete; what happens to shares when an owner is archived; whether ownership needs effective dating; whether the tax ID is the identity; and who the audit actor is, given that ADR-0006 defines accounts and never users.

---

## 3. Problem statement

Rentflow is a multi-tenant SaaS whose primary user is a property-management agency (`ADR-0006:8`). The agency manages rental units on behalf of owners. Today no accepted artifact gives the system a first-class notion of **who owns the property**, and three concrete failures follow:

1. **The agency cannot answer "whose flat is this?" from the system.** The relation exists in the domain (ADR-0003 gives it a module) but not in any contract.
2. **Co-ownership cannot be represented.** `GLOSSARY.md:17` defines Co-ownership and `ADR-0003:25` says Real Estate lists owners in the plural, yet nothing in the matrix lets a property hold more than one owner reference. Spanish residential stock is routinely held `proindiviso` — spouses, siblings after an inheritance, a company plus an individual.
3. **The agency cannot produce the owner-side facts its own documents need.** The contract template emits an "owner name" (`GLOSSARY.md:38`); with co-ownership that field is underspecified, and with no owner module it has no source.

The value of this module is not a CRUD screen. It is **a single, trustworthy record of who owns what share of which property, of a quality the agency can put in front of a landlord.** Everything below is judged against that sentence.

**What the module is *not*, and this matters for every later decision:** the glossary gives ownership **no monetary consumer**. `Income` is "derived per active Contract" (`GLOSSARY.md:30`); `Invoice` is a rent obligation under a Contract (`GLOSSARY.md:29`); the Dashboard reads Contract, Expense, and Incident (`GLOSSARY.md:55`). Nothing consumes an `Ownership Share` percentage to compute money. Any argument in this document that treats a correct total as a billing-correctness requirement would be inventing a requirement, and is therefore rejected in Section 12.

**Success signals** (proposed, to be confirmed):

- An agency employee can answer "who owns this flat, and in what proportion?" in one screen, including "we do not know yet".
- A property onboards before its titling is known, without inventing numbers.
- An owner with a Spanish tax ID cannot be silently duplicated inside an account.
- Support can reconstruct who changed a share, when, and from what, from the system rather than from logs.
- A co-owned contract can name every owner, not one arbitrary one.

---

## 4. Scope

### In scope

- `Owner` aggregate: natural person (DNI/NIE) and legal entity (CIF), with contact and address data, a stable surrogate identity, and an explicit tax-identifier model.
- `Ownership Share`: the quota relation between an Owner and a Real Estate property, with co-ownership.
- Owner create, read, update, search, archive, restore.
- Owner portfolio: the properties an owner holds.
- Co-ownership editing for a property (the single write path for shares).
- Change audit for owner and share mutations.
- Responsive UI on the existing React PWA (one implementation, two viewports).
- Account-scoped isolation on every read and write.

### Out of scope, with the reason

| Excluded | Reason |
| :--- | :--- |
| Rent split, owner payouts, owner statements | No consumer exists in the glossary or ADRs. This is the single largest scope trap in the issue (Section 12.4). |
| Ownership rights types (`usufructo`, `nuda propiedad`, bare ownership) | Real in Spanish practice, but the glossary models one percentage and adding a rights taxonomy changes the concept. Deferred with a trigger. |
| Point-in-time ownership ("who owned it on the invoice date") | Needs either effective dating or a consumer. Handled by the contract-side lessor snapshot decision, Section 10.4. |
| Owner dedupe / merge tooling | Schema must permit it (stable surrogate key, alias table); the operation is deferred. |
| Owner identity-document upload | ADR-0005 does not list Owner as an owning record. Gap recorded in Section 10.5 and raised as a governance action, not silently built. |
| Bulk CSV import | Deferred, but the uniqueness and normalisation rules must not make it impossible. |
| Owner login or owner portal | Owners are records, not users. Confirm this is intended. |
| Property-side ownership editing | The ADR-0003 revisit clause stays dormant; see Section 8.3 and option C. |

---

## 5. Canonical vocabulary

Terms below are quoted from the glossary. Proposed additions are marked **NEW** and must be added to `GLOSSARY.md` in the same change that implements them (ADR-0002).

| Term | Definition used here | Status |
| :--- | :--- | :--- |
| Owner | A natural person or legal entity that owns rental properties and on whose behalf the agency manages them. | Glossary `:16` — **definition needs editing**, see C-3 |
| Co-ownership | A property owned by more than one Owner. | Glossary `:17` |
| Ownership Share | The percentage stake a specific Owner holds in a Real Estate property (0 when unknown). | Glossary `:18` |
| Real Estate | A rental unit or listing managed by the agency. | Glossary `:19` |
| Contract | The rental agreement linking Tenants to a Real Estate property. | Glossary `:27` |
| Invoice / Income | Financial records under the Contract context. | Glossary `:29-30` |
| **Owner Type** | Natural person or legal entity. Drives which identifier formats are legal and how the display name is rendered. | **NEW** |
| **Tax Identifier** | A typed, normalised tax identifier (DNI, NIE, CIF, Passport, or *other*) attached to an Owner; one is primary, others are historical aliases. | **NEW** |
| **Legal Name** | The single computed display name of an Owner — `given names + surnames` for a natural person, registered name for a legal entity. | **NEW** |
| **Ownership Allocation** | The complete set of Ownership Shares for one property, and the transactional unit in which they are written. | **NEW** |
| **Titling Status** | Derived state of an allocation: `Unassigned` (no shares), `Unknown` (shares recorded, no known stake), `Partial` (known total below 100), `Complete` (exactly 100). `Over` (> 100) is rejected and therefore not a stored state. | **NEW** |

**Language rule.** Canonical names are English (ADR-0002). End-user UI strings are Spanish, because the product serves Spanish agencies (DNI/NIE/CIF, Idealista export, LOPD/GDPR consent). Spanish is used for *labels*, never for keys, table names, or API fields. This document follows that split and marks UI copy explicitly.

---

## 6. Invariants

Numbered so the options can be scored against them.

- **I-1 — Tenant isolation.** A record in another account is indistinguishable from one that does not exist. Includes the non-obvious path: a share in account A must not reference a property in account B.
- **I-2 — Referential soundness.** No share references a missing property or a missing owner, enforced in the schema rather than by convention.
- **I-3 — Over-allocation is impossible.** The shares of one property never sum above 100. This is the only ownership rule that is unconditionally enforceable, because it is incoherent under every reading of the term.
- **I-4 — Unknown is representable.** "We have not recorded this owner's stake" is stored and displayed distinctly from "this owner's stake is zero".
- **I-5 — Single writer.** Exactly one module and one operation family mutates ownership.
- **I-6 — Single representation.** There is exactly one place a property's owners are recorded. No property-level owner column may exist as a second truth.
- **I-7 — Stable identity.** `Owner.id` is immutable and never reused. Nothing outside Owners is keyed on a tax identifier.
- **I-8 — Archival preserves.** Archiving never removes a share, never orphans a reference, and never eliminates history.
- **I-9 — Auditable mutation.** Every ownership and owner-lifecycle change is attributable to an actor, a time, and a before/after state, in a durable store the support path can query.

---

## 7. Dimension 1 — Data model and co-ownership

### 7.1 Two aggregates

```
Owner                      (aggregate root, id = immutable UUID)
  ├── account_id
  ├── owner_type           natural_person | legal_entity
  ├── legal_name           computed, searchable
  ├── tax_identifier       typed, normalised, one primary
  ├── contact / address    personal data, redactable
  └── lifecycle            Active | Archived (+ archived_at, reason, actor)
  └── version              optimistic lock

TaxIdentifierAlias         (owner_id, type, value, reason, effective_from)

OwnershipAllocation        (aggregate root, key = (account_id, property_id))
  ├── version              optimistic lock, the concurrency token
  └── OwnershipShare [0..n]
        ├── owner_id
        ├── units          INTEGER NULL   — NULL = unknown
        └── (account_id, property_id) composite FK
```

```
erDiagram
    OWNER ||--o{ TAX_IDENTIFIER_ALIAS : "has history"
    OWNER ||--o{ OWNERSHIP_SHARE : "holds"
    OWNERSHIP_ALLOCATION ||--|{ OWNERSHIP_SHARE : "comprises"
    REAL_ESTATE ||--|| OWNERSHIP_ALLOCATION : "is described by (account_id, id)"
    OWNER {
        uuid id PK
        uuid account_id
        string owner_type
        string legal_name
        string lifecycle
        int version
    }
    OWNERSHIP_ALLOCATION {
        uuid account_id PK
        uuid property_id PK
        int version
    }
    OWNERSHIP_SHARE {
        uuid id PK
        uuid owner_id FK
        int units "nullable, 1/10000 of the property"
    }
```

The decisive structural choice is the second root: **the invariant belongs to the whole set of shares for a property, not to an individual share.** A `CHECK` on a share row can only see that row; PostgreSQL cannot express a cross-row sum as a constraint. Making the allocation the aggregate root gives the sum a transactional home and gives concurrent edits a version to conflict on (Section 8.5).

`GLOSSARY.md:18` calls Ownership Share an entity owned by Owners Management. That remains true: it is an entity *inside* the allocation aggregate, not a standalone root. This does not contradict ADR-0003.

### 7.2 The percentage, the unit, and the residue

`GLOSSARY.md:18` calls the term a **percentage** and defines 0 as unknown. Both facts must survive, and neither fixes the storage type.

| Representation | 7 heirs at equal shares | Verdict |
| :--- | :--- | :--- |
| `double` / `float` | 14.285714285714286 × 7 = 99.99999999999999 | **Rejected.** Equality and sum checks become epsilon-dependent; a JSON round trip can change the stored value. |
| `numeric(5,2)` — literal percentages | 14.29 × 7 = 100.03 | Workable only with a "last row absorbs the remainder" rule. It *looks* correct, which is why it gets embedded in API contracts and statements before the residue rule exists. |
| **Integer units, `total_units = 10000`** | 1429×5 + 1428×2 = 10000 | **Recommended.** Exact integer equality; the remainder is an explicit, testable decision rather than a rounding artefact. |

Recommendation: **store integer units; render percentages.** The glossary term stays `percentage` because that is what users see and what the domain means; the stored form is a fixed-point encoding of it. This is an encoding decision, not a vocabulary change, and it should be recorded as such so nobody re-opens it as a glossary conflict.

**Residue rule** (must be written down, in this order): allocate by largest remainder, deterministic on `owner_id`, and one row per property is designated the residue holder when a split does not divide evenly. The UI never asks a user to type the residue: the last row is a computed, read-only remainder and the user is offered "Dividir a partes iguales".

**Unknown, not zero.** The glossary's "0 when unknown" collapses two different facts — *this owner owns nothing* and *we have not recorded this owner's stake* — and a total computed over that collapse can report 100% coverage when one owner was never entered. Recommendation: `units` is `NULL` for unknown and `0` for a genuine zero stake. The API renders `{ "percentage": 0, "known": false }` so the glossary's wording is honoured at the contract boundary while the ambiguity disappears inside. If the team prefers the literal single-value reading, the fallback is a non-null `units` defaulting to 0 with the ambiguity accepted and documented — but then I-4 fails, `Titling Status: Unknown` cannot be distinguished from `Partial`, and the data-quality reporting in Section 10.7 is not buildable.

**This decision needs a glossary edit and an ADR.** It is recorded as decision D-01 and governance action G-1, because ADR-0002 makes the glossary a control artifact: changing a stored representation of a defined term is a decision, not an implementation detail.

### 7.3 The 100% question

The issue asks whether a mismatch is a warning or a hard block. Framed that way it is a binary about *when validation runs*. It is actually three separate questions, and only one of them has a defensible universal answer.

| Question | Answer adopted | Why |
| :--- | :--- | :--- |
| May an allocation exceed 100? | **Never.** Hard rejection. | A 120% property is incoherent under every reading. It is the only rule that needs no knowledge of the product. |
| Must an allocation reach 100 before saving? | **No.** Partial and unknown are legal saved states. | `GLOSSARY.md:18` documents unknown. A hard block would make the glossary's own definition unstorable, and would force employees to invent a split to unblock onboarding — converting a data-quality problem into a confident lie. |
| Must an allocation reach 100 before a consumer uses it? | **Only if a consumer exists.** Today none does. | No artifact consumes the percentage for money (Section 3). Inventing a gate to justify the invariant would be fabricating a requirement. |

**Adopted policy: `≤ 100` is the hard invariant; `= 100` is a derived data-quality signal, not a gate.** The `Titling Status` (`Unassigned` / `Unknown` / `Partial` / `Complete`) is computed from the stored shares and surfaced in the portfolio, the property screen, and a data-quality report. When a real consumer appears — owner statements, rent distribution, a tax export — the gate is added *at that consumer*, reading the same status, without touching the write path. That placement is the whole design: the person typing has the least information, and the party that needs the guarantee is not in the room.

**An explicit override is therefore unnecessary**, which is a simplification worth naming: with no hard `= 100` rule there is nothing to override. A permission-gated override flag would have been required under a strict policy and would have become a permanent unaudited back door.

### 7.4 Identity: surrogate key, typed identifiers, and the role boundary

- **`Owner.id` is a UUID and the only foreign-key target.** The tax ID is a mutable *attribute*. This is not a stylistic preference: a DNI becomes a NIE on residency change, a CIF changes on corporate transformation, an autónomo's identifier changes on status change, and mistyped digits are corrected routinely. If the tax ID were the key, every correction would cascade through shares, contracts, and audit — and the uniqueness rule would block creating the corrected record while the stale one exists.
- **Identifiers are typed, not one column.** `DNI | NIE | CIF | Passport | Other`. The glossary already recognises Passport as an identity document (`GLOSSARY.md:33`); the issue's "DNI/NIE or CIF" silently excludes a non-resident owner who has only a passport, who is a normal customer for a Barcelona agency.
- **Aliases record change.** `TaxIdentifierAlias` holds previous identifiers with a reason and date, so "this owner used to be NIE X" is answerable and so the uniqueness check can consider history rather than fighting it.
- **Uniqueness is per account, over the normalised active identifier.** Normalisation (uppercase, strip separators, zero-pad DNI to eight digits) is part of the key, otherwise `12345678Z`, `12345678z` and `12345678 Z` are three owners to the index and one person to the tax authority.
- **CIF is not a document type** anywhere in the current artifact set (`GLOSSARY.md:33` lists DNI/NIE and Passport). This is a real gap for a module about legal identity and is raised in Section 10.5.

**The role boundary, stated explicitly.** The same legal person can be a landlord and a tenant. `Tenant` is "identified by name, phone, and email" (`GLOSSARY.md:24`) — no tax ID at all — and `Candidate` carries an ID document (`GLOSSARY.md:26`). There is no shared Party identity, and merging across roles would require Owners to write Tenant records, which `ADR-0003:12` forbids. **Decision: identity is role-local in v1; cross-role deduplication is explicitly out of scope and cannot be added later without a Party context.** This is recorded as an accepted limitation rather than left to be discovered.

### 7.5 Verdict for Dimension 1

Two aggregates (`Owner`, `OwnershipAllocation`), integer units with `NULL` for unknown, `≤ 100` as the only hard rule, `Titling Status` derived, surrogate identity with typed identifier aliases, role-local identity. Four glossary edits and one new ADR are prerequisites (Section 17).

---

## 8. Dimension 2 — Module boundaries and API

### 8.1 Who writes, who reads

`ADR-0003:12` settles the rule: one owner, read-only references elsewhere. `ADR-0003:30` adds that invariants are enforced only in the owning module.

| Concern | Owner | Notes |
| :--- | :--- | :--- |
| `Owner`, `Ownership Share`, tax aliases, ownership audit | Owners Management | Writes and reads |
| Property facts (address, status, lifecycle) | Real Estate Management | Owners reads |
| Contract, Invoice, Income | Contract context | Owners reads none of them |

**Consequence that must be stated in the ADR set:** the ownership relation is written by exactly one operation family, and no other module holds a mutable copy. The tempting second representation — a `owner_id` or `owner_name` column on the property — is forbidden by I-6, because it is precisely the single-owner assumption that co-ownership breaks and it would become a second truth within one release.

### 8.2 The read contract, and the honest reading of "read-only references (foreign keys)"

`ADR-0003:12` speaks of *foreign keys*, a PostgreSQL construct, in a system with one database (`ADR-0003:8`). The honest reading is that **cross-module reads may join; cross-module writes may not.** A batched in-process port is the *stricter* choice, not the required one — which inverts the assumption a service-oriented instinct would make.

Two consequences:

1. **The portfolio can be one SQL query**, not an N+1 loop. At agency scale that is the difference between a 40 ms screen and a 1.5 s screen.
2. **Coupling to Real Estate's columns is the price.** That price must be paid once, in one named place, not scattered. Recommendation: a single `OwnerPortfolioQuery` in Owners owning the only join to Real Estate's table, protected by an integration test that fails if the joined columns change, and a repository rule that no other Owners class may name a Real Estate table (Section 11.6).

If the portfolio later needs to sort or filter by property attributes, the join is already there. A batched port would not allow server-side sorting by address without loading everything.

### 8.3 The canonical editor, and ADR-0003's revisit clause

ADR-0003 leaves one question genuinely open: `ADR-0003:32` — "Ownership Share is initially anchored under Owners Management; **revisit if property-side ownership editing becomes primary**", mirrored at `GLOSSARY.md:71`. Issue #3 answers it by silence, and the answer matters because it decides which package implements the write endpoint.

**Adopted resolution: one write command, two entry surfaces.** The `PUT` ownership operation is implemented by Owners Management code even when addressed by a property-keyed URL. The property screen's owner list is a read-only view with a "Gestionar titulares" action that deep-links into the allocation editor. This satisfies `ADR-0003:25` (Real Estate lists, does not mutate) and does not require touching the ADR.

**The clause must stop being unfalsifiable.** Recommendation: predefine its trigger and write it down — *the clause fires when more than half of share edits originate from the property screen for two consecutive quarters, or when a property-onboarding flow requires editing shares before the property exists.* Without a trigger, the clause is re-litigated informally in every review (governance action G-5).

### 8.4 Matrix corrections this module requires

Two glossary rows are inconsistent with ADR-0003 and with each other (Section 2, C-1 and C-2). Minimal correction:

| Row | Current | Required | Why |
| :--- | :--- | :--- | :--- |
| Real Estate Management — Reads | `Owner`, `Contract`, `Tenant`, `Document` | add `Ownership Share` | `ADR-0003:25` already says it lists its owners; without the relation grant it can only list one owner |
| Contract context — Reads | `Tenant`, `Real Estate`, `Contract Template`, `Document` | unchanged, **plus a note** that lessor data reaches Contract through its Real Estate read, snapshotted at signing | Avoids a new Contract→Owners edge on the money path |
| Owners Management — Reads | `Real Estate (of an owner)` | unchanged | Sufficient: the portfolio is a read of Real Estate |

### 8.5 API surface

```
POST   /api/v1/owners
GET    /api/v1/owners?q=&status=&cursor=&limit=
GET    /api/v1/owners/{ownerId}
PUT    /api/v1/owners/{ownerId}                      (If-Match: version)
POST   /api/v1/owners/{ownerId}/archive
POST   /api/v1/owners/{ownerId}/restore
GET    /api/v1/owners/{ownerId}/portfolio?cursor=&limit=
GET    /api/v1/owners/tax-identifiers/lookup?value=   (normalise, classify, report existing)

GET    /api/v1/property-ownership/{propertyId}
PUT    /api/v1/property-ownership/{propertyId}         (If-Match: allocation version)
```

- **`PUT` on the allocation, not `PATCH` per share.** Adding a second owner always passes through 150%; a per-share endpoint either disables validation mid-flight or rejects most calls, and no client can orchestrate a multi-request transaction with rollback on a phone.
- **The resource is keyed by property but served by Owners.** Document this in the OpenAPI description, or the next developer will assume Real Estate owns it and add the write dependency that ADR-0003 forbids.
- **`Owner` list carries `propertyCount`** so the list screen does not issue one query per row.
- **The portfolio returns references plus the share**, not full property payloads, unless the screen needs a field.

**Error contract** (`application/problem+json`, stable codes, machine-readable details — never prose alone):

| Code | HTTP | Detail | When |
| :--- | :--- | :--- | :--- |
| `OWNER_TAX_ID_EXISTS` | 409 | `existingOwnerId`, `legalName`, `archived` | Create/update collides on the normalised identifier |
| `OWNERSHIP_TOTAL_OVER` | 422 | `submittedUnits`, `persistedUnits`, `excess`, `totalUnits` | Allocation above 100 |
| `OWNERSHIP_VERSION_CONFLICT` | 412 | `current` allocation, `attemptedVersion` | `If-Match` mismatch |
| `PROPERTY_NOT_IN_ACCOUNT` | 404 | — | Unknown property, or one in another account |
| `OWNER_ARCHIVED` | 409 | — | Mutation on an archived owner |
| `SHARE_OWNER_ARCHIVED` | 422 | `ownerIds` | Assigning an archived owner to a property |

Returning the conflicting owner's id and name inside `OWNER_TAX_ID_EXISTS` is deliberate: it lets the client render a resolution panel without a second call, and within one account it discloses nothing the user cannot already see.

### 8.6 Concurrency, and why the obvious implementation corrupts data

**The write skew.** A property has one share at 90%. Two employees each add a 10% co-owner.

```
T1: SELECT sum(units) for property P            -> 9000
T1: validate 9000 + 1000 <= 10000               -> ok
T2: SELECT sum(units) for property P            -> 9000   (own snapshot)
T2: validate 9000 + 1000 <= 10000               -> ok
T1: INSERT share (owner B, 1000); COMMIT
T2: INSERT share (owner C, 1000); COMMIT
Final total: 11000
```

No constraint objects. A `CHECK` on a share row sees only that row; PostgreSQL cannot express a cross-row sum as a `CHECK`. `REPEATABLE READ` does not help: the two inserts are of different rows, so there is no write-write conflict and no serialization failure — this is write skew, detected only under `SERIALIZABLE`. A re-sum *inside* T2 after its own insert still sees 90 + 10 = 100, so a transaction-local assertion passes while the database is wrong. A deferred constraint trigger re-summing at commit is also insufficient unless the trigger itself takes a lock.

| Mechanism | Prevents it | Cost |
| :--- | :--- | :--- |
| **Optimistic version on the allocation row** (recommended) | Both read version 5; the second `UPDATE … WHERE version = 5` affects zero rows; the transaction aborts with 412 and the committed state | One column, one `INSERT … ON CONFLICT DO NOTHING` to materialise the row, a conflict UI |
| `SELECT … FOR UPDATE` on the allocation row | Locks the property's allocation for the transaction | Serialises all edits to one property; **`FOR UPDATE` on a missing row locks nothing** — the row must be materialised first, which is the trap |
| `SERIALIZABLE` + retry loop | SSI aborts one transaction | Needs a retry budget and a way to surface aborts as a domain error |

**Decision: optimistic version, because the write is a replace-all and therefore already reads current state.** The isolation level must be stated in the design rather than inherited from the framework default. The loser gets 412 with the current allocation in the body, so the UI can show a diff rather than a dead end. A retry loop must never silently re-apply an edit against a set the user never saw.

### 8.7 Foreign keys must carry the account

`ADR-0003:12` sanctions cross-module foreign keys; `ADR-0006:18` requires the account dimension in every index. A plain `property_id → real_estate.id` satisfies neither fully: it permits a share in account A to reference account B's property, a leak that survives every application-level scoping test because the write is scoped and the referent is not.

```
ownership_share (account_id, property_id) -> real_estate (account_id, id)
ownership_share (account_id, owner_id)    -> owner       (account_id, id)
```

This requires `UNIQUE (account_id, id)` on the Real Estate table — a cross-module migration dependency that must be scheduled explicitly, not assumed.

---

## 9. Dimension 3 — UX/UI

One responsive implementation on the existing React 19 PWA. "Mobile and desktop" is a viewport requirement, not two deliverables; ADR-0002 and `ARCHITECTURE.md:4` already fix one frontend and one design subsystem.

### 9.1 Navigation

| Surface | Proposal | Reasoning |
| :--- | :--- | :--- |
| Desktop | Top-level sidebar entry **Propietarios** | For an agency, the owner is a principal, not a sub-entity of a property |
| Mobile | Reachable from the overflow menu **and** a global search that indexes owners by name and tax identifier | A six-module product cannot give Owners a bottom tab without evicting a daily-use destination. Search, not browse, is the mobile affordance. |
| Property screen | Read-only owner list + "Gestionar titulares" | The primary lookup in an agency is property → owner ("whose flat is this, who authorises the repair") |

**Creation must start from the property too.** If an owner can only be created inside the Owners module, employees will create properties with no owner and never back-fill. The property flow offers "Asignar titular" and "Crear propietario", both writing through the same allocation command.

### 9.2 Creating an owner

**Owner Type is derived, not chosen.** DNI/NIE implies a natural person; CIF implies a legal entity. A segmented control is the direct cause of the "company asked for a first name" defect. Show the deduced type as a read-only conclusion ("Persona física — DNI") with one manual override for "Pasaporte / sin NIF".

**On a mid-form type change, never clear fields.** Carry the typed text into the surviving field and let the user split it by hand; do not parse a company name into a first name. Guarantee a server-computed `legalName` on every representation so no client re-implements the formatting rule.

### 9.3 Tax identifier entry

- One field labelled **NIF / CIF**, `inputmode="text"`, uppercase transform, autocorrect and spellcheck off, stripping spaces, dots and hyphens. `inputmode="numeric"` cannot work: a DNI ends in a letter and a NIE starts with one.
- Validate **on blur and on submit**, never per keystroke — the check letter is invalid for every prefix of a valid DNI, so live validation paints the field red from the second character.
- Never disable the submit button on a check-character failure; explain it.
- Three distinct Spanish messages: format/length, "La letra no corresponde a los números", "El dígito de control no coincide". A generic "NIF no válido" is not actionable.
- **The check character is validated, not memorised.** A user cannot know whether their CIF's control character is a letter or a digit; any message asking them to choose is unusable.
- Normalise **server-side as well**, because imports and any future Idealista path will not pass through this form.

### 9.4 Duplicate owner

Duplicate creation is the expected outcome when two colleagues onboard the same landlord. A 409 that tears down the form is a data-loss flow, not an error state.

Flow: keep the form and every typed field; show an inline panel naming the existing owner with `Abrir`, `Reactivar` (when the match is archived), and `Corregir el NIF`. Never auto-merge, never silently rewrite the identifier, never clear the form.

Because uniqueness spans archived owners (Section 10.2), the lookup must too — otherwise an employee who cannot find an archived landlord in the picker creates the duplicate the feature exists to prevent.

### 9.5 Co-ownership editor

A bottom sheet on mobile, a modal on desktop, opened from the property's "Titularidad" section:

1. A stacked bar showing allocated percentage, with the unallocated remainder in grey.
2. One row per owner: name, percentage field, remove action.
3. A total line that *describes* rather than accuses: "78 % asignado · 22 % sin asignar". An incomplete total is not an error and must not be styled as one.
4. Actions: `Añadir propietario`, `Dividir a partes iguales`, `Guardar`.
5. The last row is a computed, read-only remainder after a split, so a user can never type themselves into 100,03 %.

**The 7-heir case is the design driver, not an edge case.** Seven inputs cannot sum to 100 at two decimals by hand; the split action and the remainder row are what make the arithmetic disappear. For a single owner, do not render a percentage field at all — default to 100 and let the user change it only by adding a co-owner.

### 9.6 Portfolio

Search-first, locale-aware sort by `legalName`, sticky filter chips including **"Mostrando archivados"** and **"Sin inmuebles"**. Archived owners must be findable, or they become duplicates.

- An owner with no properties is a normal row with a primary action, not an error.
- An unknown share renders **"Sin datos"**, never "0 %".
- A property whose titling is `Partial` carries a chip, so the data-quality gap is visible where work happens.
- Keep query, filters and scroll in the URL so PWA back navigation does not reset the list.

### 9.7 Offline

The PWA caches an app shell; that is not an offline data store.

| Action | Offline |
| :--- | :--- |
| Read owner / portfolio | Allowed, with explicit freshness ("Actualizado hace 2 h · sin conexión") |
| Create owner | Draft with a client-generated id, "Pendiente de sincronizar" badge, idempotent replay, cannot be assigned as a shareholder until accepted |
| Edit owner | Blocked or queued to the same conflict queue |
| **Edit ownership** | **Online only.** A roster has no correct last-write-wins merge: two devices can each produce a valid 100 % roster describing different ownership. |

---

## 10. Dimension 4 — Edge cases and lifecycle

### 10.1 Owner with zero properties

The default state of a newly created owner, not an edge case — and it currently contradicts `GLOSSARY.md:16`. Resolution: edit the definition to "zero or more" and treat zero as normal. `GET /portfolio` returns `200` with `items: []`; `404` would force every client to distinguish empty from absent, and they will get it wrong.

### 10.2 Duplicate tax identifiers

- **Uniqueness is a partial unique index on `(account_id, normalised_value)` over active owners**, not service logic. Service checks race and re-implement a constraint the database already offers.
- **Uniqueness spans archived owners.** An archived owner still *is* that person; a second live row creates two identities for one taxpayer. The correct action for "we archived them by mistake" is restore, not re-create — which is why the duplicate panel offers `Reactivar`.
- **The alternative predicate is worse.** `WHERE archived_at IS NULL` looks friendlier but makes restore fail at runtime with no resolution path, and leaves two rows for one person.
- **Same person, two accounts, is legitimate** — an agency and the owner's own holding company may both manage them. The constraint is account-scoped by design; a global unique index would be both wrong and a cross-tenant existence oracle (Section 11.5).
- **Fuzzy name matching is out of scope.** Spanish two-surname names produce false positives and trigram search is an expensive write-path operation.

### 10.3 Archival, deletion, and the unimplementable requirement

The issue asks to archive "when the owner has historical contracts or invoices". Read literally, Owners Management cannot implement it: the glossary grants Owners a read of `Real Estate (of an owner)` only (`GLOSSARY.md:48`), and Contract/Invoice belong to the Contract context (`GLOSSARY.md:52`). Options are to add an undeclared read, or to make another module the gate on an Owners lifecycle action — the second is worse, because a write-time cross-module coupling is invisible in the ownership matrix.

**Adopted resolution — invert the rule.** Owners archives freely and unconditionally; the consumer refuses. Contract creation refuses an archived owner; existing contracts keep working. This keeps the check where the data is, needs no new read grant, and is the only version that survives the current module map.

Supporting rules:

- **Archival never changes title.** Shares remain; the property roster renders the owner as "archivado (histórico)"; portfolio totals are unchanged because the person still legally owns.
- **No hard delete.** Not for owners with shares, not for owners with history, and in v1 not at all. `DELETE` is not exposed. This removes the entire "do you have history?" cross-module question from the design.
- **Archival needs a reason and an actor.** Note the gap: ADR-0006 defines accounts and never users, so "who archived this" has no referent yet. Until an identity model exists, the actor is a display string and the audit is honest about it (Section 10.6).
- **`Archivar` is the label, never `Eliminar`.** Confirmation names the consequence: "El propietario dejará de aparecer en las listas de trabajo. Sus inmuebles y su historial no cambian."

### 10.4 Point-in-time ownership

The question "who owned this flat when the March 2027 invoice was issued?" is real, and answering it from a current-state table is impossible after the fact.

**Resolution: the Contract snapshots its lessor side at signing.** ADR-0004 already treats renewal as a new Contract and never mutates a signed one (`ADR-0004:18`), so the seam exists. Contract obtains lessor data through its existing Real Estate read — no new Contract→Owners edge — and freezes the names and shares it used. Ownership history then lives in the documents that needed it, and the live ownership table stays a simple current-state structure.

Consequences to state honestly:

- A contract cannot show "the current owners" after a sale. If the product needs that, effective dating becomes mandatory and this decision reverses (Section 15, D-06).
- The ownership audit table keeps the option open: because it records before/after allocations from day one, a later temporal model can be *derived* rather than invented — provided the audit table starts with the first share row.
- This requires the Contract Template's singular "owner name" merge field to become a list (Section 2, C-2).

### 10.5 Identity evidence has no home

`ADR-0005:12` enumerates Document Store owning records as Tenant, Candidate, Contract. **Owner is absent**, and CIF is not a document type anywhere (`GLOSSARY.md:33` lists DNI/NIE and Passport). A module whose job is legal identity cannot attach the document that proves it, and two authorities for the same fact will diverge: the typed tax ID and the scanned DNI.

This is a compliance gap, not tidiness. It needs a new ADR extending ADR-0005 to Owner (with consent and retention rules), and a decision about which of the typed identifier or the document is authoritative on mismatch. **v1 stores typed identifiers only and records the gap**; no owner ID document upload is built until that ADR exists.

### 10.6 Auditability

One append-only table, written in the same transaction as the change:

```
ownership_audit
  id, account_id, property_id (nullable), owner_id (nullable),
  actor_ref, occurred_at (UTC), request_id, client_build, route_template,
  change_kind  (SHARES_REPLACED | OWNER_CREATED | OWNER_UPDATED | OWNER_ARCHIVED
                | OWNER_RESTORED | TAX_IDENTIFIER_CHANGED | DATA_REPAIR),
  attempted_version, committed_version,
  before (jsonb), after (jsonb), note
```

Explicitly not a generic audit framework and not event sourcing: the observed need is "who changed what, from what to what", and no ADR in this project defines platform-wide audit. Indexed **on both axes** — `(account_id, property_id, occurred_at)` and `(account_id, owner_id, occurred_at)` — because the first support question is "who touched this flat", and an owner-only index makes it a scan.

Four properties required, because an audit trail that silently stops writing is worse than none:

1. **Same transaction as the mutation.** An asynchronous writer makes a gap indistinguishable from "nothing changed". For ownership changes, fail closed.
2. **Freshness alerting** on `audit_write_failures` and on a gap between mutation and audit time.
3. **A support-readable timeline**, read-only, by property, by owner, by time. A table only a DBA can query does not answer a ticket in five minutes.
4. **Raw payload gated.** The `before`/`after` JSONB may contain a legal name; reading it is a second, itself-audited permission.

The 3 AM question this is built for: *"the app shows my flat with the wrong owner."* The engineer opens the property timeline, sees the last `SHARES_REPLACED`, its actor, time, before/after units, and the client build — and can tell within a minute whether this was a legitimate transfer, a stale client, or a bug.

### 10.7 Observability and repair

**Reason codes, not error rates.** A single "share write failure rate" merges user error with a rounding bug, a race, and a stale client:

`user_over_allocation · version_conflict · stale_client · duplicate_tax_id · cross_account_denial · validation · unknown`

**No unbounded labels.** Never label a metric with `account_id`, `owner_id`, `property_id`, or a tax identifier — cardinality explosion plus tenant data in a system with weaker access control than the database.

| Signal | Action |
| :--- | :--- |
| Any **successful** cross-account read | Page immediately |
| Audit write failure or audit lag | Page |
| A known-complete property whose stored total drifts from 100 | Page (should be impossible) |
| Portfolio p95 / response bytes over budget | Page on error-budget burn only |
| `Partial` / `Unknown` allocation count per account | Dashboard only — it is an expected state, and paging on it trains responders to ignore alerts |
| Duplicate-identifier rejections | Dashboard only |

Cross-account **denied** attempts and successful cross-account reads must be separate events: a stale bookmark is a benign 404, a successful read is a breach.

**Drift is a representation problem, not a monitoring problem.** With integer units a total of 99,999999 % is impossible by construction. The residual risk is a legacy import or a manual repair, so: a scheduled invariant query counts properties whose stored units do not equal `total_units` **without auto-repairing**; repair is break-glass, locks the allocation, records `before`/`after` with `change_kind = DATA_REPAIR`, bumps the version, requires a reason, and notifies affected owners. Silent repair destroys the evidence needed to explain an old statement.

**Personal data.** Never log or trace a tax identifier, legal name, address, phone, email, or free-text note — not in application logs, span attributes, metric labels, session replay, or URL query strings (access logs see URLs). Do not echo field values in validation messages: `"DNI 12345678Z no válido"` is a leak; `"la letra no corresponde a los números"` is not. Disable SQL parameter logging where logs leave the host. Use the internal `owner_id` UUID as the debugging key; resolving it to a person is a separate, permissioned action.

### 10.8 Cross-boundary deletion

| Event | Rule |
| :--- | :--- |
| A property is deleted in Real Estate | Refused physically. Both aggregates carry legal or financial history, so both are archived, never deleted. The FK is `RESTRICT` as a safety net, not as a workflow. |
| A property is archived | The allocation stays; the portfolio shows it as retired |
| An owner is archived | No effect on shares |
| An owner is hard-deleted | Not possible in v1 |

`ON DELETE CASCADE` here would silently erase who owned what and would look, in review, like housekeeping. `RESTRICT` as the *default* would let Owners veto Real Estate's lifecycle. The rule that avoids both is the account-wide "archive, never delete" policy, which is recorded as governance action G-3.

### 10.9 Migration posture

There is no schema in this repository, so there is no legacy data to transform *here*. Two caveats:

1. `ARCHITECTURE.md:8-10` names `aap-backend` and `aap-frontend` as subsystems that are absent from this repository. If an implementation exists elsewhere, the dirty-data census (duplicate identifiers, blank identifiers, properties carrying a legacy owner text) must run there before the unique index is declared.
2. The installable PWA means old clients keep calling the API for weeks. The dangerous window is between "new clients write shares" and "old clients stop writing": if a legacy owner field is still writable, an old cached shell can silently replace a co-ownership set with a single name, and that loss appears in no new-path audit. **Freeze legacy writes with an upgrade prompt** rather than letting last-write-wins decide. This requires a client build header on every request, which does not exist yet.

---

## 11. Dimension 5 — Test strategy

No build, CI, or test configuration exists in this repository (`AI-Coding.SDD.md:28` names JUnit 5, integration and contract tests, JaCoCo and PIT as the intended stack; none is present). The seams below are therefore a commitment, not a description.

### 11.1 Seams

| Seam | Proves | Tooling | Determinism |
| :--- | :--- | :--- | :--- |
| **Pure unit** | Tax-identifier format, checksum, normalisation; allocation arithmetic and largest-remainder; titling-status derivation; archive transitions with the contract count injected | Plain JUnit, no Quarkus, no DB | Deterministic |
| **Database / integration** | Partial unique index, composite account-scoped FKs, `units` bounds, optimistic version, FK `RESTRICT`, `NULL` vs `0` semantics | Testcontainers PostgreSQL — **not H2**: partial indexes, composite FKs and constraint semantics differ | Deterministic |
| **Concurrency** | Exactly one 412, never a 110 % total | Two independent connections, barrier after the first `SELECT`, explicit commits | Deterministic because the conflict is version-driven |
| **REST contract** | Status codes, problem+json bodies, percentage wire encoding, `items: []` for an empty portfolio, 404-not-403 | Quarkus test + OpenAPI diff gate | Deterministic |
| **Tenant isolation** | The matrix in 11.4, on ≥ 2 seeded accounts | Integration, real PostgreSQL | Deterministic |
| **Module boundary** | Owners imports no Real Estate internals; no Owners class names a Real Estate table except `OwnerPortfolioQuery` | ArchUnit + a query-string allowlist check | Deterministic |
| **Browser** | Split-equally produces exactly 100 %; remainder row read-only; duplicate panel keeps form state; NIF keyboard; mobile viewport | Playwright | Behaviour-scoped, not pixel-scoped |

### 11.2 Test patterns that must be forbidden

1. **Thread-race tests without barriers.** "N threads add 10 % and assert 100 %" exercises the retry loop, not the invariant, and fails intermittently until someone disables it.
2. **No negative control.** Each concurrency mechanism needs a test that deliberately removes the version predicate and asserts the invariant *does* break. Without it nobody knows the test can fail.
3. **Tests inside one rolled-back transaction.** They hold a single connection, so commit-time behaviour never runs and the race is invisible — the suite is green and production corrupts.
4. **Mocked repositories for constraint behaviour.** A mock cannot prove a partial unique index works.
5. **H2.** It will not reproduce partial indexes, composite FK behaviour, or `NULL` ordering.

### 11.3 Fixtures

| Fixture | Asserts |
| :--- | :--- |
| Two accounts, each with owners | Every isolation test seeds both |
| P1: one natural person 50 % + one legal entity 50 % | Both identifier types, `TitlingStatus.Complete` |
| P2: three owners 33,33 / 33,33 / 33,34 in units (3333/3333/3334) | The residue rule, and that display percentages sum to 100,00 |
| P3: seven heirs | Largest-remainder across many rows |
| P4: one share with `units = NULL` | Unknown is storable and distinguishable from 0 |
| P5: one share with `units = 0` | A genuine zero stake is not "unknown" |
| P6: property with no shares | `Unassigned`, portfolio returns `200 []` |
| P7: 90 % allocated, 10 % free | The concurrency fixture |
| O0: owner with zero properties | Empty portfolio, not 404 |
| Oarch: archived owner holding a share | Archival leaves the total unchanged |
| Same tax identifier in both accounts | Allowed; a 409 here is a tenancy defect |
| **Rejection set** (never seed data): 120 % total, share −1, share 10001 units, duplicate `(owner, property)`, duplicate identifier in one account, bad DNI letter, share attached to an archived owner, share in A on B's property | Each must fail, at the stated layer |

Valid fixtures are built through a factory that asserts validity; invalid ones are written as SQL only inside rejection tests, so seeded data can never encode a state production forbids.

### 11.4 Tenant isolation: what a 404 test misses

The most likely leak in this module is not a response containing a foreign row — it is the duplicate check becoming an existence oracle.

- Same normalised identifier created in A **and** in B must both succeed. A 409 leaks that agency B manages that DNI.
- Same identifier twice in A returns 409 with the existing owner named.
- Every Owners query method invoked as A returns only A's ids — enumerate the methods so a new method without the filter fails the build.
- IDOR matrix over `GET`, `PUT`, `archive`, `restore`, `portfolio` across accounts → 404 and **no state change**.
- Share creation against B's property is rejected at the **database** level, not only in the service.
- `account_id` supplied in body or header while authenticated as A is ignored.

### 11.5 Module boundary: what a build rule proves

An ArchUnit rule can assert that Owners production classes do not import `realestate.internal` / `..persistence` / `..domain`, with an allowlist for the read contract. It cannot see a JPQL or native query naming the table, a table name in a string, a second datasource, or an entity flushed from a shared package.

So the rule proves **dependency direction**, not data ownership. Three layers, each labelled honestly:

1. ArchUnit import rule (necessary, cheap).
2. A CI check that no Owners class outside `OwnerPortfolioQuery` contains a Real Estate table name in a string or query file.
3. If per-module database roles are ever possible, an integration test asserting that an `UPDATE` from the Owners role fails. No ADR specifies such roles today (`ADR-0006:12` puts isolation in the data-access layer), so this layer is currently unavailable and recorded as an accepted risk rather than claimed.

### 11.6 Spanish tax identifiers: tested versus cited

Test table: valid DNI; wrong check letter; 7 and 9 digits; leading zero; lowercase; spaces, dots and hyphens; NIE with X/Y/Z each valid plus an invalid prefix; legal-entity identifier with a valid and an invalid control character **per letter group**; wrong length; a passport; a blank identifier; normalisation idempotence; and `12345678-Z` colliding with `12345678z`.

**Pinned to a citation, not recalled:** the exact control-character algorithm and the letter-group table for legal entities, and whether the legacy algorithm must still be accepted for existing records. Cite the tax authority's published NIF check specification, with document and date, beside the table, and take the vectors from official examples. A third-party reference implementation is a cross-check, not the authority. A recalled checksum produces a validator that rejects valid owners and accepts invalid ones, with a test suite that agrees with the bug.

### 11.7 Acceptance criteria must be observable

| Dimension | Verifiable criterion |
| :--- | :--- |
| Data model | Three shares of 3333/3333/3334 units sum to exactly 10000 and report `Complete` |
| Data model | A share with `units = NULL` renders "Sin datos" and is excluded from the known total |
| API | `PUT /property-ownership/{id}` without `If-Match` returns 428; stale version returns 412 with the committed allocation |
| API | Adding a share that would reach 110 % returns 422 with `submittedUnits`, `persistedUnits`, `excess` |
| Isolation | Creating an owner with an identifier that exists only in another account returns **201** |
| Isolation | Attaching a share to another account's property fails at the database constraint |
| UX | Typing three equal shares then `Dividir a partes iguales` yields exactly 100 % with no error banner |
| UX | Submitting a duplicate NIF keeps every field populated and names the existing owner |
| Lifecycle | Archiving an owner with two shares leaves the property total unchanged and writes one `OWNER_ARCHIVED` row |
| Tests | Removing the version predicate makes at least one concurrency test fail |

Anything that cannot be written in that form is a decision, not a criterion — which is what Section 15 is for.

---

## 12. Options

Three coherent end-to-end options, not three variants of one decision. Each is a package across all five dimensions, because the choices interact: the storage unit constrains the API, which constrains the UX, which constrains the tests.

### 12.1 Option A — Current-state register (glossary-literal, minimal)

**Shape.** One `Owner` table and one `ownership_share` table with `numeric(5,2)` percentage, `0` meaning unknown exactly as `GLOSSARY.md:18` says. Per-share `POST`/`PATCH`/`DELETE`. Server validates `sum ≤ 100` before each write, best effort. Portfolio is a direct SQL join. Lifecycle is `active`/`archived` with `updated_at`/`updated_by`. No allocation aggregate, no version, no audit table, no `NULL`-vs-`0` distinction.

**Assumptions.** Ownership is administrative bookkeeping, not a record anyone will be held to. Co-ownership is at most two owners, always at whole percentages. No consumer will need a sum. Only one person edits a property at a time.

**Trade-offs.** Cheapest to build and to explain; three tables, no state machine, no concurrency token. It is also fully compliant with the glossary *as literally written*, which is not a small virtue. It buys that simplicity by making the two hardest facts unrepresentable: a property's completeness and a safe concurrent edit.

**Failure modes.** (i) Two employees add 10 % each and the property reaches 110 % with no error and no record — the write skew in Section 8.6, which no constraint catches. (ii) "Unknown" and "zero" are the same value, so a data-quality report cannot be built and a portfolio can claim 100 % coverage that does not exist. (iii) Three equal owners cannot be saved without one of them owning 33,34 %. (iv) There is no way to explain a change after a dispute. (v) Adding a second write surface later has no version token to conflict on.

**Reversibility.** High for the schema (a migration), **low for the data**: once percentages have been written by hand with a residue hack, migrating to exact units requires deciding what the user meant.

**Operational cost.** Lowest: three tables, no jobs, no audit volume, no reconciliation.

**Disqualifying conditions.** Qualifies as disqualified when any of these hold: the agency ever edits one property concurrently (it will); real co-ownership of three or more owners appears; anyone asks "is this property's titling complete?"; a consumer needs a sum; or `usufruct`/bare-ownership separation must eventually be represented.

---

### 12.2 Option B — Allocation aggregate with explicit unknown state and locked writes **(recommended)**

**Shape.** `Owner` and `OwnershipAllocation` as two aggregates (Section 7.1). Integer units, `NULL` = unknown, `0` = genuine zero. `TitlingStatus` derived. One write path: `PUT /property-ownership/{propertyId}` replacing the whole set under an optimistic version. Composite account-scoped foreign keys. Append-only audit from the first row, indexed on both property and owner. Portfolio is one join with `propertyCount` denormalised onto the owner list. Archive/restore only, no hard delete. Contract snapshots its lessor side.

**Assumptions.** The agency really does onboard properties before the titling is known — which `GLOSSARY.md:18` already asserts by defining an unknown state. Contract can snapshot its lessor via its existing Real Estate read without a new module edge. The team will write four glossary edits and two new ADRs before freezing the schema. Two employees editing one property is plausible.

**Trade-offs.** More machinery: two aggregates, a derived status, an audit table, a version column, four glossary edits. In exchange the module can state what it knows, refuse what is incoherent, and explain what changed. The costs are concentrated in *decisions that must be made now* rather than in ongoing operational load — which is the right place for cost in a greenfield design.

**Failure modes.** (i) **The unknown-state ambiguity survives** if `NULL` and `0` are conflated in the UI: a user sees "0 %" and concludes the owner has no stake. Mitigation: render "Sin datos" and keep the two states distinct in every DTO. (ii) **The residue confuses someone.** 14,29 / 14,29 / 14,28 looks like a bug; mitigation: show units on demand and always offer the split action. (iii) **Conflict friction.** 412 with no diff is a dead end; mitigation: return the committed allocation in the body. (iv) **Titleing status accumulates unseen.** Hundreds of `Partial` properties nobody fixes; mitigation: a per-account data-quality gauge and a worklist. (v) **The composite FK dependency is missed** and the tenant guarantee silently degrades to application-level; mitigation: schedule Real Estate's `UNIQUE (account_id, id)` as an explicit cross-module migration.

**Reversibility.** High for the aggregate and the API (additive), **medium** for integer units (one migration, but before any consumer computes money from them), **low** for the contract-snapshot decision once documents are issued.

**Operational cost.** Medium: one audit table with two indexes, one nightly drift query, one data-quality gauge, one portfolio budget. No event bus, no read model, no queue, no cron repair.

**Disqualifying conditions.** If the product confirms that ownership is display-only forever and that no one will ever ask whether a titling is complete, Option A's simplicity wins and B's machinery is ceremony. If contract snapshots are rejected (the product must show current owners on a live contract), then effective dating becomes mandatory and B's data model changes materially — see D-06.

---

### 12.3 Option C — Re-anchor ownership to Real Estate

**Shape.** `Ownership Share` moves under Real Estate Management — or into a dedicated Titling context, following the precedent ADR-0004 set for Contract. The property table gains the relation; the property screen owns the edit; Owners becomes owner CRUD plus a portfolio that needs no cross-module join.

**Assumptions.** Property-side editing is or will become the primary workflow, which is exactly the condition `ADR-0003:32` names. One team owns both modules, so coordination cost is zero.

**Note on legitimacy.** This is **not** a violation to be rejected out of hand. `ADR-0003:32` and `GLOSSARY.md:71` both pre-authorise revisiting the anchor, and ADR-0004 set the precedent that a relation spanning two modules can get its own context rather than an arbitrary anchor. C is viable *with a new ADR that supersedes 0003's anchor line* — ADR-0001 requires a new record, not an edit.

**Trade-offs.** Best query shape (no cross-module join, no read-grant question), one writer by construction, and the edit surface where the work happens. Costs: every consumer of ownership now depends on Real Estate, making it a foundational context; the Owners module shrinks to identity plus a derived portfolio; and the module that the glossary names as the source of truth no longer is.

**Failure modes.** (i) **Contracts begin reading ownership directly** from Real Estate, so an ownership change needs a coordinated release. (ii) **The relation grows property-shaped** — rent shares, expense split factors, statement recipients — because that is the convenient place, and Owners never recovers. (iii) **Owners becomes vestigial**, and "owner with zero properties" becomes a cross-module query anyway. (iv) **A second re-anchor** when owner statements arrive, which is the scenario the clause was written to permit.

**Reversibility.** Low. Once Contract and Invoice depend on the property-side relation, moving it back is a schema migration plus a consumer migration under production documents.

**Operational cost.** Lower day one, higher on every later release: each ownership change becomes a multi-module coordination event.

**Disqualifying conditions.** Disqualified unless a new ADR supersedes the anchor. It becomes the *better* option if the trigger in Section 8.3 fires — property-first editing dominates — in which case C is not a deviation but the clause executing as designed.

---

### 12.4 A fourth option that must be named and rejected

**"Ownership exists so rent can be split between co-owners."** This is the assumption most likely to enter the room unexamined, and it is not supported: `Income` derives from active Contracts (`GLOSSARY.md:30`), `Invoice` is a tenant obligation (`GLOSSARY.md:29`), and no artifact consumes a share percentage. Building for it would import an owner-statement model, a money-precision decision (`GLOSSARY.md:70` is still open), and a settlement flow into an issue about identity and titling.

**Rejection:** satisfy the need when it exists, as its own issue and ADR. Until then the percentage is a *record*, not a *calculation input*. Stating this explicitly is what prevents the module from growing a billing subsystem by accretion.

---

## 13. Comparison

| Criterion | A — Current-state register | B — Allocation aggregate | C — Re-anchored to Real Estate |
| :--- | :--- | :--- | :--- |
| Glossary compatibility today | **Literal** | Needs 4 edits | Needs anchor ADR |
| ADR-0003 compliance today | Compliant | **Compliant** | Requires superseding ADR |
| Co-ownership > 2 owners | Broken (residue) | **Supported** | Supported |
| Unknown vs zero | Collapsed | **Distinguished** | Distinguishable if designed |
| Concurrent edits | Silent 110 % | **412 with current state** | Silent unless added |
| "Is this titling complete?" | Unanswerable | **Derived status** | Answerable |
| Portfolio query | One join | **One join** | One join, no cross-module |
| Cross-module read grant | Needs the same fix | **Needs the same fix** | Not needed |
| Audit / support explanation | `updated_at` only | **Append-only, two indexes** | Inside Real Estate |
| Schema work | Smallest | **Small** | Small, plus an ADR |
| Decisions required before code | Few | **Eleven (Section 15)** | One, but a governance one |
| Operational load | Lowest | **Low–medium** | Low now, high later |
| Reversibility | High (schema), low (data) | **High, medium for units** | Low |

---

## 14. Recommendation

**Adopt Option B**, delivered in three slices (Section 18).

1. **Option A contradicts the glossary's own stated behaviour.** `GLOSSARY.md:18` documents an unknown state. A hard `= 100` rule makes that state unstorable; a permissive `≤ 100` rule in A has no way to distinguish unknown from zero and no way to prevent the write skew that produces 110 % under ordinary concurrent use. A is only coherent if ownership is bookkeeping nobody checks — and the issue exists precisely because the agency wants to check it.
2. **Option C is legitimate but premature.** It is the right answer if the trigger in Section 8.3 fires, and the correct response then is a new ADR, not a quiet schema choice. Adopting it now would pre-empt the clause rather than execute it, and would make Real Estate the foundational context for money before a single consumer exists.
3. **The expensive parts of B are decisions, not code.** Eleven decisions and four glossary edits cost review time now; getting them wrong costs a data migration later. That is the right trade in a greenfield repository with no schema.
4. **The genuinely deferrable work is deferrable.** Merge, effective dating, rights types, owner documents, bulk import, and the read-model projection all wait, each with a named trigger in Section 15.

**What would change the recommendation**

- Written confirmation that ownership is display-only forever and nobody will ask whether a titling is complete. Then **A**, and B's machinery is ceremony.
- A product decision that a live contract must display *current* owners after a sale. Then contract snapshots fail, effective dating becomes mandatory, and B's model changes materially (D-06).
- The Section 8.3 trigger firing. Then **C**, via a superseding ADR.

**Where the recommendation is weakest, stated plainly:** B's `NULL`-for-unknown is a deliberate deviation from the literal "0 when unknown" wording. It is better modelling, and it costs a glossary edit plus an argument with anyone who reads the original sentence. If the team declines the edit, the fallback (non-null `0`, ambiguity accepted) still works for I-3 and I-5 but silently loses I-4 — and with it the data-quality reporting that justifies half of B.

---

## 15. Decision register

Eleven decisions must be closed before schema freeze. Each has a recommendation, an owner, reversibility, and the trigger that reopens it.

| ID | Decision | Recommendation | Reversibility | Reopen trigger |
| :--- | :--- | :--- | :--- | :--- |
| D-01 | Storage unit for a share | Integer units, `total_units = 10000`, percent rendered | **One-way** after any consumer | Glossary demands a different denominator |
| D-02 | Meaning of "unknown" | `units NULL`; API renders `percentage: 0, known: false` | Medium (glossary edit) | Team insists on the literal sentinel → accept I-4 loss |
| D-03 | `= 100` enforcement | Derived `TitlingStatus`; no save-time block; gate only at a future consumer | High | A consumer appears |
| D-04 | `> 100` enforcement | Hard reject, always | High | Never |
| D-05 | Aggregate root | `OwnershipAllocation` per `(account_id, property_id)` | Low cost early | Performance shows contention |
| D-06 | Point-in-time ownership | Contract-side lessor snapshot; no temporal columns | **One-way** once audits lag or documents issue | Live contracts must show current owners |
| D-07 | Hard delete | Not exposed; archive only | Medium | A confirmed legal erasure obligation |
| D-08 | Tax-identifier uniqueness | Partial unique `(account_id, normalised)` over active; spans archived | Medium (data cleanup) | Import demands relax it → needs merge first |
| D-09 | Identity across roles | Role-local; no Party context | **One-way** once tenants accumulate | Tenant and owner are the same person in practice |
| D-10 | Write concurrency | Optimistic version on the allocation row; isolation level stated | Medium | Contention proves a pessimistic lock better |
| D-11 | Portfolio read path | One named SQL join owned by Owners | Medium | Needs property-attribute sorting → still fine; needs extraction → port |

Also required before code: **D-12** whether Owner becomes a Document Store owning record (blocks any ID upload), and **D-13** who the audit actor is, given ADR-0006 defines no users.

---

## 16. Goldfish review synthesis

Six reviewers, six lenses, fresh contexts, each reading the actual glossary and ADRs. Deduplicated by theme; the lens that raised each first is named.

### 16.1 Themes

**1. The glossary has already decided more than the issue admits — and the issue contradicts two of those decisions.** *(contrarian, structural, inventive)*
`GLOSSARY.md:18` documents unknown ownership; `ADR-0003:12` sanctions cross-module foreign keys; `ADR-0003:32` pre-authorises re-anchoring. Meanwhile the issue's strict-100 framing and its owner-side editing both cut against `GLOSSARY.md:18` and against the dormant revisit clause. The reviewers' sharpest shared point: the issue presents a design, while SDD Prompt 2 requires an option comparison — and the design it presents already forecloses two accepted decisions.

**2. The invariant `= 100` is unstorable, and `≤ 100` needs one locked place.** *(readiness, structural)*
Both reviewers independently derived the same write skew and the same conclusion: `REPEATABLE READ` does not prevent it, a deferred trigger does not prevent it unless it takes a lock, and PostgreSQL cannot express a cross-row `CHECK`. The readiness lens added the trap that matters in implementation: `SELECT … FOR UPDATE` on a row that does not exist locks nothing. This is the review's most concretely useful finding and it is now Section 8.6.

**3. "Real Estate lists its owners" is not backed by the matrix.** *(structural)*
The single sharpest citation in the whole review: `ADR-0003:25` says Real Estate lists owners, but `GLOSSARY.md:49` grants Real Estate a read of **Owner** only — not **Ownership Share**. Co-ownership therefore cannot be served from any granted read, and the alternative is a property-level single-owner pointer that will drift. This became Section 8.4.

**4. The percentage has no consumer, so the invariant has no justification beyond data quality.** *(contrarian, ops)*
The contrarian traced every candidate consumer — Dashboard, Contract, Real Estate, Income — and found none that divides by a share. That is why this document refuses to invent a billing gate and places the enforcement point at a consumer that does not yet exist. It is also why Section 12.4 exists: to name and kill the "ownership is for splitting rent" assumption before it enters the room.

**5. Unknown-as-zero is the defect that outlives the schema.** *(inventive, ux, structural)*
Three lenses converged: a total computed over a sentinel cannot distinguish "complete" from "one owner never entered", and the portfolio will report 100 % coverage that does not exist. The inventive lens reframed it as the contradiction "absence is not a number", with the resolution being a state rather than a value — adopted as D-02 with the cost named honestly.

**6. Archival is unimplementable as specified, and the fix is inversion.** *(contrarian, structural, readiness)*
All three found that "archive when the owner has historical contracts or invoices" requires a read Owners is not granted. Two of them independently proposed the same inversion: archive freely in Owners, refuse at the consumer. That is now Section 10.3, and it needs no new grant.

**7. The Document Store cannot hold the evidence this module is about.** *(structural, inventive, contrarian)*
`ADR-0005:12` lists Tenant, Candidate, Contract — not Owner. CIF is not a document type anywhere. A module whose purpose is legal identity stores a typed string with no document behind it, and the compliance question is not architectural. Raised as D-12 and G-4 rather than papered over.

**8. Composite foreign keys are the only real tenant guarantee on the share table.** *(readiness, structural, ops)*
`ADR-0006:18` requires the account in every index; a plain `property_id` FK permits a share in account A pointing at account B's property, and that survives every application-level scoping test. All three reviewers reached this independently from different directions, which is why it is an invariant (I-1) and not a note.

**9. The duplicate-check is the most likely cross-tenant leak.** *(readiness, ux)*
An account-scoped uniqueness rule implemented without the account predicate, or a global unique index, turns owner creation into an existence oracle: a 409 tells agency A that agency B manages that DNI. A 404-only isolation suite never exercises it. Now Section 11.4.

**10. Audit is the difference between a five-minute answer and no answer.** *(ops, structural)*
The ops lens supplied the operational detail the design was missing: index the audit on the property axis as well as the owner axis, because the first support question is "who touched this flat"; fail closed for ownership changes; keep PII out of telemetry including URL query strings.

**11. The PWA's old clients are an ownership-data hazard, not a caching nuisance.** *(ops, migration)*
The dangerous window is the one between new clients writing shares and old clients still writing a legacy single-owner field: an old cached shell can replace a co-ownership set with one name, and the loss appears in no new-path audit. Freeze legacy writes with an upgrade prompt; require a client build header. Neither mechanism exists today.

**12. Scope discipline, applied concretely.** *(contrarian, readiness)*
The contrarian's structural case — four of six requirements are already ratified by the glossary and cheap; dual-device UX, unbounded audit, and the contract/invoice archival rationale are additions without evidence — is reflected in Section 4's exclusion table and in the three-slice staging. The readiness lens added what can actually be demonstrated as shipped versus what must be declared deferred.

### 16.2 Findings adopted as design changes

| # | Change | Source lens | Where |
| :--- | :--- | :--- | :--- |
| 1 | Integer units; percent rendered only | ops, ux, structural, readiness | 7.2 |
| 2 | `OwnershipAllocation` as aggregate root | structural, readiness | 7.1 |
| 3 | `units NULL` = unknown, distinct from `0` | inventive, ux, structural | 7.2 |
| 4 | `≤ 100` hard, `= 100` derived status | readiness, contrarian | 7.3 |
| 5 | Replace-all `PUT` under optimistic version | ux, readiness, structural | 8.5, 8.6 |
| 6 | Composite account-scoped FKs to both parents | readiness, structural, ops | 8.7 |
| 7 | Real Estate granted read of `Ownership Share` | structural | 8.4 |
| 8 | Contract snapshots lessor; no temporal columns | structural, contrarian, ops | 10.4 |
| 9 | Archive freely in Owners; consumer refuses | contrarian, structural, readiness | 10.3 |
| 10 | No hard delete for either aggregate | structural, ops | 10.8 |
| 11 | Audit indexed on property **and** owner axes | ops | 10.6 |
| 12 | Audit in the same transaction; fail closed | ops, structural | 10.6 |
| 13 | Duplicate-check treated as a tenancy-oracle risk | readiness, ux | 11.4 |
| 14 | Tax identifier typed + aliased, surrogate key | structural, ops | 7.4 |
| 15 | Type derived from identifier, not a picker | ux | 9.2 |
| 16 | Read-only remainder row; split-equally action | ux | 9.5 |
| 17 | Duplicate panel keeps form state, offers Reactivar | ux | 9.4 |
| 18 | Ownership edits online-only in the PWA | ux, ops | 9.7 |
| 19 | `≤ 100` and unknown both surfaced in one status | readiness, ux | 5, 10.7 |
| 20 | ArchUnit rule labelled as dependency direction only | readiness, structural | 11.5 |
| 21 | Rejection fixtures kept out of seed data | readiness | 11.3 |
| 22 | ARCHITECTURE.md corrected or marked superseded | readiness, ux, inventive | 1, G-2 |
| 23 | "Rent split" named and rejected explicitly | contrarian | 12.4 |
| 24 | Revisit clause given a measurable trigger | inventive | 8.3 |

### 16.3 Objections not resolved by this document

1. **Who is the consumer of the percentage, and what decision does it drive?** Every lens that touched it said the answer determines whether the allocation is an aggregate or a join table. Section 12.4 rejects the billing assumption; nothing replaces it with a positive answer.
2. **Must a live contract show the owners as of signing, or the current owners?** This single answer decides D-06, the contract snapshot, and whether effective dating returns to v1.
3. **Can the same legal person be owner and tenant in one account, and must the product present one identity?** Decides whether D-09 is a deferral or a latent defect.
4. **Who is the audit actor?** ADR-0006 defines accounts and never users. Until an identity model exists, "who archived this" is a string.
5. **Does an implementation exist outside this repository?** `ARCHITECTURE.md:8-10` names subsystems that are absent here. If `aap-backend` lives elsewhere, the dirty-data census in Section 10.9 must run there before any unique index is declared.
6. **Is the agency's real data even complete?** The whole design assumes properties are onboarded before titling is known. `GLOSSARY.md:18` says so; no one has counted. A three-day census of ten real properties would settle it cheaper than this document did.

---

## 17. Governance actions

ADR-0001 requires new records rather than edits. Each action below is a prerequisite for the corresponding slice, not a nice-to-have.

| ID | Action | Blocks | Rationale |
| :--- | :--- | :--- | :--- |
| **G-1** | **ADR-0007 — Ownership Share representation**: integer units, `NULL` for unknown, `≤ 100` invariant, residue rule, `TitlingStatus` | Slice 2 | Records D-01 to D-04 and the deviation from the literal "0 when unknown" |
| **G-2** | **Glossary edits**: `Ownership Share` (unit + unknown), `Owner` ("zero or more"), new terms from Section 5; **and** correct `ARCHITECTURE.md:12-18` to the canonical English entity set | Slice 1 | ADR-0002:18 makes a stale glossary a defect |
| **G-3** | **ADR-0008 — Aggregate deletion and archival**: legal/financial aggregates are archived, never hard-deleted | Slice 1 | Resolves the `ON DELETE` default before the first migration sets it by habit |
| **G-4** | **ADR-0009 — Owner identity**: surrogate key, typed identifier list with aliases, normalisation and uniqueness, role-local identity | Slice 1 | Also records the accepted limitation that cross-role dedupe is out of scope |
| **G-5** | **ADR-0010 — Ownership audit and observability**: audit table shape, actor field, retention, PII policy, client build header | Slices 2–3 | No ADR currently covers audit; four modules would otherwise invent four shapes |
| **G-6** | **ADR-0005 extension — Owner as a Document Store owning record**, with consent and retention, and a decision on identifier-vs-document authority | Slice 3 (blocks any ID upload) | The module cannot hold identity evidence today |
| **G-7** | **ADR-0003 amendment — grant Real Estate a read of `Ownership Share`**, and record the revisit trigger in measurable terms | Slice 2 | Section 8.4 and 8.3 |

G-2 and G-7 are the two that make the *existing* documents true. The rest record decisions that would otherwise be made silently in code.

---

## 18. Alternatives weighed

For `/egm-draft` to carry into Alternatives Considered. Rejections are attributed to this brainstorm, not to a maintainer.

1. **Rejected by author — `double`/`float` percentages.** Epsilon-dependent equality; JSON round-trip drift; a sum check that is true and false depending on the order of operations.
2. **Rejected by author — `numeric(5,2)` percentages as the stored truth.** Exact per value, not closed under division; three equal owners sum to 99,99 and the residue rule gets discovered after the API contract is published.
3. **Rejected by author — hard `= 100` block at save.** Makes the glossary's documented unknown state unstorable and forces employees to invent a split to unblock onboarding.
4. **Rejected by author — warning-only with no derived state.** No enforcement point and no data-quality signal; the failure surfaces only when someone asks.
5. **Rejected by author — per-share `PATCH` endpoints.** Forces clients through transient totals above 100 and duplicates the invariant on the client.
6. **Rejected by author — ownership owned by Real Estate (without an ADR).** Legitimate only via a superseding decision; adopting it quietly pre-empts the ADR-0003 revisit clause instead of executing it.
7. **Rejected by author — a property-level `owner_id` column alongside the share table.** A second truth within one release; the exact single-owner assumption co-ownership breaks.
8. **Rejected by author — client-side portfolio composition.** Cannot paginate or sort a merged set server-side; turns the PWA cache into a correctness dependency.
9. **Rejected by author — a stored portfolio read model / projection in v1.** Eventual consistency and rebuild machinery for a query one indexed join already satisfies.
10. **Rejected by author — hard delete with a cross-module "has history" check.** Owners is not granted a read of Contract or Invoice, and the check belongs to a module that is not in this repository.
11. **Rejected by author — a generic audit framework or event sourcing.** The observed need is one append-only table; a framework is not reversible and no platform-wide audit ADR exists.
12. **Rejected by author — fuzzy name-based deduplication.** Spanish two-surname names produce false positives; trigram matching is an expensive write-path operation.
13. **Rejected by author — effective-dated shares in v1.** The contract-side snapshot answers the historical question on a seam ADR-0004 already created; temporal columns add the overlap-and-gap bug class before any consumer needs it.
14. **Rejected by author — an "Unknown owner" placeholder record.** Pollutes the owner list, has no valid identifier, becomes the default recipient of every unassigned property, and is effectively permanent once referenced.
15. **Rejected by author — a shared `Party` identity across owner, tenant, and candidate.** Correct long-term and out of scope here; recorded as a limitation (D-09) so it is a decision rather than an oversight.

---

## 19. Key decisions and staging

### 19.1 Decisions resolved by this brainstorm

1. Integer ownership units; percentage is a rendering, and the residue is allocated by largest remainder.
2. `OwnershipAllocation` is the aggregate root; `Ownership Share` is an entity inside it.
3. `≤ 100` is the only hard rule; `= 100` is a derived data-quality status, gated only at a consumer that does not yet exist.
4. Unknown is a state (`units NULL`), not a sentinel, rendered as "Sin datos".
5. One write path: replace the whole allocation under optimistic concurrency, with the committed state returned on conflict.
6. Cross-module reads may join (ADR-0003 sanctions the foreign key); writes may not. One named join, one governance check.
7. Composite account-scoped foreign keys to both parents; the Real Estate `UNIQUE (account_id, id)` is an explicit cross-module dependency.
8. Archival never changes title; nothing is hard-deleted; the consumer refuses, not the producer.
9. Contract snapshots its lessor at signing; no temporal columns in Owners; the audit table keeps the option open.
10. `Owner.id` is an immutable surrogate; tax identifiers are typed, normalised, aliased attributes.
11. Identity is role-local; cross-role deduplication is an accepted limitation.
12. Owner Type is derived from the identifier; `legalName` is computed server-side.
13. Ownership editing is online-only; owner creation may be offline as a visible draft.
14. Audit is same-transaction, indexed on property and owner, and readable by support.
15. The "rent split" reading is explicitly rejected until an ADR and an issue exist for it.

### 19.2 Delivery slices

Not implementation tasks — a staging proposal so the design can be reviewed in pieces.

**Slice 1 — Identity.** `Owner` aggregate with computed `legalName`, typed identifier with normalisation and checksum, account-scoped partial unique index, archive/restore, search and list with `propertyCount`, responsive form, audit of owner mutations. Blocked by G-2, G-3, G-4. Delivers the "stable legal identity the agency can trust" value immediately and is independently shippable.

**Slice 2 — Ownership.** `OwnershipAllocation` and shares, `PUT` replace-all with optimistic version, co-ownership editor, `TitlingStatus`, composite FKs, ownership audit table, Real Estate's `UNIQUE (account_id, id)`, and the property screen's read-only owner list with the deep link. Blocked by G-1, G-5, G-7. This is the slice that must not ship before D-01 to D-05 and D-10 are closed.

**Slice 3 — Portfolio and consumer integration.** Portfolio screen and query, Contract's lessor snapshot plus the Contract Template multi-owner merge field, the data-quality report and `TitlingStatus` gauge, the drift query, and the status/role model that gives the audit an actor.

Deferred, with no regret: merge tooling, effective-dated shares, ownership rights types (`usufruct`), owner ID-document upload, bulk import, a read-model projection, hard delete, and any rent-split calculation.

### 19.3 What "shipped" must demonstrate

An owner can be created, found, edited, and archived with a validated Spanish identifier, cannot be duplicated inside its account, and is findable when archived. A property can carry three owners at 33,33 / 33,33 / 33,34 as exact units, and the editor refuses 120 %. A property with unknown titling can be saved and reports `Unknown`. A portfolio of 400 properties loads in one request. Two simultaneous edits produce one 412 and never a 110 % total. A support engineer can reconstruct who changed a share from the audit timeline alone, without logs and without SQL.

---

## 20. Handoff

This is a brainstorm, not a design. Left to `/egm-draft`:

- Table DDL, index definitions, constraint expressions, and the exact composite FK syntax.
- The OpenAPI fragment: schemas, error codes, `If-Match` semantics, and the portfolio projection.
- The tax-identifier contract: allow-listed characters, normalisation function, type detection, checksum algorithm with its citation, and the shared test-vector file.
- The allocation command's contract: validation order, residue algorithm, conflict payload.
- The retention number for the audit table, and the anonymisation path for erasure.
- Wireframes and the Spanish copy deck.
- The ADRs in Section 17, each written as a Nygard record.

**Before drafting:** close the eleven decisions in Section 15, land G-2 and G-7 so the accepted documents stop contradicting each other, and answer the one question the reviews kept circling — *does anything consume the ownership percentage, and will a live contract show owners as of signing or as they are now?* Everything else in this document is downstream of that answer. The design's weakest joint is the `NULL`-for-unknown deviation; its most expensive mistake would be freezing the schema before D-01, D-06, and D-09 are settled.
