# FW3 Backlog

Prioritized backlog for the fw3 ERP, reconciled with the repo's `TODO.md` and recent
development. Items are grouped into priority tiers ordered by **go-live readiness** —
what it takes to retire the legacy system, then complete it, harden it, and grow it.
Original to-do entries are kept close to verbatim, with new **_(suggested)_** items
folded in. Each tier opens with the reasoning behind its rank.

> [!NOTE]
> Reconciliation: the inventory life-cycle (weighted-average ledger, purchasing, sales,
> production + pack-off, quality/quarantine, scrap, vendor returns, cycle counts),
> physical + hierarchical locations, container inventory, RM/FG regulatory data, ERP
> business variables, pounds-canonical units (kg→lb on receipt), and the P0 correctness
> fixes (delete button, "Stock" page) are all **done**. This list is the work that
> remains. See **Recently completed** at the bottom for the items cleared off `TODO.md`.

## P0 — Go-live blockers

> The system can't replace the legacy Formula Web / MSSQL stack without these:
> the accounting round-trip, the existing data, and a ledger that can be trusted as
> the book of record. Do first.

- [ ] **Complete QBWC sync** — the QuickBooks Desktop round-trip (items, customers, receipts/bills, invoices). Accounting depends on it.
- [ ] **Legacy data migration** _(suggested)_ — import items, on-hand balances, open POs/SOs, and lot history from the Formula Web / MSSQL system. A hard go-live blocker.
- [ ] **Enforce append-only ledgers at the DB level** _(suggested)_ — deny `UPDATE`/`DELETE` on `InventoryTxn`, `LocationMove`, and `ContainerTxn`; corrections become reversing entries. This is the real guarantee behind the delete-button fix.
- [ ] **Record the actor on `InventoryTxn`** _(suggested)_ — add `createdById`; the quantity/cost ledger has no actor today (only `LocationMove`, `ContainerTxn`, and `ScrapRecord` do). Required for a complete audit trail.
- [ ] **Attribute ledger lines to lots (`lotId` on `InventoryTxn`)** _(suggested)_ — link quantity lines to `ReceivedLot` for traceability through QC, consumption, and shipment (lost in the WIP blend by design). Foundation for the recall/genealogy report.

## P1 — Financial completeness

> The money paths that AR and sales need day to day. The system runs without them, but
> the books are incomplete until they land.

- [ ] **Partial payments** — record payments against an SO; mark it "paid" when `sum(partial_payment.amount) == due`.
- [ ] **Issue refunds** — for overpayment and approved cancellations.
- [ ] **Profit margin per customer ranking** — drive SO pricing/margin off the customer's A–D rating; extends the now-done "show cost on SO" work.
- [ ] **Tests for financial paths** _(suggested)_ — valuation, costing, COGS-at-ship, and QB sync, per `CONVENTIONS.md`.

## P2 — Workflow hardening

> Correctness guards and floor/lab flows that close real gaps. High value, but the
> system already runs without them.

- [ ] **Location rules** — e.g. no move back to the receiving dock after warehousing; valid-transition guards on `LocationMove`.
- [ ] **Allow packing before QC** — relax QC-gated pack-off where policy permits, with a hold/release rule so unapproved FG still cannot ship.
- [ ] **Separate floor from 2 lb pours** — distinguish full production floor pours from 2 lb (sample/lab) pours in the compounder flow.
- [ ] **Query robot RMs** — surface which raw materials the dosing robot holds / can dispense.
- [ ] **Sell RM to customer / order FG from vendor** — the schema already supports trading both directions (item-or-container PO/SO lines, the `trade_both_directions` migration); verify the UI exposes both and the ledger/QB postings are correct.
- [ ] **Oversell / negative-stock guards** _(suggested)_ — confirm consume/ship/transfer can never drive an (item, status) position negative.
- [ ] **Sales-order allocation / reservations** _(suggested)_ — soft-reserve FG against open SOs to avoid double-promising stock.

## P3 — Reporting & labels

> Visibility and the physical paperwork. Needed for real operations but not blocking.

- [ ] **Production reports.**
- [ ] **Shipping reports.**
- [ ] **Sales reports.**
- [ ] **Labels** — receiving, retain, shipping, sample, and batch labels.
- [ ] **Barcode / label scanning** _(suggested)_ — scan-driven receiving, moves, and cycle counts; pairs with the label work.
- [ ] **Lot genealogy report** _(suggested)_ — which RM lots could have fed a given FG lot (recall support); builds on lot attribution.

## P4 — UX & RBAC

> Polish and per-role surfacing.

- [ ] **Role-specific views** — surface only the relevant screens/actions per user role (RBAC-driven UI).
- [ ] **Improve location-visibility UX** (the "what's on a location" view exists; refine it).
- [ ] **Headers on stock adjustment.**

## Architecture / tech debt

> Decoupling work that pays off as integrations multiply. Not user-facing.

- [ ] **Accounting Adapter** — decouple QuickBooks behind a generic accounting adapter, so a different ledger can be swapped in.
- [ ] **Regulatory Adapter** — put the FormPak+ integration behind a regulatory adapter (mirrors the accounting adapter).
- [ ] **Two-ledger schema clarity** — the `(ItemStock, ItemStockLocation)` and `(InventoryTxn, LocationMove)` split reads as redundant at a glance; document/reconcile the rationale (ties to the recorded two-ledger trade-off notes).

## Later — New modules

> Large standalone initiatives. Schedule deliberately, after the core is solid.

- [ ] **Scheduling tool** _(init done)_ — build out from the scaffolding already in place.
- [ ] **Customer-service CRM.**
  - [ ] Mark an SO "confirmed" / "requested" when the customer pays.
  - [ ] Customer purchase & cost history.
  - [ ] Merge customer entries (link ids, allow unmerge; sync history, contacts, addresses).
- [ ] **Sales CRM.**

## Recently completed

- [x] **Broken inventory delete button** — removals now go through "adjust out" (recorded `ADJUSTMENT` lines); items with `InventoryTxn` history can't be deleted.
- [x] **Broken references to the "Stock" page** — dangling links from the Stock→Inventory merge fixed.
- [x] **Separate Inventory from Item Details** — quantity/cost live in `ItemStock` + the ledger, not on `InventoryItem`.
- [x] **Containers showing in inventory** — and removal of the ad-hoc "new item" / opening-stock entry points (use adjustments or POs).
- [x] **Container inventory** — `Container` / `ContainerStock` / `ContainerTxn` packaging stock.
- [x] **Facilitate scrap in all inventory stages** — first-class `ScrapRecord` across `INV`, `WIP`, `QUARANTINE`.
- [x] **QC-failed RMs back to vendor** — `VendorReturn` / RMA flow off the `REJECTED` quarantine state.
- [x] **Compounder Tool API** — formulation details + work-order list exposed; consumption/batch-progress reports accepted back.
- [x] **Cycle count tool** — count entry → signed `ADJUSTMENT` lines per (item, location); blind-count support.
- [x] **Show cost when generating an SO** — unit cost per line, default profit margin, below-cost warning ("don't lose our shirt").
- [x] **RM & FG regulatory data** — IFRA category limits for RMs; FormPak+ profile + IFRA levels for FGs.
- [x] **ERP business variables** — working hours, default profit margin, pph/workstation, production efficiency, company holidays, production cost factor.
- [x] **Pounds-canonical units** through the system; kg→lb auto-convert on receipt _(verify the conversion-formula disclaimer is surfaced in the UI)_.
- [x] **Location visibility** — what is currently on a location.
- [x] **Sub-locations** — hierarchical building / aisle / rack with `bbb-a-nnn` codes.
- [x] **Compounder tool API documentation.**
