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

---

## 2. Authentication & Connection Protocol

Unlike a typical OAuth-fronted REST API, eAdaptor authenticates every call directly against Enterprise credentials issued by the target CargoWise instance - there is no separate bearer-token exchange step. Two supported connection methods:

### Option A: HTTP Basic Authentication (over TLS)

`POST https://<partner_instance>.cargowise.com/eAdaptor/eAdaptorInboundService.svc`

```http
Authorization: Basic base64(ClientID:UserId:Password)
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://www.wisetechglobal.com/eAdaptor/2011/03/SubmitShipment"
```

### Option B: WS-Security UsernameToken Header (SOAP)

```xml
<soapenv:Header>
  <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd">
    <wsse:UsernameToken>
      <wsse:Username>ClientID/UserId</wsse:Username>
      <wsse:Password Type="...#PasswordText">Password</wsse:Password>
    </wsse:UsernameToken>
  </wsse:Security>
</soapenv:Header>
```

### Successful Connection Indicator

There is no standalone "token response" - a successful connection is confirmed by the first `UniversalResponse` acknowledgement returned for a submitted message (see §5):

```xml
<UniversalResponse xmlns="http://www.wisetechglobal.com/eAdaptor/2011/03">
  <Status>SUCCESS</Status>
  <ReceivedDataContext>
    <EnterpriseID>LOG</EnterpriseID>
    <ServerID>PROD</ServerID>
    <CompanyCode>LOGSGS</CompanyCode>
  </ReceivedDataContext>
  <TransactionID>eAdaptor-20260914-004471</TransactionID>
</UniversalResponse>
```

Credentials, endpoint hostnames and company/server codes are issued per-partner by the CargoWise Enterprise administrator - there is no self-service sandbox signup.

---

## 3. eAdaptor Message Operations

eAdaptor is message-type-oriented rather than REST-resource-oriented - each operation is a distinct XML document type submitted to the same inbound endpoint (or received at your own outbound listener), not a separate URL path per action.

| Operation | Direction | Message Type | Description |
| :--- | :--- | :--- | :--- |
| **Create / Update Shipment** | Inbound (App → CargoWise) | `UniversalShipment` | Submits a new booking or shipment record, or updates an existing one by `OrderNumber` / `ForwardingNumber`. |
| **Acknowledge Submission** | Outbound (CargoWise → App) | `UniversalResponse` | Synchronous ack for an inbound message - success, or schema/business-rule validation errors. |
| **Status / Milestone Event** | Outbound (CargoWise → App) | `UniversalEvent` | Raised when a tracked milestone changes (booking confirmed, vessel departed/arrived, customs cleared, etc.). |
| **Cancel / Void Shipment** | Inbound (App → CargoWise) | `UniversalShipment` (`IsShipmentCancelled = true`) | Same message type as create/update, with the cancellation flag set. |

---

## 4. Full XML Payload Request Template (UniversalShipment)

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

    <AdditionalReferenceCollection>
      <AdditionalReference>
        <Type>
          <Code>CUR</Code>
          <Description>Customer Reference</Description>
        </Type>
        <ReferenceNumber>PO-88213</ReferenceNumber>
      </AdditionalReference>
    </AdditionalReferenceCollection>

  </Shipment>
</UniversalShipment>
```

---

## 5. Data Field Dictionary

### DataContext / Header Fields

| Field Path | Type | Required | Description / Allowed Values |
| :--- | :--- | :--- | :--- |
| `DataContext.EnterpriseID` | String | Yes | CargoWise Enterprise code issued to the partner (e.g. `LOG`). |
| `DataContext.ServerID` | String | Yes | Target server instance (e.g. `PROD`, `UAT`). |
| `DataContext.CompanyCode` | String | Yes | Branch/company code within the Enterprise (e.g. `LOGSGS`). |
| `DataContext.DataProvider` | String | No | Identifies the originating external system for audit/support purposes. |
| `LocalProcessing.OrderNumber` | String | Yes | Unique ID generated by your application (max 35 chars). |

### Parties Fields (`OrganizationAddressCollection`)

| Field Path | Type | Required | Description / Allowed Values |
| :--- | :--- | :--- | :--- |
| `OrganizationAddress.AddressType` | String | Yes | Enum: `Shipper`, `Consignee`, `Carrier`, `Broker`, `NotifyParty`. |
| `OrganizationAddress.CompanyName` | String | Yes | Legal name of the party. |
| `OrganizationAddress.City` / `.Country` | String | Yes | City and ISO 3166-1 alpha-2 country code. |

### Routing & Cargo Fields

| Field Path | Type | Required | Description / Allowed Values |
| :--- | :--- | :--- | :--- |
| `TransportMode.Code` | String | Yes | Enum: `SEA`, `AIR`, `ROA` (road), `RAI` (rail), `BULK`. |
| `PortOfLoading.Code` / `PortOfDischarge.Code` | String | Yes | UN/LOCODE or CargoWise 5-character port code (e.g. `DEHAM`, `SGSIN`). |
| `VesselName` / `VoyageFlightNo` | String | No | Name of the assigned vessel and its voyage identifier. |
| `PackingLineCollection.PackingLine.Weight` | Decimal | Yes | Gross weight value. |
| `PackingLineCollection.PackingLine.WeightUnit` | String | Yes | Enum: `KG`, `LB`, `MT`. |
| `PackingLineCollection.PackingLine.GoodsDescription` | String | Yes | Clear textual description of cargo. |
| `AdditionalReferenceCollection.AdditionalReference.ReferenceNumber` | String | No | External reference (customer PO number, app order ID, etc.), tagged with a `Type.Code`. |

---

## 6. Outbound Event Notification (`UniversalEvent`)

Once a tracked milestone changes inside CargoWise (booking confirmed, vessel departed/arrived, customs cleared), an outbound `UniversalEvent` message is sent to the endpoint agreed with the partner during onboarding.

### Request Sent by CargoWise to the Partner Endpoint
`POST https://api.yourapp.com/cargowise/events`

#### HTTP Headers
```http
Content-Type: text/xml; charset=utf-8
SOAPAction: "http://www.wisetechglobal.com/eAdaptor/2011/03/UniversalEvent"
```
(Endpoint-level authentication - shared credentials or mutual TLS - is agreed per integration; CargoWise does not publish a standard payload-signature scheme the way a webhook-native platform would.)

#### `UniversalEvent` XML Payload
```xml
<UniversalEvent xmlns="http://www.wisetechglobal.com/eAdaptor/2011/03">
  <Event>
    <DataContext>
      <EnterpriseID>LOG</EnterpriseID>
      <ServerID>PROD</ServerID>
      <CompanyCode>LOGSGS</CompanyCode>
    </DataContext>
    <EventType>
      <Code>VESSEL_DEPARTED</Code>
      <Description>Vessel Departed Port of Loading</Description>
    </EventType>
    <EventDateTime>2026-10-05T12:00:00Z</EventDateTime>
    <Shipment>
      <LocalProcessing>
        <OrderNumber>APP-2026-9921</OrderNumber>
      </LocalProcessing>
      <VesselName>OCEAN GIANT</VesselName>
      <VoyageFlightNo>2026V01</VoyageFlightNo>
      <PortOfLoading>
        <Code>DEHAM</Code>
      </PortOfLoading>
    </Shipment>
    <Remarks>Departed on schedule per carrier confirmation.</Remarks>
  </Event>
</UniversalEvent>
```

The receiving application should reply with a synchronous `UniversalResponse` (see §2) acknowledging receipt.
