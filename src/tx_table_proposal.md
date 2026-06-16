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

  start --> order_rm --> tx_order_rm -->|optional| unorder_rm --> tx_unorder_rm --> order_rm
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
  cast_rm_to_wip["Change Lot \`kind\` to WIP"]
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
  tx_consume_rm["Write from WIP to FG in tx table."]
  scrap_wip_rm("Scrap RM from WIP.")
  tx_scrap_wip_rm(["Write from WIP to Scrap in tx table."])
  pack_fg("Packer packs FG.")
  create_fg["Create FG Lot in \`Lot\` table."]
  tx_create_fg["Write from WIP to FG in \`InventoryTx\` table against new FG LOT."]
  fg_in_inventory((("FG in Inventory.")))

  lot_in_wip -->|optional| scrap_wip_rm --> tx_scrap_wip_rm
  lot_in_wip --> consume_rm --> tx_consume_rm --> pack_fg --> create_fg --> tx_create_fg --> fg_in_inventory
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

> [!NOTE]
> Tracing which WIP LOTs _could_ have affected a pour is separate from the internal LOTs consumed when a pour is performed. Internal LOTs should **always** consume FIFO regardless of traced internal LOT(s).

> [!TIP]
> Pours should generate `Consumptions` against the RM (possibly using the Command/Event pattern), and those consumptions should be applied against the WIP inventory in FIFO order _after_ the pour is completed.

> [!IMPORTANT]
> RSM wants to use `InventoryTx` table for **both** Raw Materials and Finished Goods.

> [!TIP]
> CoPilot recommends a `Lot` table in order to track both Raw Materials and Finished Goods within the `InventoryTx` table. To enable tracking both RM LOTs (`int`) and FG LOTs (`varchar`), it recommends having a `kind` enum column and cast all LOT codes to `varchar` in the `Lot` table. _When accessing the table, row `kind` should **always** be checked._

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
  :

When more Finished Good is found during a cycle count, and the inventory is positively adjusted, what should be the `from`?
  :

Should additional "Special" locations be added for system boundaries ("Vendor", "Customer", etc.)? Current flowcharts show both yes and no.
  :
