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

```mermaid
---
title: Proposal
---
erDiagram
```

## Requirements

- Single source of truth
- Transactions for **all** RM and FG movements
- net-0 tables
- minimal tool breakage

## Notes

> Tracing which WIP LOTs _could_ have affected a pour is separate from the internal LOTs consumed when a pour is performed. Internal LOTs should **always** consume FIFO regardless of traced internal LOT(s).

> Pours should generate `Consumptions` against the RM (possibly using the Command/Event pattern), and those consumptions should be applied against the WIP inventory in FIFO order _after_ the pour is completed.

> Use views as interfaces for existing tools.

## Questions

- Why is the current system tying pours directly to LOT numbers for consumption?
  - Floor is incapable of truly following FIFO.
- Why does the system require a specific LOT assignment at time of `Start Prep`?
- If using `to` and `from` columns on `Tx` table, what type?
