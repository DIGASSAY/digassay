# Freight & Cargo Integrations — v1.00

Master page for DIGASSAY's freight/cargo vendor integrations. Each vendor has its own child page (numbered `1.01`, `1.02`, ... below) with full technical detail; this page explains what's supported today and how the pieces fit together as one architecture, not three unrelated one-offs.

## Supported today

| # | Vendor | What it is |
| :--- | :--- | :--- |
| [1.01](<Freight and Cargo 1.01 CargoWise.md>) | **CargoWise** (WiseTech Global) | Enterprise freight-forwarder logistics platform - eAdaptor XML messaging (`UniversalShipment` / `UniversalEvent`). |
| [1.02](<Freight and Cargo 1.02 Coneksion.md>) | **Coneksion** (Youredi) | Cloud integration-as-a-service middleware for ocean/bulk carrier connectivity - OAuth 2.0 REST API + webhooks. |
| [1.03](<Freight and Cargo 1.03 Endur.md>) | **OpenLink Endur** (ION Group) | Front-to-back ETRM/CTRM platform (trade, risk, logistics) - JVS/OpenComponents, User Tables, DEX. Targets release S25. |

## How the pieces fit together

Two of the three (CargoWise, Coneksion) connect straight to their own vendor networks. The third, Endur, **already has its own established links to many of those same carriers** as part of its own Logistics/Operations module - so a given carrier's data can reach DIGASSAY either **directly** through its own connector, or **indirectly by way of Endur**, if that's the path the counterparty or trading desk already uses. Either way, everything lands in one place and is normalized to one internal shape before DIGASSAY ever sees it - DIGASSAY is never written to know three different vendor formats.

The connection is two-way. DIGASSAY doesn't just consume tracking data - it reports back: inspection issues logged against a leg (see the Inspection Evidence table in the delivery diary) go back out to the relevant carrier electronically, and DIGASSAY keeps the vendor side informed as a contract's main terms change, rather than that sync happening by phone or email.

```mermaid
flowchart LR
    classDef n fill:#DCE5D5,stroke:#2F4A32,color:#263526,stroke-width:2px;
    classDef c fill:#2F4A32,stroke:#1F3021,color:#FFFFFF,stroke-width:4px;
    classDef roadmap fill:#F5F0E3,stroke:#B89B5E,color:#2D3C21,stroke-width:2px,stroke-dasharray:5 5;

    subgraph V["Freight &amp; Cargo Vendors"]
        direction TB
        CW["CargoWise"]
        CX["Coneksion"]
        CARR["Carrier / operator systems<br/>(e.g. Oldendorff, Pacific Basin)"]
    end

    OE["OpenLink Endur<br/>(ION Group)"]
    CARR -.->|"same carrier, reachable<br/>via Endur's own link too"| OE

    V -->|inbound| LZ[("Landing Zone")]
    OE -->|"inbound, via Endur"| LZ

    LZ --> NF["Normalize to<br/>Common Format"]
    NF --> DA(("DIGASSAY"))

    DA -->|"issues logged +<br/>terms updates"| RF["Common Format<br/>(outbound)"]
    RF --> LZ

    DA -.-> ACC["Accounting / Finance<br/>(potential)"]
    DA -.-> LEG["Legal<br/>(potential)"]

    class CW,CX,OE,CARR n; class LZ,NF,RF c; class DA c; class ACC,LEG roadmap;
```

**Landing Zone → Common Format** — every inbound message, whichever vendor or path it arrived by, lands here first and is normalized into one internal shape (the same shape DIGASSAY's own `DeliveryLegTracking` / `EvidenceRecord` tables already use) before DIGASSAY consumes it. A new vendor is one more thing writing into the Landing Zone, not a new thing DIGASSAY has to learn to speak.

**Return loop** — the same Landing Zone/Common Format layer runs in reverse: an inspection issue raised in DIGASSAY, or a contract term change, is written out in the common shape and translated back into each vendor's own format (a CargoWise `UniversalEvent`, a Coneksion webhook call, an Endur `DEX` update) for delivery to whichever system actually needs to hear about it.

**Accounting / Finance and Legal** (dashed, potential) — not scoped yet, but the natural next connections once freight/cargo data is flowing: settlement and invoicing data to Accounting/Finance, and contract terms/compliance obligations to Legal, both consuming the same Common Format rather than a fresh bespoke integration each.
