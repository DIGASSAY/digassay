# Integration Guide: Connecting an App to CargoWise (WiseTech Global)

## Overview
**CargoWise** (by WiseTech Global) is an enterprise logistics management software used widely by freight forwarders, NVOCCs, and 3PLs. Connecting an external app to CargoWise allows you to pass shipment data, create bookings, or trigger jobs directly inside a company's CargoWise enterprise environment.

---

## 1. System Architecture & Flow (Mermaid Diagram)

CargoWise exposes integration capabilities primarily via **eAdaptor** (a SOAP/REST web service interface) or **CargoWise eServices / API Gateway**. The diagram below illustrates the end-to-end flow between an external application, the eAdaptor gateway, and the CargoWise Enterprise database/workflows behind it.

```mermaid
sequenceDiagram
    autonumber
    actor App as External Application
    participant Gateway as CargoWise eAdaptor Gateway
    participant Enterprise as CargoWise Enterprise DB &amp; Workflows

    Note over App, Gateway: Phase 1: Connection &amp; Authentication
    App->>Gateway: HTTPS connection (Basic Auth over TLS, or WS-Security header: ClientID / UserId / Password)
    Gateway-->>App: TLS session established

    Note over App, Enterprise: Phase 2: Shipment / Booking Submission
    App->>App: Construct UniversalShipment XML (DataContext, Parties, Routing, PackingLineCollection)
    App->>Gateway: POST /eAdaptor/eAdaptorInboundService.svc (UniversalShipment XML envelope)
    activate Gateway
    Gateway->>Gateway: Validate schema + envelope metadata
    Gateway->>Enterprise: eAdaptor ingestion (UniversalShipment)
    activate Enterprise
    Enterprise-->>Gateway: Record created / workflow triggered
    deactivate Enterprise
    Gateway-->>App: UniversalResponse XML (ACK or validation errors)
    deactivate Gateway

    Note over Enterprise, App: Phase 3: Status &amp; Event Updates
    Enterprise->>Gateway: UniversalEvent raised (status change, milestone)
    Gateway->>App: eAdaptor outbound POST (UniversalEvent XML)
    App-->>Gateway: 200 OK (Ack)
```

### Protocol & Security
* **Interface**: eAdaptor Inbound Web Service (SOAP or REST endpoint).
* **Data Format**: WiseTech Universal XML (`UniversalShipment` or `UniversalEvent`).
* **Authentication**: Basic Authentication (over SSL/TLS) or WS-Security Header with Enterprise Credentials (`ClientID`, `UserId`, `Password`).

### Integration Steps
1. **XML Construction**: Construct a valid `UniversalShipment` XML schema representing the booking/shipment.
2. **Envelope Wrapping**: Wrap the payload in an eAdaptor messaging wrapper containing metadata header tags.
3. **HTTP Post**: Send an HTTP POST request to the target organization's eAdaptor endpoint (`https://<partner_instance>.cargowise.com/eAdaptor/eAdaptorInboundService.svc`).
4. **Acknowledgement Handling**: Receive a synchronous `UniversalResponse` XML indicating success or schema validation errors.

---

## 2. Payload Data Required for Shipment / Booking Ingestion

CargoWise uses a comprehensive XML structure called `UniversalShipment`. Below are the core fields required to create a new forwarder booking or shipment record.

### Key Data Fields

| Data Section | Field Path / Element | Description / Requirements |
| :--- | :--- | :--- |
| **Sender/Recipient Info** | `<SenderID>`, `<RecipientID>` | Identifies originating system and targeted CargoWise company code |
| **Shipment Header** | `<DataContext><EnterpriseID>` | CargoWise Enterprise code (e.g., `ABC`) |
| | `<DataContext><CompanyCode>` | Branch/Company code (e.g., `ABCSIN`) |
| | `<DataContext><ServerID>` | Target server instance ID |
| **Parties** | `<OrganizationAddressCollection>` | List of addresses keyed by Type (`Shipper`, `Consignee`, `Carrier`, `Broker`) |
| **Routing Details** | `<PortOfLoading>` | UN/LOCODE or 3-letter port code |
| | `<PortOfDischarge>` | UN/LOCODE or 3-letter port code |
| | `<TransportMode>` | Transport code (`SEA`, `AIR`, `ROA`, `BULK`) |
| | `<VesselName>`, `<VoyageFlightNo>` | Name of vessel and voyage identifier |
| **Cargo & Packing** | `<PackingLineCollection>` | Items breakdown: weight, volume, package count, marks & numbers |
| **Custom References** | `<AdditionalReferenceCollection>` | External reference numbers (e.g., Customer PO Number, App Order ID) |

---

## 3. Sample XML Request Payload (UniversalShipment Snippet)

```xml
<UniversalShipment xmlns="http://www.wisetechglobal.com/eAdaptor/2011/03">
  <Shipment>
    <DataContext>
      <EnterpriseID>LOG</EnterpriseID>
      <ServerID>PROD</ServerID>
      <CompanyCode>LOGSGS</CompanyCode>
      <DataProvider>MY_EXTERNAL_APP</DataProvider>
    </DataContext>

    <LocalProcessing>
      <OrderNumber>APP-2026-9921</OrderNumber>
    </LocalProcessing>

    <TransportMode>
      <Code>SEA</Code>
    </TransportMode>

    <PortOfLoading>
      <Code>DEHAM</Code>
      <Name>Hamburg</Name>
    </PortOfLoading>

    <PortOfDischarge>
      <Code>SGSIN</Code>
      <Name>Singapore</Name>
    </PortOfDischarge>

    <VesselName>OCEAN GIANT</VesselName>
    <VoyageFlightNo>2026V01</VoyageFlightNo>

    <OrganizationAddressCollection>
      <OrganizationAddress>
        <AddressType>Shipper</AddressType>
        <CompanyName>Acme Export Ltd</CompanyName>
        <City>Hamburg</City>
        <Country>DE</Country>
      </OrganizationAddress>
      <OrganizationAddress>
        <AddressType>Consignee</AddressType>
        <CompanyName>Pacific Import Pte Ltd</CompanyName>
        <City>Singapore</City>
        <Country>SG</Country>
      </OrganizationAddress>
    </OrganizationAddressCollection>

    <PackingLineCollection>
      <PackingLine>
        <PackQty>10</PackQty>
        <PackType>
          <Code>PK</Code>
        </PackType>
        <Weight>12500.00</Weight>
        <WeightUnit>KG</WeightUnit>
        <Volume>35.00</Volume>
        <VolumeUnit>CBM</VolumeUnit>
        <GoodsDescription>Machinery Parts</GoodsDescription>
      </PackingLine>
    </PackingLineCollection>

  </Shipment>
</UniversalShipment>
```
