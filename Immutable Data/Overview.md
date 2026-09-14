# Immutable Data — Overview

A roadmap capability: a tamper-evident, shared history of trade events that counterparties can rely on without taking either side's word for it. Three pages cover it — this **Overview** (why it matters, in plain terms), **[Concept](Concept.md)** (the model, without the jargon), and **[Execution](Execution.md)** (how it would actually be built).

## The problem it solves

A physical trade dispute is hard to resolve cleanly because there usually isn't one history both sides trust equally. An assay result, an inspection finding, a delivery timestamp — today these live as PDFs, screenshots and email threads, any of which can be edited, reissued or "lost" after the fact with no way for the other side to prove it. Even acting in complete good faith, two counterparties can end up with two different stories about what happened and when, simply because there was never one shared, provably-unaltered record to point to.

## What "immutable" means here

Not that nothing can ever change — new information is added constantly (a fresh assay result, a follow-up inspection, an updated status). It means what's *already been recorded* can never be silently altered or deleted. A correction is itself a new, dated entry layered on top of the old one, never a rewrite of it. Anyone with access to a record can independently verify it hasn't been tampered with since it was written — they don't have to trust the platform, or the counterparty, to tell them so.

## Today vs. the roadmap

The live platform already behaves this way at the *database* level: the Inspection Evidence and Assay records tables are insert-only — there is no code path that updates or deletes a row once written (see `usp_EvidenceRecord_Create`, `usp_AssayExchange_SetOutcome`, which only ever add). That's honest append-only behaviour today, but it's a property of the application code, not something a counterparty could independently verify without trusting DIGASSAY's own server. The roadmap step described in [Concept](Concept.md) and [Execution](Execution.md) is making that tamper-evidence **independently provable** — hash-linked and shared across the counterparties themselves, not just a convention inside one party's own database.

## Who sees what

Not every record should be visible to every counterparty. A seller may commission a private assay report from their own umpire before ever sharing a number with the buyer; an inspection report might be relevant only to the two parties present at that leg, not the whole chain of custody. "Immutable" and "visible to everyone" are treated as two separate properties, not one — see **[Concept](Concept.md#selective-visibility)** for how a record can be tamper-evident and provably unaltered while its actual contents stay restricted to the parties entitled to read it.

## Status

Roadmap — not yet built. It extends the **Inspection Evidence** and **Assay Exchange** records already live in the [Delivery Diary](https://digassay.nrgpix.com) today; see **[digassay.nrgpix.com](https://digassay.nrgpix.com)** for the current, insert-only version of these records.
