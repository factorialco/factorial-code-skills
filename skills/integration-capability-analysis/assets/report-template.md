# <Vendor> — <domain> integration mapping

**Vendor specification:** <name / file / URL, with version or date if stated>
**Specification type:** <OpenAPI 3.0 | developer portal | PDF guide | file layout | mixed>
**Domain:** <payroll | finance> — <one line on why>
**Factorial documentation:** <domain page URL, including its version>
**Analysed on:** <date>

> **Completeness of input:** <whole spec | partial — what was missing or
> unreachable, and that the verdicts below are bounded by it>

## Capability summary

| Capability | Direction | Verdict | Vendor endpoints | Blocking gap |
|---|---|---|---|---|
| <capability> | Outbound | Fully supported | `POST /path` | — |
| <capability> | Outbound | Partially supported | `POST /path`, `PATCH /path` | — <em-dash: shippable as-is> |
| <capability> | Inbound | Partially supported | `GET /path` | <what must be closed before it can ship> |
| <capability> | Inbound | Not supported | — | <absent / spec silent — see question N> |

<One short paragraph: what can be built now, the biggest single blocker, and
whether this vendor is viable for the domain at all.>

## Cross-cutting findings

These apply to every capability. Write "not documented" where the spec is silent.

- **Authentication:** <scheme; applied to operations or only declared; per-tenant credentials?>
- **Record identity:** <the vendor's identifier, whether it is returned on create, and how we match records on the next sync>
- **Legal entity / company scoping:** <how the vendor addresses it>
- **Effective dating:** <supported, immediate-only, or unclear>
- **Pagination & incremental reads:** <mechanism; modified-since filter?>
- **Idempotency & corrections:** <keys, replace semantics, or none>
- **Errors:** <declared at all? per record or per batch?>
- **Rate limits:** <stated limits, or not documented>
- **Sandbox:** <available? how obtained?>
- **Setup order:** <dependencies between capabilities or records, from the Factorial domain page and the vendor spec>

## Specification quality

Defects in the vendor's document, including examples that contradict the
schema. "None found" if there are none.

- <defect>

---

## <Capability> — <verdict>

**Direction:** <Outbound — Factorial → vendor | Inbound — vendor → Factorial>

### Endpoints

In the order an implementation calls them.

| # | System | Operation | Purpose |
|---|---|---|---|
| 1 | Factorial | `GET /...` | <read the source records> |
| 2 | Vendor | `GET /...` | <resolve a reference id the create needs> |
| 3 | Vendor | `POST /...` | <create the record> |
| 4 | Vendor | `PATCH /...` | <update it on later changes> |

### Field mapping

| Source field | Destination field | Required | Transform | Evidence |
|---|---|---|---|---|
| `GET /employees → first_name` | `POST /workers → person.given_name` | Yes | — | schema |
| `GET /employees → birth_on` | `POST /workers → person.dob` | Yes | `YYYY-MM-DD` → `DD/MM/YYYY` | example |
| `GET /contracts → working_hours` | `POST /workers → fte` | No | hours ÷ legal weekly hours | docs §3.2 |

**Value mappings** — <for each code-list field: source value → destination
value, or "configured per customer">

| Source value | Destination value |
|---|---|
| `<value>` | `<value>` |

### Gaps

**Vendor-required fields with no Factorial source:** <field — resolution:
constant / per-customer configuration / derivation / blocking gap>

**Factorial data with no vendor destination:** <fields, or "none">

### Verdict

**Limitations:** <numbered, for partial verdicts; for not supported, what is
absent and whether the spec rules it out or is silent>

1. <limitation>

**Viability:** <partial only — shippable as-is, or needs the gap closed first and
what closing it takes>

<Repeat the capability section for every capability on the domain page.>

---

## Questions for the vendor

Ordered by how much the answer would change the plan.

1. <question> — *changes:* <which capability or mapping depends on it>

## Assumptions made

<Anything assumed rather than read from either specification. Every line here
is a place the document could be wrong.>
