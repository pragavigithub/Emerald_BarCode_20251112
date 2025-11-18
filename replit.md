# Warehouse Management System (WMS)

## Overview
A Flask-based Warehouse Management System (WMS) designed to streamline inventory operations by integrating seamlessly with SAP. The system focuses on enhancing efficiency, accuracy, and control over warehouse logistics through functionalities such as barcode scanning, goods receipt, pick list generation, and inventory transfers. It aims to minimize manual errors and maximize throughput for small to medium-sized enterprises by providing real-time data synchronization with SAP.

## User Preferences
*   Keep MySQL migration files updated when database schema changes occur
*   SQL query validation should only run on initial startup, not on every application restart

## Recent Updates

### November 18, 2025 - Multi GRN Edit Functionality (Complete)

**Feature:** Draft Multi GRN batches can now be edited after initial creation.

**Implementation:**
- **Edit Entry Point:** New `/multi-grn/<batch_id>/edit` route with draft-only and owner-only access controls
- **Index UI:** Added "Edit" button (orange) for draft status batches, positioned between View and Delete buttons
- **Step 2 (PO Selection) Editing:**
  - Displays existing POs with remove buttons
  - Allows adding more POs from available list
  - Prevents removing last PO (batch must have ≥1 PO)
  - Page reload after PO removal to refresh UI
- **Step 3 (Line Selection) Editing:**
  - Added "Remove" button for each line item in step3_detail.html
  - Prevents removing last line item (batch must have ≥1 line)
  - Cascade deletes batch_details and serial_details on line removal
  - Page reload after line removal to refresh counters
- **Step 4 & 5 (Details & QR):** Already supported via existing "Add Item" and "Print Labels" buttons

**User Workflow:**
1. Click "Edit" on draft batch from index page
2. Modify PO selection (Step 2) - add/remove POs
3. Modify line selection (Step 3) - add/remove line items
4. Update batch/serial details using "Add Item" button
5. Regenerate QR labels using "Print Labels" button
6. Submit when ready for QC approval

**Validation:**
- Only draft batches can be edited
- Only batch owner can edit
- Batch must always have at least 1 PO and 1 line item
- Clear error messages when validation fails

**Files Modified:** `modules/multi_grn_creation/routes.py`, `modules/multi_grn_creation/templates/multi_grn/index.html`, `modules/multi_grn_creation/templates/multi_grn/step2_select_pos.html`, `modules/multi_grn_creation/templates/multi_grn/step3_detail.html`

---

### November 17, 2025 - Multi GRN Integer Pack Distribution (Complete)

**Issue:** QR labels displayed decimal quantities per pack (e.g., "Qty per Pack: 110.25"), making item counting impractical.

**Requirement:** Quantity per pack should be integers only. When total quantity is not evenly divisible, first packs get extra units.

**Implementation:**
- ✅ Created `distribute_quantity_to_packs()` helper function with ROUND_HALF_UP rounding
- ✅ Updated all qty_per_pack calculations to use integer division with ROUND_HALF_UP
- ✅ Modified QR label generation to calculate and display individual pack quantities
- ✅ Preserved total quantities (no loss from rounding)

**Examples:**
- 11 ÷ 3 packs = [4, 4, 3] ✓
- 110.5 ÷ 4 packs → rounds to 111 → [28, 28, 28, 27] ✓
- 110.25 ÷ 4 packs → rounds to 110 → [28, 28, 27, 27] ✓

**Result:**
- **Before:** 110.25 ÷ 4 packs → all packs show "27.56" (decimal)
- **After:** 110.25 ÷ 4 packs → packs show 28, 28, 27, 27 (integers)
- Total quantity preserved through smart rounding (ROUND_HALF_UP)
- First packs receive extra units when quantity doesn't divide evenly

**Files:** `modules/multi_grn_creation/routes.py` (lines 23-57, 1091, 2044, 2075, 1718-1719), `migrations/mysql/changes/2025-11-17_multi_grn_integer_pack_distribution.md`

---

### November 17, 2025 - Multi GRN Single Batch Generation Fix (Complete)

**Issue:** When entering "Number of Packs/Bags = 2", the system created 2 separate batch entries in the SAP JSON with split quantities, instead of using packs for QR label identification only.

**Root Cause:** Batch generation logic created multiple `MultiGRNBatchDetails` records (one per pack), causing the JSON builder to generate multiple batch entries.

**Fix:**
- ✅ Modified batch generation to create SINGLE batch_detail record with full quantity
- ✅ Stored `no_of_packs` and `qty_per_pack` fields for QR label generation only
- ✅ Updated QR label generation to create multiple labels from single batch record
- ✅ Added ItemCode field to QR label display

**Result:**
- **Before:** Number of Packs = 2 → JSON has 2 BatchNumbers entries (qty 3, qty 2)
- **After:** Number of Packs = 2 → JSON has 1 BatchNumbers entry (qty 5)
- QR labels still generate correctly: 2 labels showing "1 of 2" and "2 of 2"
- All QR labels now display the ItemCode

**Files:** `modules/multi_grn_creation/routes.py` (lines 1038-1086, 1656-1759), `modules/multi_grn_creation/templates/multi_grn/step3_detail.html` (line 633), `migrations/mysql/changes/2025-11-17_multi_grn_single_batch_generation.md`

---

### November 17, 2025 - Multi GRN Bug Fixes (Complete)

#### 1. QR Label Duplication Fix
**Issue:** Batch-managed items generated 4 QR labels instead of 2 when Number of Bags = 2

**Root Cause:** Nested loop creating multiple labels per batch_detail record
- Each batch_detail already represents one pack/bag
- Code was looping again over num_packs for each batch_detail
- Result: 2 batch_details × 2 num_packs = 4 labels (incorrect)

**Fix:**
- ✅ Removed nested loop in batch label generation
- ✅ Each batch_detail now generates exactly ONE label
- ✅ Corrected pack numbering: "1 of 2", "2 of 2"
- ✅ total_packs calculated from len(batch_details)

**Result:**
- Number of Bags = 2 → 2 QR labels (correct)
- Each label prints on separate page (existing CSS confirmed)

**Files:** `modules/multi_grn_creation/routes.py` (lines 1649-1705), `migrations/mysql/changes/2025-11-17_multi_grn_qr_label_duplication_fix.md`

#### 2. Empty Receive Qty Field Error Fix
**Issue:** Step 3 crashed with decimal.InvalidOperation when Receive Qty field left empty

**Root Cause:** 
- Empty form input ("") converted directly to Decimal without validation
- SAP OpenQuantity could be None/empty causing same error
- Users expected to select items at Step 3, enter quantities at Step 4

**Fix:**
- ✅ Added comprehensive input sanitization for SAP OpenQuantity values
- ✅ Handle empty/None values with safe Decimal('0') default
- ✅ Form input validation with try/except for conversion errors
- ✅ Fallback to open_qty with logging for diagnostics
- ✅ Skip lines with selected_qty <= 0

**Result:**
- Users can now leave Receive Qty blank at Step 3 (item selection only)
- Quantities can be entered in Step 4 (Line Item Details)
- Robust handling of invalid SAP data or form input

**Files:** `modules/multi_grn_creation/routes.py` (lines 242-259)

---

### November 14, 2025 - Multi GRN Critical Bug Fixes (Complete)
**Issue:** Standard items were incorrectly including BatchNumbers section in SAP JSON, causing API errors

**Root Cause:** Three critical bugs preventing proper SAP item validation:
1. **Wrong field names** - Code looked for `batch_required`/`serial_required` but SAP returns `batch_managed`/`serial_managed`
2. **Missing validation** - PO line items never validated with SAP, missing batch/serial flags
3. **Wrong values** - manage_method used 'B'/'S'/'N' instead of SAP's 'A'/'R'

**Fixes:**
- ✅ Fixed all 3 item addition paths to use correct SAP field names
- ✅ Added SAP validation when selecting PO line items
- ✅ Changed manage_method to use SAP's actual values

**Result:**
- Standard items → No BatchNumbers (correct)
- Batch items → Include BatchNumbers (correct)
- Serial items → Include SerialNumbers (correct)
- Quantity-managed → Include BatchNumbers for lot consolidation (correct)

**Files:** `modules/multi_grn_creation/routes.py` (lines 253-285, 1061-1063, 1756-1792)

## System Architecture
The system is built on a Flask web application backend, utilizing Jinja2 for server-side rendering. A core architectural decision is the deep integration with the SAP B1 Service Layer API for all critical warehouse operations, ensuring data consistency and real-time updates. PostgreSQL is the primary database target for cloud deployments, with SQLite serving as a fallback. User authentication uses Flask-Login with robust role-based access control. The application is designed for production deployment using Gunicorn with autoscale capabilities.

**UI/UX Decisions:**
*   Intuitive workflows for managing inventory, including serial number transfers and real-time validation against SAP B1.
*   Dynamic dropdowns for bin locations, populated from SAP B1.
*   Enhanced GRPO workflow with read-only warehouse fields automatically populated from Purchase Order data.
*   Comprehensive pagination, filtering, and search functionalities across key modules (GRPO Details, Sales Order Against Delivery Details, Inventory Counting History, Inventory Transfer).

**Technical Implementations:**
*   **SAP B1 Integration:** Utilizes a dedicated `SAPMultiGRNService` class for secure and robust communication with the SAP B1 Service Layer, including SSL/TLS verification and optimized OData filtering. Conditional handling of batch/serial numbers in SAP JSON prevents API errors.
*   **Modular Design:** New features are implemented as modular blueprints with their own templates and services.
*   **Frontend:** Jinja2 templating with JavaScript libraries like Select2 for enhanced UI components.
*   **Error Handling:** Comprehensive validation and error logging for API communications and user inputs.
*   **Optimized SAP SQL Query Validation:** SQL query validation runs only on initial startup using a flag-based system.
*   **Database Migrations:** A comprehensive MySQL migration tracking system is in place for schema changes, complementing the primary PostgreSQL strategy.
*   **GRPO Integer Quantity Distribution:** Implements intelligent integer quantity distribution per pack, ensuring no decimal quantities on QR labels.
*   **Persistent QR Scan State:** Uses a database-backed `TransferScanState` model for persistent pack tracking during inventory transfers, avoiding session limitations.

**Feature Specifications:**
*   **User Management:** Comprehensive authentication, role-based access, and self-service profile management.
*   **GRPO Management:** Standard Goods Receipt PO processing, intelligent batch/serial field management, and a multi-GRN module for batch creation from multiple Purchase Orders via a 5-step workflow with SAP B1 integration and QR label generation. Includes dynamic SAP bin location lookup.
*   **Inventory Transfer:** Enhanced module for creating inventory transfer requests with document series selection, SAP B1 validation, and robust QR label scanning with duplicate prevention and quantity accumulation.
*   **Direct Inventory Transfer:** Barcode-based inventory transfer module with automatic serial/batch detection, real-time SAP B1 validation, warehouse and bin selection, QC approval workflow, and direct posting to SAP B1 as StockTransfers. Includes camera-based scanning.
*   **Sales Order Against Delivery:** Module for creating Delivery Notes against Sales Orders with SAP B1 integration, including SO series selection, cascading dropdown for open SO document numbers, item picking with batch/serial validation, and individual QR code label generation.
*   **Pick List Management:** Generation and processing of pick lists.
*   **Barcode Scanning:** Integrated camera-based scanning for various modules (GRPO, Bin Scanning, Pick List, Inventory Transfer, Barcode Reprint).
*   **Inventory Counting:** SAP B1 integrated inventory counting with local database storage for tracking, audit trails, user tracking, and timestamps, including a comprehensive history view.
*   **Branch Management:** Functionality for managing different warehouse branches.
*   **Quality Control Dashboard:** Provides a unified oversight for quality approval workflows across Multi GRN, Direct Transfer, and Sales Delivery modules, with SAP B1 posting integration upon approval.

## External Dependencies
*   **SAP B1 Service Layer API**: For all core inventory and document management functionalities (GRPO, pick lists, inventory transfers, serial numbers, business partners, inventory counts).
*   **PostgreSQL**: Primary relational database for production environments.
*   **SQLite**: Local relational database for development and initial setup.
*   **Gunicorn**: WSGI HTTP server for deploying the Flask application in production.
*   **Flask-Login**: Library for managing user sessions and authentication.