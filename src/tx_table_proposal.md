# Tx Table Proposal

## Diagrams

```mermaid
---
title: "Current Schema"
---
erDiagram
  ordertbl
  formulainfo
  orderdetail
  whorder
  Fragrance
  whprep {
    int WHPrepID PK
    int FormulaInventory
    float PrepQty
  }
  whorderdetail {
    int WHOrderDetailID PK
    float QtyReceived
  }
  whprepdetail
  StorageLocation
  WHPrepStockDetail {
    int WHPrepStockDetailID PK
    int WHPrepID FK
    int OrderDetailID FK
    float QtyUsed
  }
  formula_stock_lot_adjustment {
    int FormulaStockLotAdjustmentID PK
    int WHPrepID FK
    float Qty
    string Comments
    date DATE
  }
  whprepdetail_qtydetail {
    int QtyDetailID PK
    int WHPrepDetailID FK
    int WHOrderDetailID FK
    float QtyUsed
    float QtyUsedBM
  }
  multi_StorageLocation
  WHAdjustment_Fragrance {
    int WHAdjustmentFragranceID PK
    int FragranceID FK
    int WHOrderDetailID FK
    float Qty
    int LocationId FK
    int MultiLocationID FK
    string Comments
    date DATE
  }
  whprep_StorageLocation_Lot

  ordertbl ||--|{ orderdetail : OrderID
  formulainfo ||--o{ orderdetail : FormulaInfoID
  orderdetail ||--o| whprep : OrderDetailID
  orderdetail ||--o{ WHPrepStockDetail: StockOrderDetailID
  whprep ||--|{ whprepdetail : WHPrepID
  whprep ||--o{ WHPrepStockDetail : WHPrepID
  whprep ||--o{ formula_stock_lot_adjustment : WhPrepID
  whorder ||--|{ whorderdetail : WHOrderID
  Fragrance ||--o{ whorderdetail : FragranceID
  Fragrance ||--o{ whprepdetail : FragranceID
  Fragrance ||--o{ WHAdjustment_Fragrance : FragranceID
  whorderdetail ||--o{ whprepdetail_qtydetail : Lot
  whorderdetail ||--o{ WHAdjustment_Fragrance : Lot
  whorderdetail ||--o{ multi_StorageLocation : Lot
  whprepdetail ||--o{ whprepdetail_qtydetail : WHPrepDetailID
  whprepdetail ||--o{ whprep_StorageLocation_Lot : WHPrepDetailID
  StorageLocation ||--o{ WHAdjustment_Fragrance : LocationID
  StorageLocation ||--o{ multi_StorageLocation : StorageLocationID
  StorageLocation ||--o{ whprep_StorageLocation_Lot : StorageLocationID

```

> [!WARNING]
> Current layout not showing Formula Stock Inventory?

```mermaid
---
title: "Proposed Schema"
---
erDiagram
  InventoryTx {
    int      tx_id  PK
    int      tenant_id FK
    int      item_id FK
    int      lot_id FK "NULL — proposed; lost in WIP"
    varchar  type "('RECEIPT','CONSUME','PRODUCTION_OUTPUT','SHIPMENT','ADJUSTMENT','TRANSFER')"
    varchar  state "('INV','WIP','QUARANTINE')"
    decimal  quantity "signed: + inbound, - outbound"
    decimal  unit_cost
    decimal  value "signed = quantity * unit_cost"
    decimal  balance_qty "on-hand after this line"
    decimal  balance_avg_cost "moving avg after this line"
    varchar  doc_type "('PURCHASE_ORDER','PRODUCTION_RUN','SALES_ORDER','ADJUSTMENT') NULL"
    int      doc_id "NULL"
    varchar  note "NULL"
    datetime occurred_at
  }
  Item {
    int      item_id PK
    varchar  kind "('RM', 'FG')"
    int      rm_id FK "NULL"
    int      fg_id FK "NULL"
  }
  Lot {
    int      lot_id PK
    int      item_id FK
    datetime created_at
  }

  formulainfo
  Fragrance
  StorageLocation
  whorderdetail {
    int WHOrderDetailID PK
    float QtyReceived
  }

  whorderdetail ||--|{ InventoryTx : "whorderdetail.WHOrderDetailID :: InventoryTx.lot_id"
  InventoryTx }o--|| Item : "InventoryTx.item_id :: Item.item_id"
  InventoryTx }o--o| Lot : "InventoryTx.lot_id :: Lot.lot_id (NULL in WIP)"
  Lot }o--|| Item : "Lot.item_id :: Item.item_id"
  Item ||..|| Fragrance : "Item.rm_id :: Fragrance.id"
  Item ||..|| formulainfo : "Item.fg_id :: formulainfo.id"
```

> [!NOTE]
> The flowcharts below are modeled on fw3's **two ledgers**. Rectangles are ledger
> writes — `InventoryTx:` lines carry a `type`, a `status` (`INV` / `WIP` /
> `QUARANTINE`) and a signed quantity; `LocationMove:` lines are physical relocations
> within a status. Rounded nodes are user/physical actions, diamonds are decisions,
> and triple-circle nodes are resting states. Physical `Location`s are real; system
> boundaries (Vendor, Customer, Scrap, Shipping) are **not** entities. A status
> transfer (e.g. `QUARANTINE` → `INV`) is two value-balanced `InventoryTx` lines, and
> lot identity (`ReceivedLot`) is tracked in `INV`/`QUARANTINE` but lost in the `WIP`
> blend.

```mermaid
---
title: Receiving to Inventory
---
flowchart TB
  start((Start))
  order_po("User sets PO to Ordered.")
  receive("User receives a PO line (RM arrives).")
  tx_receive["InventoryTx: RECEIPT, status QUARANTINE, +qty."]
  lot_receive["ReceivedLot created (origin RECEIPT, QC PENDING); placed at the is_receiving Location."]
  is_qc_pass{"Lot passes QC?"}
  reject_rm("Lot marked REJECTED (held / returned to vendor).")
  tx_qc_pass["InventoryTx: TRANSFER QUARANTINE → INV (2 value-balanced lines); breakdown moves to the default INV Location."]
  rm_in_inventory((("RM in INV — located, lot APPROVED")))

  start --> order_po -->|no inventory write until receipt| receive
  receive --> tx_receive --> lot_receive --> is_qc_pass
  is_qc_pass -->|no| reject_rm
  is_qc_pass -->|yes| tx_qc_pass --> rm_in_inventory
```

```mermaid
---
title: Manipulate Raw Material Inventory
---
flowchart TB
  rm_in_inventory(("RM in INV — located"))
  adjust_negative("User applies a negative adjustment / cycle-count loss.")
  tx_adjust_negative["InventoryTx: ADJUSTMENT, INV, −qty."]
  adjust_positive("User applies a positive adjustment / cycle-count gain.")
  tx_adjust_positive["InventoryTx: ADJUSTMENT, INV, +qty."]
  relocate_rm("User relocates RM.")
  tx_relocate_rm["LocationMove: INV, Location A → Location B (qty & status unchanged)."]
  stage_wip("Stage RM into WIP (refill cans).")
  tx_stage_wip["InventoryTx: TRANSFER INV → WIP (2 lines); removed from its INV Location."]
  rm_in_wip((("RM in WIP — unlocated, lot-less blend")))

  rm_in_inventory -->|optional| adjust_negative --> tx_adjust_negative
  rm_in_inventory -->|optional| adjust_positive --> tx_adjust_positive --> rm_in_inventory
  rm_in_inventory -->|optional| relocate_rm --> tx_relocate_rm --> rm_in_inventory
  rm_in_inventory --> stage_wip --> tx_stage_wip --> rm_in_wip
```

```mermaid
---
title: Manipulate WIP
---
flowchart TB
  A@{ shape: brace-r, label: "To add WIP positive, adjust INV then stage into WIP" }
  rm_in_wip(("RM in WIP — unlocated, lot-less blend"))
  scrap_wip("Scrap RM from WIP.")
  tx_scrap_wip["InventoryTx: ADJUSTMENT, WIP, −qty (no lot)."]
  consume("Work order completes — components consumed.")
  tx_consume["InventoryTx: CONSUME, WIP, −qty (per component; no lot)."]
  output_fg("Target FG output into the vat.")
  tx_output["InventoryTx: PRODUCTION_OUTPUT, WIP, +qty (rolled-up cost)."]
  lot_fg["ReceivedLot created (origin PRODUCTION, QC PENDING)."]
  fg_qc{"FG lot passes QC?"}
  reject_fg("Lot REJECTED (not eligible to pack off).")
  packoff("Pack off QC-approved FG (FIFO over approved lots).")
  tx_packoff["InventoryTx: TRANSFER WIP → INV (2 lines); placed at an INV Location."]
  fg_in_inventory((("FG in INV — located, lot APPROVED")))

  rm_in_wip -->|optional| scrap_wip --> tx_scrap_wip
  rm_in_wip --> consume --> tx_consume --> output_fg --> tx_output --> lot_fg --> fg_qc
  fg_qc -->|no| reject_fg
  fg_qc -->|yes| packoff --> tx_packoff --> fg_in_inventory
```

```mermaid
---
title: Manipulate FG
---
flowchart TB
  fg_in_inventory(("FG in INV — located"))
  scrap_fg("User scraps FG.")
  tx_scrap_fg["InventoryTx: ADJUSTMENT, INV, −qty."]
  adjust_fg_positive("User applies a positive FG adjustment / cycle-count gain.")
  tx_adjust_fg_positive["InventoryTx: ADJUSTMENT, INV, +qty."]
  relocate_fg("User relocates FG.")
  tx_relocate_fg["LocationMove: INV, Location A → Location B."]
  ship_fg("User ships FG to fulfill a Sales Order.")
  tx_ship_fg["InventoryTx: SHIPMENT, INV, −qty; removed from its INV Location."]
  fg_shipped((("FG shipped — out of inventory")))

  fg_in_inventory -->|optional| scrap_fg --> tx_scrap_fg
  fg_in_inventory -->|optional| adjust_fg_positive --> tx_adjust_fg_positive --> fg_in_inventory
  fg_in_inventory -->|optional| relocate_fg --> tx_relocate_fg --> fg_in_inventory
  fg_in_inventory --> ship_fg --> tx_ship_fg --> fg_shipped
```

```mermaid
---
title: Inventory Lifecycle
---
flowchart
  po["Purchase Order"]
  subgraph "Material Handler"
    rm_inv_adj["Raw Material Inventory Adjustment"]
    rm_inv["Raw Material Inventory"]
  end
  scrap["Scrap"]
  subgraph "Compounder Station"
    rm_wip_adj["Raw Material WIP Adjustment"]
    rm_wip["Raw Material WIP"]
    fg_wip_adj["Finished Good WIP Adjustment"]
    fg_wip["Finished Good WIP"]
  end
  subgraph "Pack-off"
    fg_inv_adj["Finished Good Inventory Adjustment"]
    fg_inv["Finished Good Inventory"]
  end
  so["Sales Order"]

  po -->         rm_inv -->     rm_wip -->     fg_wip -->     fg_inv --> so
  rm_inv_adj --> rm_inv --> scrap
                 rm_wip_adj --> rm_wip --> scrap
                                fg_wip_adj --> fg_wip --> scrap
                                               fg_inv_adj --> fg_inv --> scrap

```

The schema below is built to facilitate this life-cycle. Quantity, cost, and
`status` (`INV` / `WIP` / `QUARANTINE`) changes are recorded as append-only rows in
`InventoryTx`; physical relocation **within** a status is recorded in the separate
append-only `LocationMove` ledger. Together they are the immutable transaction
history for auditing material flow.

```mermaid
---
title: "Lifecycle & Audit Schema"
---
erDiagram
  Item {
    int      item_id PK
    varchar  kind "('RM', 'FG')"
    varchar  sku
    varchar  description
    int      rm_id FK "NULL :: Fragrance"
    int      fg_id FK "NULL :: formulainfo"
  }
  Lot {
    int      lot_id PK
    int      item_id FK
    varchar  lot_code
    datetime created_at
  }
  Location {
    int      location_id PK
    int      tenant_id FK
    varchar  name
    varchar  code "NULL"
    boolean  is_default "default INV location"
    boolean  is_receiving "receiving / quarantine dock"
    boolean  active
  }
  ItemStock {
    int      item_stock_id PK
    int      tenant_id FK
    int      item_id FK
    varchar  status "('INV','WIP','QUARANTINE')"
    decimal  quantity
    decimal  avg_cost
  }
  ItemStockLocation {
    int      item_stock_location_id PK
    int      tenant_id FK
    int      item_id FK
    varchar  status "('INV','QUARANTINE') — located only"
    int      location_id FK
    decimal  quantity
  }
  LocationMove {
    int      location_move_id PK
    int      tenant_id FK
    int      item_id FK
    varchar  status "('INV','QUARANTINE')"
    int      from_location_id FK "NULL"
    int      to_location_id FK "NULL"
    decimal  quantity
    int      actor_id FK "NULL"
    varchar  note "NULL"
    datetime occurred_at
  }
  InventoryTx {
    int      tx_id PK
    int      tenant_id FK
    int      item_id FK
    int      lot_id FK "NULL — proposed; lost in WIP"
    varchar  type "('RECEIPT','CONSUME','PRODUCTION_OUTPUT','SHIPMENT','ADJUSTMENT','TRANSFER')"
    varchar  state "('INV','WIP','QUARANTINE')"
    decimal  quantity "signed: + inbound, - outbound"
    decimal  unit_cost
    decimal  value "signed = quantity * unit_cost"
    decimal  balance_qty "on-hand after this line"
    decimal  balance_avg_cost "moving avg after this line"
    varchar  doc_type "('PURCHASE_ORDER','PRODUCTION_RUN','SALES_ORDER','ADJUSTMENT') NULL"
    int      doc_id "NULL"
    int      created_by FK "NULL — proposed (P2 #3); who posted the tx"
    varchar  note "NULL"
    datetime occurred_at
  }
  PurchaseOrder {
    int      po_id PK
    int      vendor_id FK
    varchar  status "('Draft','Ordered','Received','Closed')"
    datetime created_at
  }
  PurchaseOrderLine {
    int      po_line_id PK
    int      po_id FK
    int      item_id FK
    decimal  qty_ordered
    decimal  qty_received
  }
  SalesOrder {
    int      so_id PK
    int      customer_id FK
    varchar  status "('Open','Picked','Shipped','Closed')"
    datetime created_at
  }
  SalesOrderLine {
    int      so_line_id PK
    int      so_id FK
    int      item_id FK
    decimal  qty_ordered
    decimal  qty_shipped
  }
  User {
    int      user_id PK
    varchar  name
  }

  Item              ||--o{ Lot               : "item_id"
  Item              ||--o{ InventoryTx       : "item_id"
  Lot               |o--o{ InventoryTx       : "lot_id (NULL in WIP)"
  User              ||--o{ InventoryTx       : "created_by"
  Item              ||--o{ ItemStock         : "item_id"
  Item              ||--o{ ItemStockLocation : "item_id"
  Location          ||--o{ ItemStockLocation : "location_id"
  Item              ||--o{ LocationMove      : "item_id"
  Location          |o--o{ LocationMove      : "from_location_id"
  Location          |o--o{ LocationMove      : "to_location_id"
  User              |o--o{ LocationMove      : "actor_id"
  PurchaseOrder     ||--|{ PurchaseOrderLine : "po_id"
  Item              ||--o{ PurchaseOrderLine : "item_id"
  SalesOrder        ||--|{ SalesOrderLine    : "so_id"
  Item              ||--o{ SalesOrderLine    : "item_id"
  PurchaseOrderLine }o..o{ InventoryTx       : "doc (PURCHASE_ORDER)"
  SalesOrderLine    }o..o{ InventoryTx       : "doc (SALES_ORDER)"
```

> [!IMPORTANT]
> `InventoryTx` mirrors fw3's `InventoryTxn` and is **item-keyed with no location
> columns**: `tenant_id`, `item_id`, `type`, `state` (`INV` / `WIP` / `QUARANTINE`),
> a **signed** `quantity`, `unit_cost` / `value`, the carried `balance_qty` /
> `balance_avg_cost`, `doc_type` / `doc_id`, `note`, and `occurred_at`. `lot_id`
> (P1 #1) and `created_by` (P2 #3) remain proposed extensions not yet in fw3.

> [!IMPORTANT]
> **fw3 splits inventory across two append-only ledgers — neither is a
> location-stamped `InventoryTx`** (this supersedes the earlier "`from`/`to` on the
> ledger" proposal):
> - **`InventoryTx`** records quantity, cost, and `status` changes (`INV` / `WIP` /
>   `QUARANTINE`); status transitions (`QUARANTINE` → `INV` at QC, `INV` → `WIP` at
>   staging, `WIP` → `INV` at pack-off) are `TRANSFER` lines. `ItemStock` is the
>   per-(item, status) position.
> - **`LocationMove`** records physical relocation **within** a status (qty and
>   status unchanged), carrying its own `actor_id`. `ItemStockLocation` is the
>   per-(item, status, location) quantity breakdown, kept only for located statuses
>   (`INV`, `QUARANTINE`); **`WIP` is not located**.
>
> `Location` is **physical only** (warehouse / room / bin / dock, with `is_default`
> and `is_receiving` flags) — there are **no** boundary/virtual locations.

The flowcharts below predate this split. Each conceptual "move" lands in the
quantity/status ledger (`InventoryTx`), the physical-location ledger
(`LocationMove`), or both; the system-boundary "locations" are not entities:

| Flowchart move | `InventoryTx` (qty / status) | Location effect | Driver |
| --- | --- | --- | --- |
| Receive RM | `RECEIPT`, `QUARANTINE`, + | placed at `is_receiving` Location | PO line |
| Pass QC | `TRANSFER` `QUARANTINE` → `INV` | breakdown → default `INV` Location | QC review |
| Relocate | — | `LocationMove` (`INV`, from → to) | move op |
| Stage into WIP | `TRANSFER` `INV` → `WIP` | removed from Location (`WIP` unlocated) | work order |
| Consume (complete) | `CONSUME`, `WIP`, − | — | work order |
| Output FG (complete) | `PRODUCTION_OUTPUT`, `WIP`, + | — (`WIP` unlocated) | work order |
| Pack-off FG | `TRANSFER` `WIP` → `INV` | placed at `INV` Location | work order |
| Adjust (+/−) | `ADJUSTMENT`, +/− | breakdown +/− if located | adjustment / count |
| Scrap | `ADJUSTMENT`, − | removed if located | adjustment |
| Ship FG | `SHIPMENT`, `INV`, − | removed from Location | SO line |

> [!NOTE]
> `InventoryTx` is **append-only**. Corrections are made by posting a reversing
> transaction, never by editing or deleting a row — this preserves a complete
> audit trail. The `doc_type` / `doc_id` pair links a line back to whatever drove
> the movement (a PO, production run, sales order, or adjustment), and the proposed
> `created_by` records who posted it.

> [!TIP]
> `Location` maps to the legacy `StorageLocation` (physical building locations). The
> system-boundary "Special Locations" below are **not** locations in fw3 — they are
> implicit (see that section). An item's on-hand and moving-average cost are carried
> per `InventoryTx` line as `balance_qty` / `balance_avg_cost`, and the
> per-(item, status) position is `ItemStock`; `ItemStockLocation` breaks the located
> statuses (`INV`, `QUARANTINE`) down by `Location`, summing to the `ItemStock` row.

> [!NOTE]
> `item_id` is set on **every** `InventoryTx` row. The proposed `lot_id` (fw3's
> `ReceivedLot`) is set only while material is lot-trackable: the `RECEIPT`, the QC
> `TRANSFER`, and `INV` adjustments carry it, as do the FG pack-off `TRANSFER` and
> `INV` lines. The `CONSUME` and `ADJUSTMENT` lines in `WIP` carry a **null** lot —
> RM blends in the refill cans, so its lot can no longer be recovered. The
> stage-into-`WIP` `TRANSFER` is the last lot-attributed line for an RM lot; lot
> identity resumes with the PRODUCTION `ReceivedLot` at output.

### Special Locations (not modeled as locations in fw3)

fw3 has **no** boundary/virtual locations; only physical `Location`s exist. The
"special locations" the flowcharts use map onto fw3 as:

- **Vendor / Customer** — system boundaries, not stored. Crossing one is an
  `InventoryTx` `RECEIPT` (inbound) or `SHIPMENT` (outbound).
- **Receiving** — a physical `Location` (`is_receiving` = true); received stock
  sits there under `QUARANTINE` status until QC passes (`TRANSFER` to `INV`).
- **Scrap** — not a location; a negative `ADJUSTMENT`.
- **WIP** — an `InventoryTx` / `ItemStock` `status`, **not** located (refill-can blend).
- **FG** — not a location; finished goods are an `Item` (`kind` = FG) held under
  `INV` status at a physical `Location`.
- **Shipping** — not modeled; the `SHIPMENT` line removes stock from its location.

## Goals

- [x] Single Source of Truth

> Accomplished with InventoryTx table for both RMs & FGs w/ views.

- [x] Transactions for **all** RM & FG movements

> `InventoryTx` and `Lot` tables accommodate this.

- [x] Net-0 tables

> Using views (old::new): 16:12

- [x] Minimal Tool Breakage

> Views with matching interface to replaced tables will reduce breakage. Will still require populating `InventoryTx` table to match current state.

## Ledger Design: Two-Ledger vs Single-Ledger

fw3 keeps **two append-only ledgers**, and the position tables are derived from
them (mutable, upserted — not ledgers):

| Table | Role | Records |
| --- | --- | --- |
| `InventoryTx` (fw3 `InventoryTxn`) | quantity / cost / status ledger | `RECEIPT`, `CONSUME`, `PRODUCTION_OUTPUT`, `SHIPMENT`, `ADJUSTMENT`, `TRANSFER`; carries running `balance_qty` / `balance_avg_cost` per (item, status) |
| `LocationMove` | physical-move ledger | relocation **within** a status (`INV` / `QUARANTINE`); value-neutral; carries `actor_id` |
| `ItemStock` | position (derived) | per (item, status) quantity + avg cost |
| `ItemStockLocation` | position (derived) | per (item, status, location) quantity — located statuses only |

A **single-ledger** design would fold physical moves into `InventoryTx` (a
nullable `location_id`, or `from`/`to`, with a relocation recorded as one
`from→to` row or two signed rows) and derive both position tables from that one
ledger. It is feasible, but the trade-offs favor keeping the split.

**Keep two ledgers (recommended) because:**

- The quantity/cost ledger is the accounting-critical record (weighted-average
  cost, `value`). Relocations are frequent and **value-neutral**; merging them in
  pollutes every cost/quantity query with physical-move noise.
- `InventoryTx`'s invariant "latest line per (item, status) is the position" stays
  clean only if value-neutral relocations are kept out of it.
- WIP is not located; INV/QUARANTINE are. The split lets `ItemStockLocation` /
  `LocationMove` keep a DB-enforceable "located rows always have a location"
  guarantee. A single ledger forces a nullable `location_id` (null for WIP) and
  loses that.
- The two ledgers can be written and locked independently; relocations don't
  contend on (or bloat the indexes of) the valuation-source table.

**A single ledger would buy:**

- One source of truth — no chance of the two ledgers drifting (though fw3 already
  keeps them consistent: every move runs in a DB transaction, and
  `ItemStockLocation` sums to the `ItemStock` status total).
- One unified actor column. But `LocationMove` already has `actor_id` and
  `InventoryTx` does not — that gap (P2 #3) is far cheaper to close by adding
  `created_by` to `InventoryTx` than by merging the ledgers.

**Verdict:** keep the two-ledger split. If a single source of truth ever became
necessary, the cleanest variant is a single **fully-located, signed** ledger
(relocation = two rows) that derives both position tables — worth taking on only
if the ledgers were actually drifting in practice, which they are not.

## Notes

> [!CAUTION]
> Changing the process to use `InventoryTx` table will break **all** inventory-related writes & updates. This will require an overhaul of the system, projected to affect more than 80% of the code.

> [!IMPORTANT]
> The `InventoryTx` ledger is keyed by `item_id` (always set) with an **optional** `lot_id`, mirroring fw3's implemented `InventoryTxn` (item-keyed, with an `INV` / `WIP` state). LOT traceability is **physically lost** when RM moves into WIP: the material is poured into refill cans and blends with other lots of the same item, so no single lot can be recovered. From that point a line records the `Item` but a `null` `Lot`; lot traceability resumes when a new FG `Lot` is created at pack-off.

> [!NOTE]
> Because WIP is a lot-less blend, only the RM LOTs whose (lot-attributed) move-into-WIP lines fed that blend can be said to _possibly_ have affected a pour — the exact lot is unrecoverable. Tracing those candidate LOTs is separate from the internal LOTs consumed when a pour is performed. Internal LOTs should **always** consume FIFO regardless of traced internal LOT(s).

> [!TIP]
> Pours should generate `Consumptions` against the RM (possibly using the Command/Event pattern), and those consumptions should be applied against the WIP inventory in FIFO order _after_ the pour is completed.

> [!IMPORTANT]
> RSM wants to use `InventoryTx` table for **both** Raw Materials and Finished Goods.

> [!TIP]
> CoPilot recommends a `Lot` table in order to track both Raw Materials and Finished Goods within the `InventoryTx` table. Each `Lot` carries a surrogate `lot_id` and references an `Item`, whose `kind` enum (`'RM'`, `'FG'`) distinguishes Raw Materials from Finished Goods. A `Lot` exists only while material is lot-trackable (RM in inventory, FG after pack-off); the `INV` / `WIP` distinction is **not** a property of the `Lot` but of each `InventoryTx` line (its `WIP` location). _When accessing a `Lot`, its `Item.kind` should **always** be checked._

> [!IMPORTANT]
> RSM wants to replace `formula_stock_lot_adjustment`, `WHPrepStockDetail`, `whprep_StorageLocation_Lot`, `whorderdetail`, `whprepdetail_qtydetail`, and `multi_StorageLocation` with views against `InventoryTx` table as interfaces to prevent tool breakage.

> [!NOTE]
> Following FIFO physically on the floor is impractical, does not have much benefit, and physical processes keep the state close enough to correct.

> [!TIP]
> Adding the `Item` table goes against keeping the number of new tables to a minimum, but we are still well below the net-0 table requirement by using the views. CoPilot insists that the `Item` table is necessary for referential integrity, keeping query complexity minimal (reducing number of joins), performance and indexing, and future flexibility.


## Questions

Why is the current system tying pours directly to LOT numbers for consumption?
  :

Why does the system require a specific LOT assignment at time of `Start Prep`?
  :

When a Raw Material is consumed for a Finished Good, but the Finished Good has not been packaged, where is the Raw Material at that point? It's in the Finished Good, but the Finished Good doesn't truly exist yet. When the Finished Good is made, where should it come from? WIP?
  : The RM is consumed out of `WIP` when the work order completes — an `InventoryTx` `CONSUME`, `WIP`, −qty (lot-less; the refill-can blend). The Finished Good is _produced_, not moved: a separate `PRODUCTION_OUTPUT`, `WIP`, +qty posts the FG into the vat (a new PRODUCTION `ReceivedLot`, QC PENDING) at the rolled-up consumed cost. They are independent signed postings on different items — consumed value equals output value, so it balances without the FG being "sourced from" the RM. Pack-off is later a `TRANSFER` `WIP` → `INV`.

When more Finished Good is found during a cycle count, and the inventory is positively adjusted, what should be the `from`?
  : There is no `from`/`to` — fw3's ledger is signed. A positive count is an `InventoryTx` `ADJUSTMENT`, `INV`, +qty (no source to debit), and the located breakdown places the extra at its `Location`; a negative count is the same shape with −qty (scrap/loss included). This applies to both RM and FG. See the `Manipulate FG` flowchart's `tx_adjust_fg_positive`.

Should additional "Special" locations be added for system boundaries ("Vendor", "Customer", etc.)? Current flowcharts show both yes and no.
  : **No** — fw3 settled this. `Location` is physical only; system boundaries (Vendor, Customer, Scrap, Shipping) are implicit in the `InventoryTx` `type` (`RECEIPT` / `SHIPMENT` / `ADJUSTMENT`), not stored as locations. The one real physical location for a "boundary" is **Receiving** (`is_receiving` = true), where received stock is held under `QUARANTINE` until QC passes. See "Special Locations" above.
