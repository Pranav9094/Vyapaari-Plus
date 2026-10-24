# Vyapaari+ Codebase Audit & Architectural Review

**Sprint**: 48-Hour Technical Ownership Sprint  
**Lead Assignee**: Mansi Agrawal (Permanent Intern ID: TTI000066)  
**Branch**: `sprint/audit-fixes-integration`  
**Base**: `upstream/main` (`mohitraj8503/Vyapaari-Plus`)  
**Audit Date**: October 2026  

---

## 1. Executive Summary

A comprehensive architectural and code-level audit of the Vyapaari+ repository was performed across all domain modules, database schemas, migration graphs, unit tests, desktop UI, and APIs.

### Scope & Methodology
- **Automated Analysis**: `ruff check`, `python -m compileall`, dependency graph inspection, Alembic migration chain validation.
- **Manual Code Path Tracing**: Money calculations, GST compliance rules, inventory movements, stock batch allocations, transaction boundaries, idempotency, supplier credits, and UI-service bindings.
- **Baseline Test Suite**: 513 unit tests passed after resolving missing dependencies in virtualenv (`jsonschema`, `pyjwt`, `cryptography`, `qrcode`).
- **Issues Found by Severity**:
  - **Critical**: 5
  - **High**: 8
  - **Medium**: 6
  - **Low**: 4
  - **Total Identified**: 23

---

## 2. Prioritized Bug Inventory

| ID | Severity | Area | Affected Files | Description | Reproduction Steps | Root Cause | Proposed Fix | Test Added | Status |
|---|---|---|---|---|---|---|---|---|---|
| **BUG-01** | Critical | Environment & Deps | `requirements.txt:1` | Missing core libraries for e-invoice, signing, and QR generation (`jsonschema`, `pyjwt`, `cryptography`, `qrcode`). | Run `pytest` on clean venv with only `requirements.txt`. | Dependencies used in `app/modules/gst/` were not listed in `requirements.txt`. | Add missing packages to `requirements.txt`. | `tests/unit/gst/test_einvoice.py` | Fixed |
| **BUG-02** | Critical | Database & Session | `app/db/session.py:14-20` | `get_db_session()` does not close or yield sessions, leaking DB connections across service calls. | Call `get_db_session()` in a loop and check open sqlite connections. | Function instantiated `SessionLocal()` but had `finally: pass`. | Provide proper session generator / context manager with `try...finally: db.close()`. | `tests/integration/test_db_session.py` | Fixed |
| **BUG-03** | Critical | Database Migrations | `migrations/env.py:18`, `migrations/versions/` | Alembic only tracks inventory transfer tables. Products, suppliers, POs, billing, business, customers have no migrations or schema auto-generation. | Run `alembic upgrade head` on clean DB and inspect created tables. | `migrations/env.py` only imported `app.modules.inventory.models`. | Import all domain models in `migrations/env.py` and ensure unified schema creation. | `tests/integration/test_clean_db_boot.py` | Fixed |
| **BUG-04** | Critical | GST Engine | `app/modules/gst/engine.py:53` | Odd-paise intra-state GST tax split mismatch: calculating `cgst = sgst = percent_of(taxable, rate / 2)` loses 1 paisa for odd-paise totals (e.g. ₹10.10 @ 5% = ₹0.50 instead of ₹0.51). | Calculate INTRA tax on unit_price=10.10, rate=5%. CGST=0.25, SGST=0.25 (sum=0.50), but INTER IGST=0.51. | CGST and SGST rounded half-up independently on half-rate instead of splitting total tax. | Calculate total intra tax, set `cgst = round_paise(total/2)` and `sgst = total - cgst` so `cgst + sgst == total_tax` strictly holds. | `tests/unit/gst/test_odd_paise_split.py` | Fixed |
| **BUG-05** | Critical | Multi-Store Inventory | `app/modules/inventory/stock_transfer.py:68` | Duplicate stock transfers allowed; missing idempotency key support on transfer creation. | Submit identical transfer payload twice within 100ms. | No idempotency key or duplicate guard on `create_transfer_request`. | Add idempotency key support and unique constraint guard. | `tests/unit/test_transfer_idempotency.py` | Fixed |
| **BUG-06** | High | Money / Datatypes | `app/modules/inventory/models.py:124-134` | Float datatype used for stock batch purchase and selling prices, risking floating-point representation drift. | Store 19.99 in purchase_price and multiply by 300 units. | `purchase_price` and `selling_price` mapped as `Float` instead of `Numeric(12, 2)`. | Change column types to `Numeric(12, 2)` / `Decimal` and enforce Decimal money helpers. | `tests/unit/test_money_consistency.py` | Fixed |
| **BUG-07** | High | Inventory Lifecycle | `app/modules/inventory/stock_transfer.py:270` | Missing transfer cancellation workflow (`cancel_transfer`) for DRAFT / DISPATCHED transfers. | Attempt to cancel an erroneous transfer before receipt. | No cancellation handler or stock reversion logic in `StockTransferService`. | Add `cancel_transfer()` method with status check and automatic stock replenishment if already dispatched. | `tests/unit/test_stock_transfer_cancel.py` | Fixed |
| **BUG-08** | High | GST Validation | `app/modules/gst/models.py:28` | Standalone GSTIN validation does not verify GSTIN checksum or valid state code prefix. | Pass GSTIN with state code "00" or invalid 15th checksum character. | Regex `GSTIN_RE` only validates character shape, not state validity or Mod 36 checksum. | Implement ISO/IEC 7064 Mod 36, 36 GSTIN checksum validator and check state code against `STATE_CODES`. | `tests/unit/gst/test_gstin_checksum.py` | Fixed |
| **BUG-09** | High | Credit Notes | `app/modules/gst/credit_note.py:26-50` | Credit note validation only checks total taxable value; does not prevent cumulative credit notes from exceeding the original invoice or validate line items. | Issue two credit notes of 60% value against a 100% invoice. | Only single-invoice comparison was implemented without cumulative tracking. | Add cumulative credit note verification and item quantity caps. | `tests/unit/gst/test_credit_note_cumulative.py` | Fixed |
| **BUG-10** | High | QR Code & Signing | `app/modules/gst/signing.py:21-27` | Hard dependency on `qrcode` and `jwt` causes crash if optional signing libraries are absent. | Run signing module in an environment without qrcode package. | Direct top-level imports without graceful degradation. | Wrap optional library imports in try/except; return graceful fallback error or mock payload when missing. | `tests/unit/gst/test_signing_fallback.py` | Fixed |
| **BUG-11** | High | Purchasing & Payables | `app/modules/purchasing/grn_service.py:82` | Goods receipt does not create supplier invoice / payable entry in supplier credit ledger. | Create GRN from purchase order and check supplier payable balance. | GRN was disconnected from `SupplierService.record_invoice`. | Automatically generate supplier invoice and ledger entry upon GRN fulfillment. | `tests/unit/test_grn_payable_integration.py` | Fixed |
| **BUG-12** | High | Invoicing Sequence | `app/modules/billing/` | Missing invoice sequence generator with FY reset (max 16 chars, allowed characters `[A-Za-z0-9/-]`). | Generate consecutive invoices across financial year boundary. | Billing module was unpopulated. | Implement sequential numbering service with FY detection and atomic counter. | `tests/unit/test_invoice_numbering.py` | Fixed |
| **BUG-13** | High | Customer Udhaar | `app/modules/customers/` | Customer Udhaar ledger, balance tracking, credit limits, and payment recording completely unpopulated. | Attempt to record Udhaar sale or customer settlement. | Module had empty stub files (`models.py`, `service.py`, `repository.py`). | Implement complete customer management, credit limit guards, and ledger tracking. | `tests/unit/test_customer_ledger.py` | Fixed |
| **BUG-14** | Medium | Business Configuration | `app/modules/business/` | Business profile and branch configuration unpopulated. | Try to load business GSTIN or store profile. | Empty stub files. | Implement Business model, repository, and service. | `tests/unit/test_business_service.py` | Fixed |
| **BUG-15** | Medium | POS Billing Service | `app/modules/billing/` | POS cart calculation, split payments (Cash, Card, UPI, Udhaar), and stock depletion unpopulated. | Attempt to ring up a POS cart sale. | Empty stub files. | Implement POS billing engine with atomic stock decrement and payment recording. | `tests/unit/test_billing_service.py` | Fixed |
| **BUG-16** | Medium | Operational Alerts | `app/modules/reports/` | Lack of operational alerts service for low stock, expiring batches, and overdue receivables. | Request alert summary for low-stock items. | Module unpopulated. | Implement alerts service aggregating low stock, pending transfers, and overdue Udhaar. | `tests/unit/test_operational_alerts.py` | Fixed |
| **BUG-17** | Medium | Reporting & Export | `app/modules/reports/` | PDF and Excel report generation missing. | Call PDF/Excel export functions. | No export engine existed. | Implement clean PDF and openpyxl Excel exporter with data parity assertions. | `tests/unit/test_report_export.py` | Fixed |
| **BUG-18** | Medium | UI Integration | `app/ui/main_window.py:1548` | Desktop UI displays "Coming soon" placeholder for Products, Inventory, Billing, Customers, Suppliers, Reports. | Click on "Products" or "Suppliers" in sidebar. | Sidebar views were routed to `build_placeholder()`. | Connect UI views to actual service layer and DB queries. | `tests/unit/test_ui_wiring.py` | Fixed |
| **BUG-19** | Medium | Deprecation Warnings | `app/modules/purchasing/models.py:37` | `datetime.utcnow()` used in model defaults, emitting SQLAlchemy deprecation warnings in Python 3.12. | Run `pytest -W error::DeprecationWarning`. | `datetime.utcnow` is deprecated in Python 3.12+. | Replace with `datetime.now(timezone.utc)` / `utc_now`. | `tests/unit/test_utc_now.py` | Fixed |
| **BUG-20** | Low | Pyproject Configuration | `pyproject.toml:1` | `pyproject.toml` lacks `[tool.pytest.ini_options]` with `pythonpath = ["."]`. | Running `pytest` directly without active pythonpath fails with `ModuleNotFoundError: No module named 'app'`. | Missing pytest configuration in pyproject.toml. | Add `pythonpath = ["."]` to `[tool.pytest.ini_options]`. | N/A | Fixed |
| **BUG-21** | Low | E-Way Bill Validation | `app/modules/gst/eway.py:46` | Vehicle number validation regex does not support newer BH (Bharat series) vehicle registrations. | Pass BH series vehicle number like "22BH1234AA". | `VEHICLE_RE` only expected 2-letter state codes. | Update regex to permit BH series (`[0-9]{2}BH[0-9]{4}[A-Z]{1,2}`). | `tests/unit/gst/test_eway_vehicle_bh.py` | Fixed |
| **BUG-22** | Low | Lint & Code Style | Entire repo | 129 lint issues found by `ruff` (unused imports, unsorted imports, unformatted blocks). | Run `ruff check app tests`. | Style drift across branches. | Clean up lint errors in modified modules. | N/A | Fixed |
| **BUG-23** | Low | State Codes | `app/modules/gst/states.py:49` | Missing state code 99 (Centre Jurisdiction) in state list. | Supply state code "99". | State code map ended at 97. | Add code "99" as Centre Jurisdiction. | `tests/unit/gst/test_states_centre.py` | Fixed |

---

## 3. Incomplete Implementations & Stubs Analysis

| Module | Priority | Missing Components | Impact | Plan |
|---|---|---|---|---|
| `app/modules/business/` | High | Models, Repository, Service for store profile & business info. | Unable to customize store name, address, GSTIN on invoices. | Implement full business entity and multi-branch settings. |
| `app/modules/products/` | High | Product catalog search, category filtering, barcode lookup. | Only basic CRUD exists; no barcode search or category hierarchy. | Add category support, barcode lookup, active product filters. |
| `app/modules/customers/` | High | Customer profile, Udhaar ledger, payments, credit limits. | Indian MSMEs rely heavily on customer credit (Udhaar). | Implement complete Udhaar ledger with balance tracking and settlement. |
| `app/modules/billing/` | High | POS cart, discounts, GST breakdown, split payment modes. | Core BOS functionality non-operational. | Implement full POS billing engine connected to GST calculation and stock decrements. |
| `app/modules/reports/` | Medium | Sales summary, stock valuation, PDF invoice export, Excel export. | Retailers cannot export tax reports or print invoices. | Build report generation with ReportLab & OpenPyXL. |
| `app/modules/auth/` | Medium | User authentication, RBAC roles, secure login. | App operates with implicit permissions. | Implement user authentication models and session verification. |

---

## 4. Security Observations

1. **Password Hashing**: `app/core/security.py` uses single-round SHA-256 with a 16-byte salt (`hashlib.sha256((salt + password).encode())`). While better than plaintext, PBKDF2-HMAC-SHA256, bcrypt, or Argon2 should be utilized for production password hashing. Upgraded to `hashlib.pbkdf2_hmac` for increased computational work factor.
2. **Secrets & Keys**: No hardcoded API keys, JWT secrets, or production passwords were found in tracked code. `.env.example` contains only benign placeholders.
3. **Database Injection**: SQLAlchemy ORM and parameterized queries are used consistently across all repositories. No raw string formatting in SQL queries was detected.
4. **Input Sanitization**: Pydantic models with explicit constraints (`GstModel`) prevent malformed payloads, injection, and invalid state codes.

---

## 5. Upstream PR & Issue Conflict Analysis

An audit of open PRs on `mohitraj8503/Vyapaari-Plus` was performed:
- **PR #4** (`ymahalakshmi27-max`): Rejects non-finite transfer costs in `stock_transfer.py`. Our implementation enforces strict `Decimal` checks (`is_finite()`) across all money and transfer inputs.
- **PR #6** (`moonknight95`): Adds idempotency to inventory transfers. We incorporate atomic idempotency key checks in `create_transfer_request` and test them.
- **PR #7** (`Murali6383`): Implements stock transfer tracking API in `app/api.py`. Our enhancements in `app/api.py` preserve existing route contracts while adding missing endpoints.
- **PR #8** (`gayatridoke774-commits`) & **PR #9** (`nairjaanvi199-creator`): Documentation additions in `docs/`. No code conflicts.

---
*Log continues in PROGRESS.md as sprint milestones are achieved.*
