# Vyapaari+ Sprint Audit Progress Log

**Sprint Window**: 48 Hours  
**Assignee**: Mansi Agrawal (Lead Assignee / Permanent Intern ID TTI000066)  
**Branch**: `sprint/audit-fixes-integration`  
**Base**: `upstream/main` (`mohitraj8503/Vyapaari-Plus`)

---

## Phase A: Setup and Baseline
- [x] Verified git remotes (`origin` = Pranav9094/Vyapaari-Plus, `upstream` = mohitraj8503/Vyapaari-Plus)
- [x] Synced with `upstream/main`
- [x] Created sprint branch `sprint/audit-fixes-integration`
- [x] Configured `.git/info/exclude` to protect local transient notes and tasks
- [x] Audited open PRs on upstream repo:
  - PR #4 (`ymahalakshmi27-max`): Non-finite transfer costs
  - PR #6 (`moonknight95`): Inventory transfer idempotency
  - PR #7 (`Murali6383`): Stock transfer tracking API
  - PR #8 (`gayatridoke774-commits`): Transfer engine documentation
  - PR #9 (`nairjaanvi199-creator`): Merchant acquisition playbook
- [ ] Installing dependencies in `.venv` (in progress)
- [ ] Baseline test run & migration check

---
## Phase B: Codebase Audit & Bug Inventory
- [ ] Static analysis, linting, and compile checks
- [ ] Module-by-module audit
- [ ] Deliverable: `docs/audit/AUDIT_REPORT.md`

---
## Phase C: High-Priority Bug Fixes
- [ ] DB initialization and startup fixes
- [ ] Stock quantities, negative stock, idempotency
- [ ] Decimal money calculation and rounding policy
- [ ] Customer balances, supplier payables, PO totals
- [ ] Atomic stock transfers and rollback guarantees

---
## Phase D: GST Implementation Review
- [ ] CGST/SGST/IGST tax splits and odd-paise rules
- [ ] Place of supply & 15-character GSTIN validation with state code rules
- [ ] Round-off, discounts before tax, inclusive/exclusive pricing
- [ ] Reverse charge handling
- [ ] Credit / Debit note validations and reversals
- [ ] E-Invoice (IRN) schema compliance
- [ ] E-Way bill generation and threshold validations
- [ ] QR code generation & missing lib handling
- [ ] Invoice sequential numbering & FY reset

---
## Phase E: Multi-Store Inventory & Supplier Workflows
- [ ] Idempotency & state machine validation
- [ ] Stock conservation invariant: dispatch source decrement = receipt destination increment
- [ ] Discrepancy reconciliation (damaged, missing, excess)
- [ ] Supplier PO -> Goods Receipt (GRN) -> Payables lifecycle
- [ ] Transaction failure injection and rollback verification

---
## Phase F: Incomplete Functionality Completion
- [ ] Business setup & configuration
- [ ] POS Billing & cart management
- [ ] Customer Udhaar ledger operations
- [ ] Operational alerts (low stock, expiry, overdue Udhaar/payables)

---
## Phase G: Full Integration & Regression Testing
- [ ] End-to-end integration workflows (`tests/integration/`)
- [ ] Clean DB migration and boot verification
- [ ] Export validation (PDF & Excel data parity)
- [ ] Coverage assessment and test hardening

---
## Phase H: README.md Overhaul
- [ ] Document verified features
- [ ] Accurate architecture and setup guides
- [ ] Verified commands for Desktop UI, API, and test suite

---
## Phase I & J: Repository Hygiene, Pull Request & Submission
- [ ] .gitignore verification
- [ ] Clean git diff vs upstream/main
- [ ] PR Body generation (`docs/audit/PR_BODY.md`)
- [ ] Autonomous Pull Request creation
