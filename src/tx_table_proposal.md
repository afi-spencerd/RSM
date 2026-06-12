# Tx Table Proposal

## Diagrams

```mermaid
---
title: Current Layout
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
title: Proposal
---
erDiagram
  InventoryTx {
    int      id   PK
    int      lot  FK
    int      from FK
    int      to   FK
    decimal  qty
    datetime created_at
  }
  whorderdetail {
    int WHOrderDetailID PK
    float QtyReceived
  }
  StorageLocation
  InventoryTx }o--|| StorageLocation : "InventoryTx.to :: StorageLocation.StorageLocationID"
  InventoryTx }o--|| StorageLocation : "InventoryTx.from :: StorageLocation.StorageLocationID"
  whorderdetail ||--|{ InventoryTx : "whorderdetail.WHOrderDetailID :: InventoryTx.lot"
```

```mermaid
---
title: Transaction Flow
---
flowchart TB
  start((Start))
  order_rm("User sets an PO to ordered.")
  tx_order_rm["Write from null to Vendor to tx table."]
  unorder_rm("User unsets 'ordered' in PO.")
  tx_unorder_rm["Write from Vendor to null to tx table."]
  rm_arrive("RM arrives at Receiving dock.")
  tx_rm_arrive["Write from Vendor to 'receiving' to tx table."]
  is_qc_pass{RM passes QC?}
  return_rm("RM is returned to Vendor.")
  tx_return_rm(["Write from Receiving to Vendor to tx table."])
  tx_receive_rm["Write from Receiving to \[building_location_id\] in tx table."]
  adjust_inventory_negative("User applies negative inventory adjustment.")
  tx_adjust_inventory_negative["Write from \[building_location_id\] to Scrap in tx table."]
  adjust_inventory_positive("User applies positive inventory adjustment.")
  tx_adjust_inventory_positive["Write from null to \[building_location_id\] in tx table."]
  relocate_rm("User relocates RM.")
  tx_relocate_rm["Write from \[building_location_id\] to \[building_location_id\] to tx table."]
  rm_in_inventory(("RM in Inventory"))
  wip_rm("Move RM into WIP.")
  tx_wip_rm["Write from \[building_location_id\] to WIP in tx table."]
  consume_rm("RM consumed in Compounder Tool.")
  tx_consume_rm(["Write from WIP to FG in tx table."])
  scrap_wip_rm("Scrap RM from WIP.")
  tx_scrap_wip_rm(["Write from WIP to Scrap in tx table."])

  start --> order_rm
  order_rm --> tx_order_rm
  tx_order_rm -->|optional| unorder_rm
  unorder_rm --> tx_unorder_rm
  tx_unorder_rm --> order_rm
  tx_order_rm --> rm_arrive
  rm_arrive --> tx_rm_arrive
  tx_rm_arrive --> is_qc_pass
  is_qc_pass -->|no| return_rm
  return_rm --> tx_return_rm
  is_qc_pass -->|yes| tx_receive_rm
  tx_receive_rm --> rm_in_inventory
  rm_in_inventory -->|optional| adjust_inventory_negative
  adjust_inventory_negative --> tx_adjust_inventory_negative
  tx_adjust_inventory_negative --> rm_in_inventory
  rm_in_inventory -->|optional| adjust_inventory_positive
  adjust_inventory_positive --> tx_adjust_inventory_positive
  tx_adjust_inventory_positive --> rm_in_inventory
  rm_in_inventory -->|optional| relocate_rm
  relocate_rm --> tx_relocate_rm
  tx_relocate_rm --> rm_in_inventory
  rm_in_inventory --> wip_rm
  wip_rm --> tx_wip_rm
  tx_wip_rm -->|optional| scrap_wip_rm
  scrap_wip_rm --> tx_scrap_wip_rm
  tx_wip_rm --> consume_rm
  consume_rm --> tx_consume_rm

```

### Special Locations

- Vendor
- Receiving
- WIP
- FG
- Shipping
- Scrap

## Requirements

- Single source of truth
- Transactions for **all** RM and FG movements
- net-0 tables
- minimal tool breakage

## Notes

> [!NOTE]
> Tracing which WIP LOTs _could_ have affected a pour is separate from the internal LOTs consumed when a pour is performed. Internal LOTs should **always** consume FIFO regardless of traced internal LOT(s).

> [!TIP]
> Pours should generate `Consumptions` against the RM (possibly using the Command/Event pattern), and those consumptions should be applied against the WIP inventory in FIFO order _after_ the pour is completed.

> [!CAUTION]
> RSM wants to use `InventoryTx` table for **both** Raw Materials and Finished Goods.

> [!IMPORTANT]
> RSM wants to replace `formula_stock_lot_adjustment`, `WHPrepStockDetail`, `whprep_StorageLocation_Lot`, `whorderdetail`, `whprepdetail_qtydetail`, and `multi_StorageLocation` with views against `InventoryTx` table as interfaces to prevent tool breakage.

## Questions

- Why is the current system tying pours directly to LOT numbers for consumption?
  - Floor is incapable of truly following FIFO.
- Why does the system require a specific LOT assignment at time of `Start Prep`?
- If using `to` and `from` columns on `Tx` table, what type?
