# Immutable Data — Overview

A tamper-evident, shared history of trade events that counterparties can rely on without taking either side's word for it — live today on every supply contract. Three pages cover it — this **Overview** (why it matters, in plain terms), **[Concept](Concept.md)** (the model, without the jargon), and **[Execution](Execution.md)** (how it's actually built).

## The problem it solves

A physical trade dispute is hard to resolve cleanly because there usually isn't one history both sides trust equally. An assay result, an inspection finding, a delivery timestamp — today these live as PDFs, screenshots and email threads, any of which can be edited, reissued or "lost" after the fact with no way for the other side to prove it. Even acting in complete good faith, two counterparties can end up with two different stories about what happened and when, simply because there was never one shared, provably-unaltered record to point to.

## What "immutable" means here

Not that nothing can ever change — new information is added constantly (a fresh assay result, a follow-up inspection, an updated status). It means what's *already been recorded* can never be silently altered or deleted. A correction is itself a new, dated entry layered on top of the old one, never a rewrite of it. Anyone with access to a record can independently verify it hasn't been tampered with since it was written.

## Live today

Every delivery milestone, inspection finding and assay outcome on a DIGASSAY contract is committed to a hash-linked ledger, one chain per contract — see **[Concept](Concept.md)** for the model and **[Execution](Execution.md)** for the real schema and service behind it. The **Quality Compliance** page builds directly on it: the agreed Quality Specification's parameters, fixed down the left, against every assay report that's arrived for a delivery — a private report commissioned at the point of extraction, a private one at the delivery point, one shared with the customer — each measured value graded against its agreed band and colour-coded by deviation. A **Trigger umpire process** action picks one of those reports at random as the disputed assay, simulates a binding Umpire Determination against it, and emails the resulting profit-and-loss impact before it ever appears on screen.

## Who sees what

Not every record is visible to every counterparty. A seller may commission a private assay report before ever sharing a number with the buyer; an inspection report might be relevant only to the two parties present at that leg, not the whole chain of custody. "Tamper-evident" and "visible to everyone" are two separate properties here, not one — see **[Concept](Concept.md#4-selective-visibility)** for how a record can be provably unaltered while its actual contents stay restricted to the parties entitled to read it. The live demo also carries a "View as" role selector and a per-item/bulk **unlock (demo)** toggle, purely so a visitor can explore what a restricted record actually contains without needing a real counterparty login.

See **[digassay.nrgpix.com](https://digassay.nrgpix.com)** — open any contract's Immutable Data view or its Quality Compliance page to see a real chain.
