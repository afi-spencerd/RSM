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
    int      lot_id FK
    int      from_location_id FK "NULL"
    int      to_location_id   FK "NULL"
    decimal  qty
    datetime created_at
    varchar  kind "('RECEIPT', 'MOVE', 'CONSUMPTION', 'ADJUSTMENT', 'PRODUCTION', 'SCRAP')"
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
    varchar  stage "('Inventory', 'WIP')"
    datetime created_at
  }

  formulainfo
  Fragrance
  StorageLocation
  whorderdetail {
    int WHOrderDetailID PK
    float QtyReceived
  }

  InventoryTx }o..|| StorageLocation : "InventoryTx.to_location_id :: StorageLocation.StorageLocationID"
  InventoryTx }o..|| StorageLocation : "InventoryTx.from_location_id :: StorageLocation.StorageLocationID"
  whorderdetail ||--|{ InventoryTx : "whorderdetail.WHOrderDetailID :: InventoryTx.lot_id"
  InventoryTx }|--|| Lot : "InventoryTx.lot_id :: Lot.lot_id"
  Lot }o--|| Item : "Lot.item_id :: Item.item_id"
  Item ||..|| Fragrance : "Item.rm_id :: Fragrance.id"
  Item ||..|| formulainfo : "Item.fg_id :: formulainfo.id"
```

```mermaid
---
title: Receiving to Inventory
---
flowchart TB
  start((Start))
  ensure_item_rm["Get-or-create the RM's \`Item\` (\`kind\` = 'RM', links to Fragrance)."]
  create_rm_lot["Create the RM \`Lot\` (\`stage\` = 'Inventory') against the \`Item\`."]
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
  tx_wip_rm["Write from \[building_location_id\] to WIP in tx table."]
  cast_rm_to_wip["Change Lot \`stage\` to 'WIP'."]
  lot_in_wip((("LOT in WIP")))

  rm_in_inventory -->|optional| adjust_inventory_negative --> tx_adjust_inventory_negative
  rm_in_inventory -->|optional| adjust_inventory_positive --> tx_adjust_inventory_positive --> rm_in_inventory
  rm_in_inventory -->|optional| relocate_rm --> tx_relocate_rm --> rm_in_inventory
  rm_in_inventory --> wip_rm --> tx_wip_rm --> cast_rm_to_wip --> lot_in_wip
```

```mermaid
---
title: Manipulate WIP
---
flowchart TB
  A@{ shape: brace-r, label: "To adjust WIP positive, adjust inventory then move to WIP"}
  lot_in_wip(("LOT in WIP"))
  consume_rm("RM consumed in Compounder Tool.")
  tx_consume_rm["Write a CONSUMPTION (from WIP to null) against the RM \`Lot\` in \`InventoryTx\`."]
  scrap_wip_rm("Scrap RM from WIP.")
  tx_scrap_wip_rm(["Write from WIP to Scrap in tx table."])
  pack_fg("Packer packs FG.")
  ensure_item_fg["Get-or-create the FG's \`Item\` (\`kind\` = 'FG', links to formulainfo)."]
  create_fg["Create FG \`Lot\` (\`stage\` = 'Inventory') against the \`Item\`."]
  tx_create_fg["Write a PRODUCTION (from null to FG) against the new FG \`Lot\` in \`InventoryTx\`."]
  fg_in_inventory((("FG in Inventory.")))

  lot_in_wip -->|optional| scrap_wip_rm --> tx_scrap_wip_rm
  lot_in_wip --> consume_rm --> tx_consume_rm --> pack_fg --> ensure_item_fg --> create_fg --> tx_create_fg --> fg_in_inventory
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

The schema below is built to facilitate this life-cycle. Every stage transition
in the flowchart above corresponds to one append-only row in `InventoryTx`, which
serves as the immutable transaction history (ledger) for auditing material flow.

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
    varchar  stage "('Inventory', 'WIP')"
    datetime created_at
  }
  Location {
    int      location_id PK
    varchar  name
    varchar  kind "('Building','Vendor','Receiving','WIP','FG','Shipping','Scrap')"
    boolean  is_virtual "true for system-boundary locations"
  }
  InventoryTx {
    int      tx_id PK
    int      lot_id FK
    int      from_location_id FK "NULL"
    int      to_location_id FK "NULL"
    decimal  qty
    varchar  kind "('RECEIPT','MOVE','CONSUMPTION','ADJUSTMENT','PRODUCTION','SCRAP')"
    varchar  source_doc_type "('PO','SO','ADJUSTMENT','PREP') NULL"
    int      source_doc_id "NULL"
    varchar  reason
    int      created_by FK "User who posted the tx"
    datetime created_at
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
  Lot               ||--o{ InventoryTx       : "lot_id"
  Location          ||--o{ InventoryTx       : "to_location_id"
  Location          ||--o{ InventoryTx       : "from_location_id"
  User              ||--o{ InventoryTx       : "created_by"
  PurchaseOrder     ||--|{ PurchaseOrderLine : "po_id"
  Item              ||--o{ PurchaseOrderLine : "item_id"
  SalesOrder        ||--|{ SalesOrderLine    : "so_id"
  Item              ||--o{ SalesOrderLine    : "item_id"
  PurchaseOrderLine }o..o{ InventoryTx       : "source_doc (PO)"
  SalesOrderLine    }o..o{ InventoryTx       : "source_doc (SO)"
```

> [!NOTE]
> `InventoryTx` is **append-only**. Corrections are made by posting a reversing
> transaction, never by editing or deleting a row — this preserves a complete
> audit trail. The `source_doc_type` / `source_doc_id` pair is a polymorphic link
> back to whatever drove the movement (a PO line, SO line, adjustment, or prep),
> and `created_by` records who posted it.

> [!TIP]
> `Location` generalizes the legacy `StorageLocation` (building locations,
> `is_virtual` = false) together with the system-boundary "Special Locations"
> below (`is_virtual` = true). Current on-hand by lot/location is a `SUM(qty)`
> view over `InventoryTx`, never a stored balance.

Each life-cycle transition maps to exactly one `InventoryTx` row:

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

### Special Locations

- Vendor
- Receiving
- WIP
- FG
- Shipping
- Scrap

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

> [!NOTE]
> Tracing which WIP LOTs _could_ have affected a pour is separate from the internal LOTs consumed when a pour is performed. Internal LOTs should **always** consume FIFO regardless of traced internal LOT(s).

> [!TIP]
> Pours should generate `Consumptions` against the RM (possibly using the Command/Event pattern), and those consumptions should be applied against the WIP inventory in FIFO order _after_ the pour is completed.

> [!IMPORTANT]
> RSM wants to use `InventoryTx` table for **both** Raw Materials and Finished Goods.

> [!TIP]
> CoPilot recommends a `Lot` table in order to track both Raw Materials and Finished Goods within the `InventoryTx` table. Each `Lot` carries a surrogate `lot_id` and references an `Item`, whose `kind` enum (`'RM'`, `'FG'`) distinguishes Raw Materials from Finished Goods; the `Lot.stage` enum (`'Inventory'`, `'WIP'`) tracks where the lot sits in its life-cycle. _When accessing a `Lot`, its `Item.kind` should **always** be checked._

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
  :
