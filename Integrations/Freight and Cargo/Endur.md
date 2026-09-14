# Integration Guide: Connecting an App to OpenLink Endur (ION Group)

## Overview
**Endur** (originally OpenLink, now marketed by **ION Group** as its Openlink commodity management platform - the industry, and most of its own consultant ecosystem, still call it "Endur") is a front-to-back Energy/Commodity Trading and Risk Management (ETRM/CTRM) platform: trade capture, position and risk, **scheduling and logistics** (truck, rail, pipeline, vessel, grid), and settlement/accounting, across oil, gas, power, LNG, bulk/metals and increasingly agriculture. Its Logistics/Operations module is the natural counterpart to DIGASSAY's own Delivery Diary - a real integration would let a physical delivery captured in DIGASSAY appear as a scheduled movement in Endur, and let Endur's own trade/position data enrich a DIGASSAY contract.

Unlike CargoWise's eAdaptor (a published, vendor-defined XML schema every partner integrates against identically), Endur has **no single universal public API contract**. Every real integration is built against the specific mechanisms below, configured per client instance - so the payload shapes in this guide are illustrative, not a fixed spec, and must be confirmed against the target deployment before use.

---

## 1. Version & Compatibility

| | |
| :--- | :--- |
| **Product name today** | ION Openlink ("Openlink Commodity Management Software"), ION Group |
| **Still commonly called** | OpenLink Endur, or just "Endur" |
| **Version targeted by this guide** | **S25** (ION's current season/year-coded release line, released January 2025) |
| **Architecture assumed stable since** | **v14** - the JVS/OpenComponents/User Table extensibility model described below has been the standard integration surface since the long-lived v14 release line, through the rename to ION and the move to S-series releases |
| **Confirm before integrating** | The specific Endur instance's version - real deployments commonly run several releases behind current, and JVS/OpenComponents customizations are written and tested against one specific version |

---

## 2. System Architecture & Flow (Mermaid Diagram)

The realistic pattern for a DIGASSAY ↔ Endur integration: DIGASSAY writes delivery/logistics data into a staging **User Table** (or a secure file drop), a scheduled **JVS/OpenComponents** process picks it up and maps it onto native Endur trade/logistics objects, and status changes flow back out via Endur's **DEX (Data Exchange)** interface.

```mermaid
sequenceDiagram
    autonumber
    actor DA as DIGASSAY
    participant Stage as Endur User Table / Secure File Drop
    participant JVS as JVS / OpenComponents Process
    participant Core as Endur Trade &amp; Logistics Objects
    participant DEX as Endur DEX (Data Exchange)

    Note over DA, Stage: Phase 1: Stage the delivery data
    DA->>Stage: Write delivery/leg record (User Table INSERT, or CSV/XML file drop)

    Note over Stage, Core: Phase 2: Scheduled ingestion &amp; mapping
    JVS->>Stage: Poll for new/changed rows (scheduled job)
    activate JVS
    JVS->>JVS: Validate + map fields to Endur schema (deal, movement, party, location)
    JVS->>Core: Create/update Logistics Movement &amp; linked Trade
    activate Core
    Core-->>JVS: Object ID(s), validation result
    deactivate Core
    JVS-->>Stage: Mark row processed / write result back
    deactivate JVS

    Note over Core, DA: Phase 3: Status &amp; event synchronisation
    Core->>DEX: Movement/trade status change (e.g. discharged, settled)
    DEX->>DA: Outbound event (DEX stream, message queue, or scheduled extract)
    DA-->>DEX: Acknowledge receipt
```

---

## 3. Integration Mechanisms

Endur exposes several distinct, independently-usable mechanisms rather than one API surface - which combination is used is a deployment decision, not a fixed choice.

| Mechanism | Direction | Purpose |
| :--- | :--- | :--- |
| **User Tables** | Bidirectional | Custom relational tables added to the Endur database itself, visible in the Endur GUI and readable/writable via JVS/OpenComponents - the standard place to stage external data for a scheduled process to pick up. |
| **JVS / OpenJVS** (Java View Studio) | Internal (Endur-side logic) | Java-based scripting/customization layer used to build the screens, validation and mapping logic that turn staged data into native Endur objects. |
| **OpenComponents** | Internal (Endur-side logic) | Component-based API/framework for building custom business logic and UI panels against Endur's core trade/position objects - often used alongside JVS. |
| **DEX (Data Exchange)** | Outbound (near real-time) | Endur's own data-exchange interface for streaming trade-lifecycle and position events to downstream systems. |
| **JDBC / ODBC** | Bidirectional | Direct database connectivity for reporting or bulk data access, where a full JVS-mediated integration isn't required. |
| **Secure File Ingestion** | Inbound | Structured file drop (CSV/XML) picked up by a scheduled import job - the lowest-friction option for periodic, non-real-time data. |
| **Exchange / Broker Adaptors** | Inbound | Native connectivity to major exchanges and brokers for trade capture - not generally relevant to a logistics-data integration like DIGASSAY's. |

---

## 4. Illustrative Data Mapping (User Table Staging)

Example only - the actual table/column names are defined per Endur instance during implementation, not published by ION.

```sql
-- Illustrative staging table for a DIGASSAY delivery leg
CREATE TABLE USER_digassay_delivery_leg (
    external_leg_id      VARCHAR(50)   NOT NULL,  -- DIGASSAY DeliveryLeg.Id
    contract_ref         VARCHAR(50)   NOT NULL,  -- DIGASSAY ContractRef (e.g. SC-000002)
    transport_mode       VARCHAR(10)   NOT NULL,  -- ROAD | RAIL | VESSEL
    departure_location    VARCHAR(150),
    arrival_location      VARCHAR(150),
    scheduled_start_utc   DATETIME2,
    scheduled_end_utc     DATETIME2,
    actual_start_utc      DATETIME2     NULL,
    actual_end_utc         DATETIME2     NULL,
    operator_name         VARCHAR(150),            -- e.g. 'Oldendorff Carriers'
    process_status        VARCHAR(20)   DEFAULT 'PENDING'  -- PENDING | PROCESSED | ERROR
);
```

A JVS/OpenComponents job polls `process_status = 'PENDING'`, maps each row onto an Endur **Logistics Movement** (and its linked **Trade**, via `contract_ref`), and writes the resulting Endur object ID plus `process_status = 'PROCESSED'` back to the row.

---

## 5. Field Mapping Dictionary

### Delivery / Movement Fields

| DIGASSAY Field | Endur Concept | Notes |
| :--- | :--- | :--- |
| `DeliveryLeg.TransportMode` | Movement transport mode | Maps to Endur's own mode codes (truck/rail/pipeline/vessel) - not a 1:1 string match, translated in the JVS mapping layer. |
| `DeliveryLeg.DepartureLocation` / `.ArrivalLocation` | Movement origin/destination location | Endur locations are master-data entities - free text must resolve to an existing (or newly-created) Endur location record. |
| `DeliveryLegTracking.PlannedStartUtc` / `.PlannedEndUtc` | Movement scheduled window | |
| `DeliveryLegTracking.ActualStartUtc` / `.ActualEndUtc` | Movement actual window | Null until DIGASSAY (or, in a live deployment, Live Carrier Tracking - see the organization README) has a real value. |
| `FreightOperator.Name` | Movement carrier/operator party | Resolved against Endur's party master data, not stored as free text. |

### Contract Linkage Fields

| DIGASSAY Field | Endur Concept | Notes |
| :--- | :--- | :--- |
| `SupplyContract.ContractRef` | Linked Trade/Deal reference | The join key between a DIGASSAY contract and its Endur-side trade record. |
| `SupplyContract.CounterpartyId` | Trade counterparty party | Resolved against Endur's counterparty master data. |
| `Quality.*` | Trade quality/specification terms | Endur models commodity specification natively; mapping depends on the commodity template configured for the instance. |

---

## 6. Status & Event Synchronisation

Outbound status changes (a movement discharged, a trade settled) are the near-real-time counterpart to the inbound staging flow, delivered via **DEX** rather than a webhook in the CargoWise/Coneksion sense - the specific transport (message queue, DEX stream subscription, or a scheduled extract) is a deployment-time choice, not a fixed HTTP callback contract.

```
Illustrative DEX event (conceptual - actual shape is deployment-specific)
{
  "eventType": "MOVEMENT_STATUS_CHANGED",
  "externalLegId": "34",
  "endurMovementId": "8842011",
  "status": "DISCHARGED",
  "eventDateTimeUtc": "2026-09-14T06:00:00Z"
}
```

The receiving application (DIGASSAY, or an intermediary) should treat this the same way it would a `DeliveryLegTracking` update: set `ActualEndUtc` once a movement is confirmed discharged/settled.
