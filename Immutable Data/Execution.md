# Immutable Data — Execution

See **[Overview](Overview.md)** for why, and **[Concept](Concept.md)** for the model. This page describes how the live product actually implements it, then sketches the genuine multi-party extension the model in [Concept](Concept.md) points toward.

## 1. What's actually running

Every contract's history is one hash-linked chain in DIGASSAY's own database (`DigAssay.LedgerEntry`, `digassay-api`), appended to and read through a small service (`src/services/ledger.js`):

| Column | Purpose |
| :--- | :--- |
| `SupplyContractId`, `SequenceNo` | Which contract's chain this entry belongs to, and its position in it |
| `SourceType`, `SourceId` | What kind of event this is (an inspection, a seller/buyer/umpire assay, a private or shared assay report, an umpire determination) and which underlying record it came from |
| `VisibilityRoles` | Who is entitled to read this entry's contents — a comma list of roles, or `ALL` |
| `PayloadJson` | The event's own data, snapshotted at commit time |
| `PrevHash`, `EntryHash` | The fingerprint chain described in [Concept](Concept.md) |
| `EventAtUtc` | The real-world event time used to order the chain |

`appendEntry()` computes each new `EntryHash` from the previous entry's hash plus the new payload, and writes the row. `viewAs(role)` reads the chain back and, for any entry whose `VisibilityRoles` doesn't include the requesting role, returns the hash and metadata but redacts the payload to `{ restricted: true }` — the entry's existence and position in the chain are never hidden, only its contents. On every read, `verifyChain()` recomputes each entry's hash from its stored payload and compares it against what was recorded — not a cached "valid" flag — so the live view can show, per entry, whether it still checks out.

## 2. Where it shows up

- **`/contracts/:id` → Immutable Data** — the full chain for a contract: every inspection, every assay, every private and shared report and umpire determination, in order, with a "View as" role selector (Public, Seller, Buyer, Umpire, Carrier) that drives the same redaction `viewAs()` performs server-side.
- **`/contracts/:id/quality-compliance`** — the agreed Quality Specification's parameters against every assay report that's arrived, one column per report, each cell locked or readable depending on the same visibility rule. A **Trigger umpire process** button picks an existing report at random, simulates a binding Umpire Determination against it (visible to every role, matching real umpire-assay convention), and appends it to the same chain.
- A demo-only **unlock (demo)** toggle, per item and in bulk, lets a visitor reveal what a restricted entry actually contains without needing a real counterparty session — clearly marked as a demonstration affordance, not something a real deployment would expose.

## 3. Report types and who's entitled to read them

| `SourceType` | Who can read the contents |
| :--- | :--- |
| `EVIDENCE`, `ASSAY_SELLER`, `ASSAY_BUYER`, `ASSAY_UMPIRE`, `ASSAY_FINAL` | Set per record, matching the real Seller/Buyer/Umpire assay-exchange process |
| `ASSAY_REPORT_PRIVATE_EXTRACTION`, `ASSAY_REPORT_PRIVATE_DELIVERY` | Seller and Umpire only |
| `ASSAY_REPORT_SHARED_WITH_CUSTOMER` | All parties |
| `ASSAY_REPORT_UMPIRE_DETERMINATION` | All parties — a ruling that resolves a dispute is shared, not private to one side |

## 4. The genuine multi-party extension

Today the chain lives in one database — DIGASSAY's own — so a counterparty still has to trust DIGASSAY's server to report the true recomputed hash honestly, even though tampering with historical data is immediately detectable by that same recomputation. The natural next step is spreading custody across the counterparties themselves, so no single organisation, DIGASSAY included, holds the only copy. **Hyperledger Fabric** (Linux Foundation) is a mature, permissioned distributed-ledger platform purpose-built for exactly that shape — its two building blocks map directly onto what [Concept](Concept.md) describes:

| Fabric concept | Maps to |
| :--- | :--- |
| **Channel** | A sub-ledger visible only to its members — e.g. one channel per contract |
| **Private Data Collection** | Within a channel, a record whose actual contents are stored only on the peers of *named* authorised organisations — every other channel member still sees a tamper-evident hash of it, just not the contents |
| **Organisation / MSP** | Each counterparty (seller, buyer, umpire, carrier) as a distinctly identified organisation — mapped onto the same real company/LEI registry identity DIGASSAY already establishes for counterparties today |
| **Ordering service** | Agrees the sequence new entries are appended in, so every participant's copy of the ledger ends up identical |

This is named here as a concrete, provable example that the model is buildable with mature, existing tools — not a commitment to build on this specific platform, and no ledger platform has been selected.
