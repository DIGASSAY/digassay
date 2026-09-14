# Immutable Data — Concept

See **[Overview](Overview.md)** for why this matters. This page lays out the model itself — five ideas, each building on the last, that together produce a shared record no single party can quietly rewrite, while still keeping sensitive content private to only the parties entitled to see it.

## 1. Hash-linked history

Every new record — a delivery milestone, an inspection finding, an assay outcome — is stamped with a fingerprint (a hash) that's mathematically derived from its own contents *and* the fingerprint of the record before it. Change so much as one character in a past record and its fingerprint changes, which breaks every fingerprint after it. This is what makes the history tamper-evident: you don't have to trust that nobody edited yesterday's entry, you can recompute the chain and see for yourself.

## 2. Recomputed, not cached

Every time a viewer loads a contract's chain, DIGASSAY recomputes each entry's fingerprint from its stored contents and compares it against the fingerprint that was recorded at the time — live, on that request, not a cached flag written once and trusted forever. A single altered byte anywhere in the history breaks the comparison immediately and visibly, for that entry and every one after it. Today that chain is held in DIGASSAY's own database; a genuine multi-party ledger, where each counterparty holds and cross-checks their own copy so no single organisation — DIGASSAY included — could unilaterally rewrite it, is the natural extension of the same model, illustrated in **[Execution](Execution.md)**.

## 3. Permissioned, not public

This isn't an open, anonymous network anyone can join — every participant is a known, identified counterparty to a specific contract (a seller, a buyer, a named umpire, a carrier), admitted the same way DIGASSAY already establishes counterparty identity, via real company/LEI registry data (see the [ecosystem diagram](https://github.com/DIGASSAY)). Only parties who are actually party to a given trade can see or add to its record.

## 4. Selective visibility

Being tamper-evident and being visible to everyone are two different properties, and this model deliberately separates them. Two concrete examples, both live on the product today:

- **A private assay report the seller commissions.** The seller may run their own umpire assay before quoting a number to the buyer. The buyer doesn't need — and isn't entitled — to read that private report. But the *fact* that a report was produced, and exactly when, is still committed to the shared history as a tamper-evident fingerprint, without the report's actual contents ever being exposed to the buyer. On DIGASSAY's Quality Compliance page this shows up literally as a locked cell — the row and column both visible, the measured value hidden — for any viewer role not entitled to read it.
- **An umpire's determination.** When a report is disputed, DIGASSAY's umpire process simulates a binding ruling against one of the existing reports and commits it visible to every party — matching real umpire-assay convention, where the ruling itself resolves the dispute for everyone rather than staying private to one side.

## 5. Proven patterns, not a new invention

This isn't a novel idea — hash-linked history with private, restricted-visibility sub-records is an established pattern in enterprise data-sharing platforms (see **[Execution](Execution.md)** for a concrete, named example of the multi-party version). DIGASSAY's own contribution isn't inventing the underlying mechanism — it's applying an established pattern to the specific shape of a physical commodity trade: contract → delivery legs → inspection evidence → assay outcome → umpire determination.
