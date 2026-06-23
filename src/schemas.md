# FW3 Data Model

Entity-relationship diagrams for the **fw3** database, generated from
`apps/api/prisma/schema.prisma`. The model is multi-tenant (almost every table
carries a `tenantId` and hangs off `Tenant`) and targets SQL Server. Money and
quantities are always `DECIMAL` (never float); primary keys are app-generated
UUIDs stored as `CHAR(36)`.

A few conventions worth knowing while reading these:

- **Enums are strings + CHECK constraints.** SQL Server has no native enum, so
  fields like `itemType`, `qcStatus`, or `status` are `NVarChar` columns
  constrained in the migration SQL. The allowed values are noted in comments.
- **No cascading deletes.** Every foreign key is `onDelete: NoAction`.
- **`state` columns** are mapped to `status` in the Prisma client (INV / WIP /
  QUARANTINE buckets).
- **Position vs. catalogue.** Master rows (`InventoryItem`, `Container`) hold no
  quantity; the on-hand position and moving-average cost live in a separate stock
  table (`ItemStock` / `ContainerStock`) and an append-only ledger
  (`InventoryTxn` / `ContainerTxn`).

The diagrams are split by domain to stay readable. The
[domain overview](#domain-overview) shows how the domains connect; `Tenant` is
omitted from the per-domain diagrams since it owns nearly everything.

## Domain overview

```mermaid
erDiagram
  Tenant ||--o{ User : "has"
  Tenant ||--o{ InventoryItem : "owns catalogue"
  Tenant ||--o{ Vendor : "buys from"
  Tenant ||--o{ Customer : "sells to"
  Tenant ||--o{ Location : "stores at"

  Tenant ||--o{ Container : "packs in"
  Tenant ||--o{ BusinessVariableValue : "configures"

  InventoryItem ||--o{ Formula : "is finished good of"
  InventoryItem ||--o{ ItemStock : "has position"
  InventoryItem ||--o{ InventoryTxn : "moves via ledger"
  InventoryItem ||--o{ ReceivedLot : "received/produced as"
  InventoryItem ||--o{ IfraCategoryLimit : "regulated by"
  InventoryItem ||--o| FgRegulatoryProfile : "FG profile"

  Vendor ||--o{ PurchaseOrder : "receives"
  PurchaseOrder ||--o{ ReceivedLot : "lands as lots"
  Customer ||--o{ SalesOrder : "places"
  SalesOrder ||--o{ Shipment : "fulfilled by"
  SalesOrder ||--o{ ProductionWorkOrder : "scheduled as"

  Formula ||--o{ ProductionWorkOrder : "drives"
  ProductionWorkOrder ||--o{ CompounderPour : "consumed by"
  ProductionWorkOrder ||--o{ PurchasingAlert : "flags shortage"

  ReceivedLot ||--o{ QualityTestResult : "QC'd by"
  Container ||--o{ ContainerTxn : "moves via ledger"
  Location ||--o{ ItemStockLocation : "holds stock"
  InventoryItem ||--o{ CycleCountLine : "counted in"
  Tenant ||--o{ QbConnection : "syncs to QuickBooks"
  Tenant ||--o{ AuditLog : "records"
```

## Tenancy & access control

Tenant isolation plus a role-based permission system. `Permission` is the only
globally-scoped table — permission keys (e.g. `inventory:read`) are app
constants shared across tenants. `RolePermission` and `UserRole` are the
many-to-many join tables.

```mermaid
erDiagram
  Tenant ||--o{ User : "has"
  Tenant ||--o{ Role : "defines"
  User ||--o{ UserRole : ""
  Role ||--o{ UserRole : ""
  Role ||--o{ RolePermission : ""
  Permission ||--o{ RolePermission : ""

  Tenant {
    uuid id PK
    string name
    string slug UK
    datetime createdAt
    datetime updatedAt
  }
  User {
    uuid id PK
    uuid tenantId FK
    string idpSub "OIDC subject (Entra)"
    string email
    string displayName
    bool isActive
  }
  Role {
    uuid id PK
    uuid tenantId FK
    string name "unique per tenant"
  }
  Permission {
    uuid id PK
    string key UK "e.g. inventory:read"
  }
  RolePermission {
    uuid roleId PK, FK
    uuid permissionId PK, FK
  }
  UserRole {
    uuid userId PK, FK
    uuid roleId PK, FK
  }
```

## Item master & formulas

`InventoryItem` is the central catalogue row — the QuickBooks-ready item master.
It deliberately holds **no inventory position**; quantity and moving-average cost
live in `ItemStock` and the ledger. `Formula` defines a finished good as a set of
raw materials by percentage of weight (`FormulaLine`); lines for a formula must
sum to 100. `ItemQualitySpec` carries the per-item QC acceptance criteria.

Regulatory data hangs off the item: `IfraCategoryLimit` records, per raw
material, the maximum percentage it may reach in a finished product of a given
IFRA use category (49th Amendment). For finished goods, `FgRegulatoryProfile` is
the FormPak+ snapshot (the third-party data we can't derive in-house) with its
per-category `FgIfraLevel` rows. `reorderPoint` flags replenishment;
`productionUse` hides R&D/lab-only materials from the compounder dosing tool.

```mermaid
erDiagram
  InventoryItem ||--o{ ItemQualitySpec : "has QC specs"
  InventoryItem ||--o{ Formula : "is finished good of"
  Formula ||--o{ FormulaLine : "has lines"
  InventoryItem ||--o{ FormulaLine : "used as raw material in"
  InventoryItem ||--o{ IfraCategoryLimit : "RM IFRA limits"
  InventoryItem ||--o| FgRegulatoryProfile : "FG regulatory profile"
  FgRegulatoryProfile ||--o{ FgIfraLevel : "IFRA levels"

  InventoryItem {
    uuid id PK
    uuid tenantId FK
    string sku "unique per tenant"
    string name
    string itemType "RAW_MATERIAL | FINISHED_GOOD"
    string physicalForm "LIQUID | SOLID"
    string unitOfMeasure "LB | KG"
    decimal salesPrice
    decimal reorderPoint "null = untracked"
    string qbItemType "INVENTORY | NON_INVENTORY | SERVICE"
    decimal standardCost
    string purchaseDescription
    string incomeAccount "QB account"
    string cogsAccount "QB account"
    string assetAccount "QB account"
    string qbListId "QBWC linkage"
    string qbEditSequence
    datetime qbSyncedAt
    bool productionUse "RM: dosable on compounder tool"
    string casNumber "RM"
    decimal flashPointC "RM: closed-cup °C"
    string prop65Status "UNKNOWN | NOT_LISTED | LISTED"
    bool active
  }
  ItemQualitySpec {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string testType "SPECIFIC_GRAVITY | REFRACTIVE_INDEX | GARDNER_COLOR | ODOR | APPEARANCE | MELTING_POINT"
    decimal minValue "numeric tests"
    decimal maxValue "numeric tests"
    string expectedValue "judgment tests"
  }
  IfraCategoryLimit {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string category "1..12 incl. 5A/7B/10A; unique per item"
    decimal maxPercent "max % in finished product, 0-100"
  }
  FgRegulatoryProfile {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK "unique (one per FG)"
    decimal flashPointC
    string complianceStatus "UNKNOWN | COMPLIANT | NON_COMPLIANT | PENDING"
    string allergenDeclaration
    string certificateUrl
    string formPakRef
    datetime syncedAt
  }
  FgIfraLevel {
    uuid id PK
    uuid tenantId FK
    uuid profileId FK
    string category "unique per profile"
    decimal maxPercent
  }
  Formula {
    uuid id PK
    uuid tenantId FK
    uuid finishedGoodId FK
    string name
    int version "unique per (tenant, FG, version)"
    bool isActive
  }
  FormulaLine {
    uuid id PK
    uuid formulaId FK
    uuid rawMaterialId FK
    decimal percentage "0 < pct <= 100; lines sum to 100"
    int sortOrder
  }
```

## Inventory position, locations & ledger

The authoritative on-hand picture. `ItemStock` is the per-(item, state) position
(INV = lot-traceable, WIP = work-in-progress). `Location` is a typed tree
(BUILDING → AISLE → RACK, plus AREA nodes), and `ItemStockLocation` breaks an
item's quantity down by location — the sum over locations for an (item, state)
equals the `ItemStock` row. `InventoryTxn` is the append-only stock ledger:
every quantity change is one signed line carrying the running balance and
weighted-average cost. `LocationMove` is the parallel ledger for physical moves
between locations.

```mermaid
erDiagram
  InventoryItem ||--o{ ItemStock : "position per state"
  InventoryItem ||--o{ ItemStockLocation : "located qty"
  InventoryItem ||--o{ InventoryTxn : "ledger lines"
  InventoryItem ||--o{ LocationMove : "moved"
  Location ||--o{ ItemStockLocation : "holds"
  Location ||--o{ Location : "parent of"
  Location ||--o{ LocationMove : "from"
  Location ||--o{ LocationMove : "to"

  ItemStock {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string status "INV | WIP (db: state)"
    decimal quantity
    decimal avgCost
  }
  Location {
    uuid id PK
    uuid tenantId FK
    string kind "BUILDING | AISLE | RACK | AREA"
    uuid parentId FK
    uuid buildingId FK "building ancestor"
    string code "full address e.g. 075-A-100"
    string segment "own part e.g. 075"
    string side "LEFT | RIGHT (racks)"
    bool isDefault
    bool isReceiving
    bool active
  }
  ItemStockLocation {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string status "INV | QUARANTINE (db: state)"
    uuid locationId FK
    decimal quantity
  }
  InventoryTxn {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string type "RECEIPT | CONSUME | PRODUCTION_OUTPUT | SHIPMENT | ADJUSTMENT | TRANSFER | SCRAP | RETURN"
    string status "INV | WIP (db: state)"
    decimal quantity "signed: + in, - out"
    decimal unitCost
    decimal value "signed = qty * unitCost"
    decimal balanceQty "on-hand after line"
    decimal balanceAvgCost "moving avg after line"
    string docType "PURCHASE_ORDER | PRODUCTION_RUN | SALES_ORDER | ADJUSTMENT"
    uuid docId
    datetime occurredAt
  }
  LocationMove {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string status "INV | QUARANTINE (db: state)"
    uuid fromLocationId FK
    uuid toLocationId FK
    decimal quantity
    uuid actorId
    datetime occurredAt
  }
```

## Lots, quality control, scrap & vendor returns

A `ReceivedLot` is a lot of material pending or past QC — origin `RECEIPT`
(from a vendor PO) or `PRODUCTION` (from a work order). Source references
(vendor, PO, work order) are snapshotted by name for history. `QualityTestResult`
records each acceptance test against the lot. `ScrapRecord` is a write-off (with
a structured reason) and `VendorReturn` is a return-to-vendor of QC-failed raw
material — both pair with a matching `InventoryTxn` (`SCRAP` / `RETURN`).

```mermaid
erDiagram
  InventoryItem ||--o{ ReceivedLot : "lot of"
  Location ||--o{ ReceivedLot : "quarantined at"
  ReceivedLot ||--o{ QualityTestResult : "results"
  ReceivedLot ||--o{ VendorReturn : "returned via"
  InventoryItem ||--o{ ScrapRecord : "scrapped"
  InventoryItem ||--o{ VendorReturn : "returned"
  Location ||--o{ ScrapRecord : "from"
  User ||--o{ ScrapRecord : "operator"
  User ||--o{ VendorReturn : "operator"

  ReceivedLot {
    uuid id PK
    uuid tenantId FK
    string origin "RECEIPT | PRODUCTION"
    uuid itemId FK
    uuid purchaseOrderId "snapshot ref"
    string purchaseOrderNumber "snapshot"
    string vendorName "snapshot"
    uuid sourceWorkOrderId "snapshot ref"
    string workOrderNumber "snapshot"
    string supplierLotNumber
    uuid locationId FK
    decimal quantity
    decimal packedQty "production lots"
    decimal returnedQty "QC-fail returns"
    decimal unitCost
    string qcStatus "PENDING | APPROVED | REJECTED | RETURNED"
    string rejectionReason
    datetime receivedAt
    uuid reviewedById "QC reviewer"
  }
  QualityTestResult {
    uuid id PK
    uuid receivedLotId FK
    string testType
    string measuredValue
    bool passed
    string notes
  }
  ScrapRecord {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    string status "INV | WIP | QUARANTINE (db: state)"
    uuid locationId FK
    decimal quantity
    decimal value "loss at avg cost"
    string reason "DAMAGED | EXPIRED | CONTAMINATED | SPILL | QC_FAILED | OTHER"
    uuid operatorId FK
    datetime occurredAt
  }
  VendorReturn {
    uuid id PK
    uuid tenantId FK
    uuid receivedLotId FK
    uuid itemId FK
    string vendorName "snapshot"
    string purchaseOrderNumber "snapshot"
    decimal quantity
    decimal unitCost
    decimal value "recoverable amount"
    string rmaNumber
    uuid operatorId FK
    datetime occurredAt
  }
```

## Purchasing

Vendors and their purchase orders. `Vendor` carries tax/payment-term details and
has any number of addresses and contacts (scoped through the vendor, no
`tenantId` of their own). The `suppliesMaterials` / `suppliesContainers` flags
drive which subjects the PO page offers. A `PurchaseOrderLine` buys **either** an
inventory item or a container (exactly one; CHECK) and tracks ordered vs.
received quantity.

```mermaid
erDiagram
  Vendor ||--o{ VendorAddress : "has"
  Vendor ||--o{ VendorContact : "has"
  Vendor ||--o{ PurchaseOrder : "issued"
  PurchaseOrder ||--o{ PurchaseOrderLine : "has lines"
  InventoryItem ||--o{ PurchaseOrderLine : "ordered as"
  Container ||--o{ PurchaseOrderLine : "ordered as"

  Vendor {
    uuid id PK
    uuid tenantId FK
    string name "unique per tenant"
    string code
    string email
    string taxId
    string paymentTerms "DUE_ON_RECEIPT | NET_15 | NET_30 | ..."
    bool suppliesMaterials
    bool suppliesContainers
    bool isActive
  }
  VendorAddress {
    uuid id PK
    uuid vendorId FK
    string kind "BILLING | SHIPPING | REMIT_TO | OTHER"
    string line1
    string city
    string region
    string postalCode
    bool isPrimary
  }
  VendorContact {
    uuid id PK
    uuid vendorId FK
    string name
    string title
    string email
    string phone
    bool isPrimary
  }
  PurchaseOrder {
    uuid id PK
    uuid tenantId FK
    uuid vendorId FK
    string poNumber "unique per tenant"
    string status "OPEN | PARTIAL | RECEIVED | CANCELLED"
    datetime orderDate
  }
  PurchaseOrderLine {
    uuid id PK
    uuid purchaseOrderId FK
    uuid itemId FK "item OR container (exactly one)"
    uuid containerId FK
    decimal quantityOrdered
    decimal unitCost
    decimal quantityReceived
    int sortOrder
  }
```

## Sales

Mirrors purchasing. `Customer` adds a buy-volume `rating` (A–D) on top of the
shared contact/address structure. A `SalesOrderLine` sells **either** an
inventory item (`lineType = ITEM`) or a container itself
(`lineType = CONTAINER`, via `productContainerId`); ITEM lines may carry a
packing plan (`containerId` + `containerQuantity`). `SalesOrder` tracks the
committed `requestedShipDate` (drives scheduler sequencing), `paidAt` (net-terms
customers may request production unpaid), and `packedAt`. Shipments and
SO-linked production work orders hang off the order — see
[Shipments](#shipments) and [Production](#production).

```mermaid
erDiagram
  Customer ||--o{ CustomerAddress : "has"
  Customer ||--o{ CustomerContact : "has"
  Customer ||--o{ SalesOrder : "placed"
  SalesOrder ||--o{ SalesOrderLine : "has lines"
  InventoryItem ||--o{ SalesOrderLine : "sold as"
  Container ||--o{ SalesOrderLine : "sold / packed as"

  Customer {
    uuid id PK
    uuid tenantId FK
    string name "unique per tenant"
    string code
    string email
    string taxId
    string paymentTerms "DUE_ON_RECEIPT | NET_15 | ..."
    string rating "A | B | C | D (buy volume)"
    string qbListId "QBWC linkage"
    string qbEditSequence
    datetime qbSyncedAt
    bool isActive
  }
  CustomerAddress {
    uuid id PK
    uuid customerId FK
    string kind "BILLING | SHIPPING | REMIT_TO | OTHER"
    string line1
    string city
    string region
    string postalCode
    bool isPrimary
  }
  CustomerContact {
    uuid id PK
    uuid customerId FK
    string name
    string title
    string email
    bool isPrimary
  }
  SalesOrder {
    uuid id PK
    uuid tenantId FK
    uuid customerId FK
    string soNumber "unique per tenant"
    string status "OPEN | PARTIAL | SHIPPED | CANCELLED"
    datetime orderDate
    datetime requestedShipDate "null = unset"
    datetime paidAt "null until paid"
    datetime packedAt "null until packed"
  }
  SalesOrderLine {
    uuid id PK
    uuid salesOrderId FK
    string lineType "ITEM | CONTAINER"
    uuid itemId FK "ITEM lines"
    uuid productContainerId FK "CONTAINER lines"
    decimal quantityOrdered
    decimal unitPrice
    decimal quantityShipped
    int sortOrder
    uuid containerId FK "packing plan (ITEM lines)"
    decimal containerQuantity
  }
```

## Shipments

A `Shipment` is a first-class physical despatch against a sales order — its own
number, carrier, and tracking. Partial fulfilment produces several shipments per
order. Each `ShipmentLine` mirrors the SO line it fulfils and **snapshots
`unitCost` at ship time** for COGS; the matching `SHIPMENT` ledger lines reduce
stock alongside.

```mermaid
erDiagram
  SalesOrder ||--o{ Shipment : "despatched as"
  Shipment ||--o{ ShipmentLine : "has lines"
  SalesOrderLine ||--o{ ShipmentLine : "fulfilled by"
  InventoryItem ||--o{ ShipmentLine : "shipped as"
  Container ||--o{ ShipmentLine : "shipped as"
  User ||--o{ Shipment : "shipped by"

  Shipment {
    uuid id PK
    uuid tenantId FK
    uuid salesOrderId FK
    string shipmentNumber "unique per tenant"
    string carrier
    string trackingNumber
    uuid shippedById FK
    datetime shippedAt
  }
  ShipmentLine {
    uuid id PK
    uuid tenantId FK
    uuid shipmentId FK
    uuid salesOrderLineId FK
    string lineType "ITEM | CONTAINER"
    uuid itemId FK
    uuid containerId FK
    decimal quantity
    decimal unitCost "COGS per unit at ship time"
    decimal value "COGS total"
  }
```

## Containers (packaging)

Packaging stock — drums, pails, jugs, cans, bottles, totes — that fragrance is
packed into, and that can also be sold or ordered on its own. The
`Container`/`ContainerStock`/`ContainerTxn` trio mirrors the
item/stock/ledger split, but there is a **single on-hand bucket** (no WIP or QC
staging) and quantities are whole counts. `capacityLb` is the nominal fill weight
used to default packing counts.

```mermaid
erDiagram
  Container ||--o| ContainerStock : "position"
  Container ||--o{ ContainerTxn : "ledger lines"

  Container {
    uuid id PK
    uuid tenantId FK
    string sku "unique per tenant"
    string name
    string containerType "DRUM | PAIL | JUG | CAN | BOTTLE | TOTE | OTHER"
    decimal capacityLb "nominal fill weight"
    decimal standardCost
    decimal reorderPoint "whole count; null = untracked"
    bool active
  }
  ContainerStock {
    uuid id PK
    uuid tenantId FK
    uuid containerId FK "unique (one per container)"
    decimal quantity "whole count"
    decimal avgCost "moving average"
  }
  ContainerTxn {
    uuid id PK
    uuid tenantId FK
    uuid containerId FK
    string type "ADJUSTMENT | CONSUME | SCRAP"
    decimal quantity "signed: + in, - out"
    decimal unitCost
    decimal value "signed = qty * unitCost"
    decimal balanceQty "on-hand after line"
    decimal balanceAvgCost "moving avg after line"
    string reason "scrap reason when type = SCRAP"
    uuid operatorId
    datetime occurredAt
  }
```

## Production

A production work order makes a target item from a formula. Flow:
`PLANNED → STAGED → IN_PROGRESS → COMPLETED` (components move INV → WIP on
staging, are consumed from WIP, and output lands in FG-WIP before a separate
pack-off step). `CompounderPour` is the append-only record of each dose an
operator reports from the compounder dosing tool; every pour also posts a
`CONSUME` ledger line. (Physical tables retain their original `ProductionRun`
names via `@@map`.) A work order may be linked to the sales order/line it
fulfils (`salesOrderId` / `salesOrderLineId`; null for ad-hoc runs) and ordered
in the scheduler queue via `queuePosition`. When the scheduler can't source a
component, it raises a `PurchasingAlert` for purchasing to act on.

```mermaid
erDiagram
  Formula ||--o{ ProductionWorkOrder : "drives"
  InventoryItem ||--o{ ProductionWorkOrder : "target of"
  SalesOrder ||--o{ ProductionWorkOrder : "fulfilled by"
  ProductionWorkOrder ||--o{ ProductionWorkOrderLine : "has components"
  InventoryItem ||--o{ ProductionWorkOrderLine : "component of"
  ProductionWorkOrder ||--o{ CompounderPour : "poured into"
  ProductionWorkOrderLine ||--o{ CompounderPour : "doses"
  InventoryItem ||--o{ CompounderPour : "component poured"
  User ||--o{ CompounderPour : "operator"
  ProductionWorkOrder ||--o{ PurchasingAlert : "shortage flagged"
  InventoryItem ||--o{ PurchasingAlert : "short of"

  ProductionWorkOrder {
    uuid id PK
    uuid tenantId FK
    string workOrderNumber "db: runNumber; unique per tenant"
    uuid targetItemId FK
    uuid formulaId FK
    decimal batchSize "in batchUnit"
    string batchUnit "LB | KG"
    decimal outputQty "target's UoM"
    string status "REQUESTED | QUEUED | PLANNED | STAGED | IN_PROGRESS | ON_HOLD | COMPLETED | CANCELLED"
    uuid salesOrderId FK "null for ad-hoc"
    uuid salesOrderLineId FK
    int queuePosition "QUEUED only"
  }
  ProductionWorkOrderLine {
    uuid id PK
    uuid productionWorkOrderId FK "db: productionRunId"
    uuid componentId FK
    decimal requiredQty
    decimal stagedQty
    decimal consumedQty
    int sortOrder
  }
  CompounderPour {
    uuid id PK
    uuid tenantId FK
    uuid productionWorkOrderId FK "db: productionRunId"
    uuid workOrderLineId FK "db: productionRunLineId"
    uuid componentId FK
    decimal quantity "canonical pounds"
    uuid operatorId FK
    datetime occurredAt
  }
  PurchasingAlert {
    uuid id PK
    uuid tenantId FK
    uuid itemId FK
    uuid workOrderId FK "nullable"
    decimal shortQty
    string status "OPEN | RESOLVED"
    uuid raisedById FK
    datetime resolvedAt
  }
```

## Cycle counts

Verifying physical inventory against the system, then posting variances as
`ADJUSTMENT` ledger lines. A `CycleCount` is scoped to a location (or the whole
tenant when `scopeLocationId` is null) and can be **blind** (system quantity
hidden from the counter). Each `CycleCountLine` is one (item, state, location)
cell: `expectedQty` is the snapshot at creation, `countedQty` is what was found.

```mermaid
erDiagram
  CycleCount ||--o{ CycleCountLine : "has lines"
  Location ||--o{ CycleCount : "scope"
  Location ||--o{ CycleCountLine : "at"
  InventoryItem ||--o{ CycleCountLine : "counted"
  User ||--o{ CycleCount : "created by"
  User ||--o{ CycleCount : "completed by"

  CycleCount {
    uuid id PK
    uuid tenantId FK
    string reference "unique per tenant"
    string status "OPEN | COMPLETED | CANCELLED"
    bool blind
    uuid scopeLocationId FK "null = whole tenant"
    uuid createdById FK
    uuid completedById FK
    datetime completedAt
  }
  CycleCountLine {
    uuid id PK
    uuid tenantId FK
    uuid cycleCountId FK
    uuid itemId FK
    string status "INV | QUARANTINE (db: state)"
    uuid locationId FK
    decimal expectedQty "system snapshot"
    decimal countedQty
    bool counted
  }
```

## Business configuration

Tenant-tunable settings. `BusinessVariableValue` stores **only the overrides** of
a code-defined variable catalog (working hours, default profit margin, pph per
workstation, production efficiency, production cost factor); unset entries fall
back to their catalog defaults. `operatorRole` is null for non-role-scoped
variables, or one row per role for role-scoped ones. `CompanyHoliday` encodes
closure days as recurrence rules so one row covers every year.

```mermaid
erDiagram
  Tenant ||--o{ BusinessVariableValue : "overrides"
  Tenant ||--o{ CompanyHoliday : "closes on"

  BusinessVariableValue {
    uuid id PK
    uuid tenantId FK
    string key "catalog key; unique per (tenant, key, role)"
    string operatorRole "FLOOR | LAB | SAMPLE_LAB; null = global"
    string value "text, validated by catalog type"
  }
  CompanyHoliday {
    uuid id PK
    uuid tenantId FK
    string name
    string ruleType "FIXED | NTH_WEEKDAY | EXPLICIT"
    int month "FIXED, NTH_WEEKDAY"
    int day "FIXED"
    int weekday "0-6 (NTH_WEEKDAY)"
    int nth "1-5 or -1=last (NTH_WEEKDAY)"
    date date "EXPLICIT one-off"
    bool active
  }
```

## QuickBooks sync & audit

Per-tenant QuickBooks Web Connector connections and their in-flight sync
sessions, plus the tenant-wide audit log (before/after JSON snapshots of every
CREATE/UPDATE/DELETE/SYNC).

```mermaid
erDiagram
  Tenant ||--o{ QbConnection : "has"
  QbConnection ||--o{ QbwcSession : "runs"
  Tenant ||--o{ AuditLog : "records"
  User ||--o{ AuditLog : "actor"

  QbConnection {
    uuid id PK
    uuid tenantId FK
    string name
    string qbwcUsername UK
    string qbwcPasswordHash "argon2"
    string companyFile
    datetime lastSyncAt
  }
  QbwcSession {
    uuid id PK
    uuid connectionId FK
    string ticket UK
    string status "OPEN | DONE | ERROR"
    string lastError
  }
  AuditLog {
    uuid id PK
    uuid tenantId FK
    uuid actorId FK "null for system/QBWC"
    string entityType
    string entityId
    string action "CREATE | UPDATE | DELETE | SYNC"
    string before "JSON snapshot"
    string after "JSON snapshot"
    datetime createdAt
  }
```
