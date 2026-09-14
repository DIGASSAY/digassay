# Immutable Data — Concept

See **[Overview](Overview.md)** for why this matters. This page lays out the model itself — five ideas, each building on the last, that together produce a shared record no single party can quietly rewrite, while still keeping sensitive content private to only the parties entitled to see it.

## 1. Hash-linked history

Every new record — a delivery milestone, an inspection finding, an assay outcome — is stamped with a fingerprint (a hash) that's mathematically derived from its own contents *and* the fingerprint of the record before it. Change so much as one character in a past record and its fingerprint changes, which breaks every fingerprint after it — instantly and visibly, to anyone checking. This is what makes the history tamper-evident without needing a referee: you don't have to trust that nobody edited yesterday's entry, you can prove it for yourself by recomputing the chain.

## 2. Shared custody, not one company's database

Today, that chain of records lives inside DIGASSAY's own database — trustworthy in practice, but a counterparty still has to take DIGASSAY's word that nothing was altered. The roadmap model spreads custody of the record across the counterparties themselves — seller, buyer, umpire, carrier each hold their own copy and take part in agreeing what gets added next. No single participant, including DIGASSAY, can unilaterally rewrite history, because every other participant is holding the same hash-linked chain and would immediately notice a mismatch.

## 3. Permissioned, not public

This isn't an open, anonymous network anyone can join — every participant is a known, identified counterparty to a specific contract (a seller, a buyer, a named umpire, a carrier), admitted the same way DIGASSAY already establishes counterparty identity today, via real company/LEI registry data (see the [ecosystem diagram](https://github.com/DIGASSAY)). Only parties who are actually party to a given trade can see or add to its record.

## 4. Selective visibility

Being tamper-evident and being visible to everyone are two different properties, and this model deliberately separates them. Two concrete examples from the current product:

- **A private assay report the seller commissions.** The seller may run their own umpire assay before quoting a number to the buyer. The buyer doesn't need — and isn't entitled — to read that private report. But the *fact* that a report was produced, and exactly when, can still be committed to the shared history as a tamper-evident fingerprint, without the report's actual contents ever being exposed to the buyer. If a dispute later turns on that report, the seller can selectively reveal its contents and any other participant can independently confirm the revealed document matches the fingerprint committed at the time — it hasn't been altered or swapped after the fact.
- **An inspection report shared between only the parties present.** A road-to-rail handover inspection might only involve the shipper and the carrier — the buyer, three legs downstream, has no need to see it. That report can be recorded as visible only to the parties who were actually there, while its existence and timing are still provably part of the same overall chain of custody as every other leg.

## 5. Proven patterns, not a new invention

This isn't a novel idea — permissioned networks with exactly this "shared ledger plus private, restricted-visibility sub-records" shape are an established pattern in enterprise data-sharing platforms (see **[Execution](Execution.md)** for a concrete, named example). DIGASSAY's own contribution isn't inventing the underlying mechanism — it's applying an established pattern to the specific shape of a physical commodity trade: contract → delivery legs → inspection evidence → assay outcome, exactly the structure already live in the product today.
