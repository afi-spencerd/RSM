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
> The flowcharts below describe the **conceptual physical lifecycle** and predate
> fw3's two-ledger location model. Their "Write from _X_ to _Y_ in tx table" steps
> and boundary "locations" (Vendor, Scrap, Shipping, …) map onto fw3's `InventoryTx`
> (quantity / cost / `INV`-`WIP`-`QUARANTINE` status) and `LocationMove` (physical
> placement) per the mapping under **Lifecycle & Audit Schema** below. They are kept
> as-is pending a re-model.

```mermaid
---
title: Receiving to Inventory
---
flowchart TB
  start((Start))
  ensure_item_rm["Get-or-create the RM's \`Item\` (\`kind\` = 'RM', links to Fragrance)."]
  create_rm_lot["Create the RM \`Lot\` against the \`Item\`."]
  order_rm("User sets an PO to ordered.")
  tx_order_rm["Write from null to Vendor to tx table."]
  unorder_rm("User unsets 'ordered' in PO.")
  tx_unorder_rm["Write from Vendor to null to tx table."]
  rm_arrive("RM arrives at Receiving dock.")
  tx_rm_arrive["Write from Vendor to 'Receiving' to tx table."]
  is_qc_pass{RM passes QC?}
  return_rm("RM is returned to Vendor.")
  tx_return_rm(["Write from 'Receiving' to 'Vendor' to tx table."])
  tx_receive_rm["Write from 'Receiving' to \[building_location_id\] in tx table."]
  rm_in_inventory((("RM in Inventory")))

  start --> ensure_item_rm --> create_rm_lot --> order_rm --> tx_order_rm -->|optional| unorder_rm --> tx_unorder_rm --> order_rm
                         tx_order_rm --> rm_arrive --> tx_rm_arrive --> is_qc_pass -->|no| return_rm --> tx_return_rm
                                                                        is_qc_pass -->|yes| tx_receive_rm --> rm_in_inventory
```

```mermaid
---
title: Manipulate Raw Material Inventory
---
flowchart TB
  rm_in_inventory(("RM in Inventory"))
  adjust_inventory_negative("User applies negative inventory adjustment.")
  tx_adjust_inventory_negative(["Write from \[building_location_id\] to Scrap in tx table."])
  adjust_inventory_positive("User applies positive inventory adjustment.")
  tx_adjust_inventory_positive["Write from null to \[building_location_id\] in tx table."]
  relocate_rm("User relocates RM.")
  tx_relocate_rm["Write from \[building_location_id\] to \[building_location_id\] in tx table."]
  wip_rm("Move RM into WIP.")
  tx_wip_rm["Write from \[building_location_id\] to WIP against the RM \`Lot\` — last lot-attributed line."]
  rm_in_wip((("RM in WIP (lot-less blend)")))

  rm_in_inventory -->|optional| adjust_inventory_negative --> tx_adjust_inventory_negative
  rm_in_inventory -->|optional| adjust_inventory_positive --> tx_adjust_inventory_positive --> rm_in_inventory
  rm_in_inventory -->|optional| relocate_rm --> tx_relocate_rm --> rm_in_inventory
  rm_in_inventory --> wip_rm --> tx_wip_rm --> rm_in_wip
```

```mermaid
---
title: Manipulate WIP
---
flowchart TB
  A@{ shape: brace-r, label: "To adjust WIP positive, adjust inventory then move to WIP"}
  rm_in_wip(("RM in WIP (lot-less blend)"))
  consume_rm("RM consumed in Compounder Tool.")
  tx_consume_rm["Write a CONSUMPTION (from WIP to null) for the RM \`Item\` — no \`Lot\` (WIP blend)."]
  scrap_wip_rm("Scrap RM from WIP.")
  tx_scrap_wip_rm(["Write a SCRAP (from WIP to Scrap) for the RM \`Item\` — no \`Lot\`."])
  pack_fg("Packer packs FG.")
  ensure_item_fg["Get-or-create the FG's \`Item\` (\`kind\` = 'FG', links to formulainfo)."]
  create_fg["Create FG \`Lot\` against the \`Item\` — LOT traceability resumes at pack-off."]
  tx_create_fg["Write a PRODUCTION (from null to FG) against the new FG \`Lot\` in \`InventoryTx\`."]
  fg_in_inventory((("FG in Inventory.")))

  rm_in_wip -->|optional| scrap_wip_rm --> tx_scrap_wip_rm
  rm_in_wip --> consume_rm --> tx_consume_rm --> pack_fg --> ensure_item_fg --> create_fg --> tx_create_fg --> fg_in_inventory
```

```mermaid
---
title: Manipulate FG
---
flowchart TB
  fg_in_inventory(("FG in Inventory."))
  scrap_fg("User scraps FG.")
  tx_scrap_fg(["Write from: 'FG' to: 'Scrap' in InventoryTx."])
  adjust_fg_positive("User performs a positive adjustment to FG.")
  tx_adjust_fg_positive["Write from: null to: 'FG' in \`InventoryTx\`."]
  pick_fg("User consumes FG to fulfill Sales Order.")
  tx_pick_fg["Write from: 'FG' to: 'Shipping'"]
  ship_fg("User ships FG to Customer.")
  tx_ship_fg(["Write from: 'Shipping' to null in \`InventoryTx\`"])

  fg_in_inventory -->|optional| scrap_fg --> tx_scrap_fg
  fg_in_inventory -->|optional| adjust_fg_positive --> tx_adjust_fg_positive --> fg_in_inventory
  fg_in_inventory --> pick_fg --> tx_pick_fg --> ship_fg --> tx_ship_fg
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

| Flowchart move | `InventoryTx` (qty / status) | Location effect |
| --- | --- | --- |
| Receive RM (PO) | `RECEIPT`, `QUARANTINE`, + | placed at `is_receiving` location |
| Pass QC | `TRANSFER` `QUARANTINE` → `INV` | breakdown moves to default `INV` loc |
| Relocate in inventory | — | `LocationMove` (`INV`, from → to) |
| Stage RM into WIP | `TRANSFER` `INV` → `WIP`, − | removed from location (`WIP` unlocated) |
| Consume RM in pour | `CONSUME`, `WIP`, − | — |
| Pack-off FG | `PRODUCTION_OUTPUT`, `INV`, + | placed at `INV` location |
| Adjust (+/−) | `ADJUSTMENT`, +/− | breakdown +/− if located |
| Scrap | `ADJUSTMENT`, − | removed if located |
| Ship FG (SO) | `SHIPMENT`, `INV`, − | removed from location |

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

The conceptual life-cycle flow and its driving documents (the flowcharts depict the
same; the per-ledger encoding is in the table above):

| Life-cycle transition | `kind` | `from` → `to` | `source_doc` |
| --- | --- | --- | --- |
| Purchase Order → RM Inventory | `RECEIPT` | `Vendor` → `Receiving` → `[building]` | PO line |
| RM Inventory Adjustment (+/−) | `ADJUSTMENT` | `null` → `[building]` / `[building]` → `Scrap` | adjustment |
| RM Inventory → RM WIP | `MOVE` | `[building]` → `WIP` | prep |
| RM WIP → FG WIP (consumed) | `CONSUMPTION` | `WIP` → `null` | prep |
| FG WIP → FG Inventory (pack-off) | `PRODUCTION` | `null` → `FG` | prep |
| FG Inventory Adjustment (+/−) | `ADJUSTMENT` | `null` → `FG` / `FG` → `Scrap` | adjustment |
| FG Inventory → Sales Order | `MOVE` | `FG` → `Shipping` → `null` | SO line |
| Any stage → Scrap | `SCRAP` | `[stage]` → `Scrap` | adjustment |

> [!NOTE]
> `item_id` is set on **every** row; `lot_id` is set only while material is
> lot-trackable. The RM-inventory lines (`RECEIPT`, `MOVE`, `ADJUSTMENT`) and the
> FG lines (`PRODUCTION`, FG `MOVE`/`ADJUSTMENT`/`SCRAP`) carry a `lot_id`. The
> `CONSUMPTION` and `SCRAP` lines out of `WIP` carry a **null** `lot_id` — RM blends
> in the refill cans, so its lot can no longer be recovered. The move-into-`WIP`
> line is the last lot-attributed line for an RM lot.

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
  : The RM is consumed out of `WIP` at pour time (a `CONSUMPTION` tx, `WIP` → `null`) and is **not** tracked as a discrete lot between pour and pack-off (consistent with not following FIFO physically on the floor). The Finished Good is _produced_, not moved: at pack-off a new FG `Lot` is created and a `PRODUCTION` tx (`null` → `FG`) brings it into inventory. Consumption and production are independent postings against different lots, so the FG is **not** sourced "from WIP" — doing so would double-debit `WIP`.

When more Finished Good is found during a cycle count, and the inventory is positively adjusted, what should be the `from`?
  : The `from` is `null` — a positive adjustment is an `ADJUSTMENT` tx (`null` → `FG`) against the FG `Lot`, mirroring the `Manipulate FG` flowchart's `tx_adjust_fg_positive`. The material has no prior tracked location (it was unaccounted-for stock surfaced by the count), so there is no source to debit. The same shape applies to a positive RM adjustment (`null` → `[building_location_id]`); a negative adjustment instead writes to `Scrap`.

Should additional "Special" locations be added for system boundaries ("Vendor", "Customer", etc.)? Current flowcharts show both yes and no.
  : **No** — fw3 settled this. `Location` is physical only; system boundaries (Vendor, Customer, Scrap, Shipping) are implicit in the `InventoryTx` `type` (`RECEIPT` / `SHIPMENT` / `ADJUSTMENT`), not stored as locations. The one real physical location for a "boundary" is **Receiving** (`is_receiving` = true), where received stock is held under `QUARANTINE` until QC passes. See "Special Locations" above.
