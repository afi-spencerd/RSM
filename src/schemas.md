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

  InventoryItem ||--o{ Formula : "is finished good of"
  InventoryItem ||--o{ ItemStock : "has position"
  InventoryItem ||--o{ InventoryTxn : "moves via ledger"
  InventoryItem ||--o{ ReceivedLot : "received/produced as"

  Vendor ||--o{ PurchaseOrder : "receives"
  PurchaseOrder ||--o{ ReceivedLot : "lands as lots"
  Customer ||--o{ SalesOrder : "places"

  Formula ||--o{ ProductionWorkOrder : "drives"
  ProductionWorkOrder ||--o{ CompounderPour : "consumed by"

  ReceivedLot ||--o{ QualityTestResult : "QC'd by"
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

```mermaid
erDiagram
  InventoryItem ||--o{ ItemQualitySpec : "has QC specs"
  InventoryItem ||--o{ Formula : "is finished good of"
  Formula ||--o{ FormulaLine : "has lines"
  InventoryItem ||--o{ FormulaLine : "used as raw material in"

  InventoryItem {
    uuid id PK
    uuid tenantId FK
    string sku "unique per tenant"
    string name
    string itemType "RAW_MATERIAL | FINISHED_GOOD"
    string physicalForm "LIQUID | SOLID"
    string unitOfMeasure "LB | KG"
    decimal salesPrice
    string qbItemType "INVENTORY | NON_INVENTORY | SERVICE"
    decimal standardCost
    string incomeAccount "QB account"
    string cogsAccount "QB account"
    string assetAccount "QB account"
    string qbListId "QBWC linkage"
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
    string supplierLotNumber
    uuid locationId FK
    decimal quantity
    decimal packedQty "production lots"
    decimal returnedQty "QC-fail returns"
    decimal unitCost
    string qcStatus "PENDING | APPROVED | REJECTED | RETURNED"
    string rejectionReason
    datetime receivedAt
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
`tenantId` of their own). `PurchaseOrderLine` tracks ordered vs. received
quantity per item.

```mermaid
erDiagram
  Vendor ||--o{ VendorAddress : "has"
  Vendor ||--o{ VendorContact : "has"
  Vendor ||--o{ PurchaseOrder : "issued"
  PurchaseOrder ||--o{ PurchaseOrderLine : "has lines"
  InventoryItem ||--o{ PurchaseOrderLine : "ordered as"

  Vendor {
    uuid id PK
    uuid tenantId FK
    string name "unique per tenant"
    string code
    string email
    string taxId
    string paymentTerms "DUE_ON_RECEIPT | NET_15 | NET_30 | ..."
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
    uuid itemId FK
    decimal quantityOrdered
    decimal unitCost
    decimal quantityReceived
    int sortOrder
  }
```

## Sales

Mirrors purchasing. `Customer` adds a buy-volume `rating` (A–D) on top of the
shared contact/address structure; `SalesOrderLine` tracks ordered vs. shipped.

```mermaid
erDiagram
  Customer ||--o{ CustomerAddress : "has"
  Customer ||--o{ CustomerContact : "has"
  Customer ||--o{ SalesOrder : "placed"
  SalesOrder ||--o{ SalesOrderLine : "has lines"
  InventoryItem ||--o{ SalesOrderLine : "sold as"

  Customer {
    uuid id PK
    uuid tenantId FK
    string name "unique per tenant"
    string code
    string email
    string taxId
    string paymentTerms "DUE_ON_RECEIPT | NET_15 | ..."
    string rating "A | B | C | D (buy volume)"
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
  }
  SalesOrderLine {
    uuid id PK
    uuid salesOrderId FK
    uuid itemId FK
    decimal quantityOrdered
    decimal unitPrice
    decimal quantityShipped
    int sortOrder
  }
```

## Production

A production work order makes a target item from a formula. Flow:
`PLANNED → STAGED → IN_PROGRESS → COMPLETED` (components move INV → WIP on
staging, are consumed from WIP, and output lands in FG-WIP before a separate
pack-off step). `CompounderPour` is the append-only record of each dose an
operator reports from the compounder dosing tool; every pour also posts a
`CONSUME` ledger line. (Physical tables retain their original `ProductionRun`
names via `@@map`.)

```mermaid
erDiagram
  Formula ||--o{ ProductionWorkOrder : "drives"
  InventoryItem ||--o{ ProductionWorkOrder : "target of"
  ProductionWorkOrder ||--o{ ProductionWorkOrderLine : "has components"
  InventoryItem ||--o{ ProductionWorkOrderLine : "component of"
  ProductionWorkOrder ||--o{ CompounderPour : "poured into"
  ProductionWorkOrderLine ||--o{ CompounderPour : "doses"
  InventoryItem ||--o{ CompounderPour : "component poured"
  User ||--o{ CompounderPour : "operator"

  ProductionWorkOrder {
    uuid id PK
    uuid tenantId FK
    string workOrderNumber "db: runNumber; unique per tenant"
    uuid targetItemId FK
    uuid formulaId FK
    decimal batchSize "in batchUnit"
    string batchUnit "LB | KG"
    decimal outputQty "target's UoM"
    string status "PLANNED | STAGED | IN_PROGRESS | ON_HOLD | COMPLETED | CANCELLED"
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
