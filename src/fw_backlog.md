# FW3 Backlog

Prioritized backlog for the fw3 ERP, reconciled with the repo's `TODO.md` and recent
development. Items are grouped into priority tiers; original to-do entries are kept
close to verbatim, with new **_(suggested)_** items folded in. Each tier opens with
the reasoning behind its rank.

> [!NOTE]
> Reconciliation: life-cycle stages 1–4 (weighted-average ledger, purchasing, sales,
> production + pack-off, quality/quarantine), physical + hierarchical locations, and
> pounds-canonical units (kg→lb on receipt) are **done**. This list is the work that
> remains. See **Recently completed** at the bottom for the items cleared off `TODO.md`.

## P0 — Correctness & data integrity

> Ledger trust is the foundation of the whole system. These either fix something
> user-facing that is broken now, or stop the ledger from being corrupted. Do first.

- [ ] **fix: broken inventory delete button**
  - [ ] Replace delete with "adjust out" — every removal is a recorded `ADJUSTMENT` line, never a row delete.
  - [ ] Block deleting an item that has any `InventoryTxn` history.
- [ ] **fix: broken references to the "Stock" page** (dangling links left by the Stock→Inventory merge).
- [ ] **Enforce append-only ledgers at the DB level** _(suggested)_ — deny `UPDATE`/`DELETE` on `InventoryTxn` and `LocationMove`; corrections become reversing entries. This is the real guarantee behind the delete-button fix.

## P1 — Core workflow & integrations

> Closes the operational loop and the accounting integration. Needed before fw3 can
> fully replace the legacy system.

- [ ] **Complete QBWC sync** — the QuickBooks Desktop round-trip (items, receipts/bills, invoices). Accounting depends on it.
- [ ] **Compounder Tool API**
  - [ ] Expose formulation details to the compounder tool.
  - [ ] Expose the work-order list to the compounder tool.
  - [ ] Accept consumption + batch-progress reports back from the tool.
- [ ] **Facilitate scrap in all inventory stages** — make scrap a first-class disposition (distinct reason/type) instead of a bare negative `ADJUSTMENT`, so loss/yield is reportable across `INV`, `WIP`, and `QUARANTINE`.
- [ ] **Cycle count tool** — count entry → reconcile to signed `ADJUSTMENT` lines per (item, location).
- [ ] **Record the actor on `InventoryTxn`** _(suggested)_ — add `createdById`; the quantity/cost ledger has no actor today (only `LocationMove` does). Required for a complete audit trail.
- [ ] **Attribute ledger lines to lots (`lotId` on `InventoryTxn`)** _(suggested)_ — link quantity lines to `ReceivedLot` for traceability through QC, consumption, and shipment (lost in the WIP blend by design).
- [ ] **Legacy data migration** _(suggested)_ — import items, on-hand balances, open POs/SOs, and lot history from the Formula Web / MSSQL system. A hard go-live blocker.

## P2 — Business value & hardening

> High value, but the system runs without them.

- [ ] **Show cost when generating a Sales Order**
  - [ ] Prevent "losing our shirt" (warn/block below-cost lines).
  - [ ] Factor production cost into the SO (use 80% RMC).
  - [ ] Set a default profit margin.
  - [ ] Customer purchase & cost history.
- [ ] **Separate Inventory from Item Details** — split the item master (sku/name/spec) from the stock position (`quantityOnHand` / `unitCost` are currently mirrored onto `InventoryItem`).
- [ ] **QC-failed RMs back to vendor** — a return/RMA flow off the `REJECTED` quarantine state.
- [ ] **Location rules** — e.g. no move back to the receiving dock after warehousing; valid-transition guards on `LocationMove`.
- [ ] **Allow packing before QC** — relax QC-gated pack-off where policy permits, with a hold/release rule so unapproved FG still cannot ship.
- [ ] **Role-specific views** — surface only the relevant screens/actions per user role (RBAC-driven UI).
- [ ] **Oversell / negative-stock guards** _(suggested)_ — confirm consume/ship/transfer can never drive an (item, status) position negative.
- [ ] **Sales-order allocation / reservations** _(suggested)_ — soft-reserve FG against open SOs to avoid double-promising stock.
- [ ] **Lot genealogy report** _(suggested)_ — which RM lots could have fed a given FG lot (recall support); builds on lot attribution.
- [ ] **Tests for financial paths** _(suggested)_ — valuation, costing, and QB sync, per `CONVENTIONS.md`.

## P3 — UX, labels & reporting

- [ ] **Headers on stock adjustment.**
- [ ] **Improve location-visibility UX** (the "what's on a location" view exists; refine it).
- [ ] **Inventory labels.**
- [ ] **Batch labels.**
- [ ] **Barcode / label scanning** _(suggested)_ — scan-driven receiving, moves, and cycle counts; pairs with the label work.
- [ ] **Core reports** _(suggested)_ — inventory valuation, stock-by-location, WIP aging.

## Later — New modules

> Large standalone initiatives. Schedule deliberately, after the core is solid.

- [ ] **Scheduling tool.**
- [ ] **Customer-service CRM.**
- [ ] **Sales CRM.**

## Recently completed

- [x] Pounds-canonical units through the system; kg→lb auto-convert on receipt _(verify the conversion-formula disclaimer is surfaced in the UI)_.
- [x] Location visibility — what is currently on a location.
- [x] Sub-locations — hierarchical building / aisle / rack with `bbb-a-nnn` codes.
