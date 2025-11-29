# Warehouse Management System (WMS)

## Overview
A Flask-based Warehouse Management System (WMS) designed to streamline inventory operations by integrating seamlessly with SAP. The system focuses on enhancing efficiency, accuracy, and control over warehouse logistics through functionalities such as barcode scanning, goods receipt, pick list generation, and inventory transfers. It aims to minimize manual errors and maximize throughput for small to medium-sized enterprises by providing real-time data synchronization with SAP. The project's ambition is to provide a robust, scalable, and user-friendly solution for modern warehouse management challenges, leveraging existing SAP infrastructure while introducing advanced features for operational excellence.

## User Preferences
*   Keep MySQL migration files updated when database schema changes occur
*   SQL query validation should only run on initial startup, not on every application restart

## System Architecture
The system is built on a Flask web application backend, utilizing Jinja2 for server-side rendering. A core architectural decision is the deep integration with the SAP B1 Service Layer API for all critical warehouse operations, ensuring data consistency and real-time updates. PostgreSQL is the primary database target for cloud deployments, with SQLite serving as a fallback. User authentication uses Flask-Login with robust role-based access control. The application is designed for production deployment using Gunicorn with autoscale capabilities.

**UI/UX Decisions:**
*   Intuitive workflows for managing inventory, including serial number transfers and real-time validation against SAP B1.
*   Dynamic dropdowns for bin locations, populated from SAP B1.
*   Enhanced GRPO workflow with read-only warehouse fields automatically populated from Purchase Order data.
*   Comprehensive pagination, filtering, and search functionalities across key modules (GRPO Details, Sales Order Against Delivery Details, Inventory Counting History, Inventory Transfer).

**Technical Implementations:**
*   **SAP B1 Integration:** Utilizes a dedicated `SAPMultiGRNService` class for secure and robust communication with the SAP B1 Service Layer, including SSL/TLS verification and optimized OData filtering. Conditional handling of batch/serial numbers in SAP JSON prevents API errors.
*   **Modular Design:** New features are implemented as modular blueprints with their own templates and services. **All module blueprints use absolute template paths** (`Path(__file__).resolve().parent / 'templates'`) for PyInstaller .exe build compatibility (updated Nov 20, 2025).
*   **Frontend:** Jinja2 templating with JavaScript libraries like Select2 for enhanced UI components.
*   **Error Handling:** Comprehensive validation and error logging for API communications and user inputs.
*   **Optimized SAP SQL Query Validation:** SQL query validation runs only on initial startup using a flag-based system.
*   **Database Migrations:** A comprehensive MySQL migration tracking system is in place for schema changes, complementing the primary PostgreSQL strategy.
*   **GRPO Integer Quantity Distribution:** Implements intelligent integer quantity distribution per pack, ensuring no decimal quantities on QR labels.
*   **Persistent QR Scan State:** Uses a database-backed `TransferScanState` model for persistent pack tracking during inventory transfers, avoiding session limitations.

**Feature Specifications:**
*   **User Management:** Comprehensive authentication, role-based access, and self-service profile management.
*   **GRPO Management:** Standard Goods Receipt PO processing, intelligent batch/serial field management, and a multi-GRN module for batch creation from multiple Purchase Orders via a 5-step workflow with SAP B1 integration and QR label generation. Includes dynamic SAP bin location lookup, QC workflow with line-by-line verification, unique QR identifiers per pack, and editing of draft batches.
*   **Inventory Transfer:** Enhanced module for creating inventory transfer requests with document series selection, SAP B1 validation, and robust QR label scanning with duplicate prevention and quantity accumulation.
*   **Direct Inventory Transfer:** Barcode-based inventory transfer module with automatic serial/batch detection, real-time SAP B1 validation, warehouse and bin selection, QC approval workflow, and direct posting to SAP B1 as StockTransfers. Includes camera-based scanning.
*   **Sales Order Against Delivery:** Module for creating Delivery Notes against Sales Orders with SAP B1 integration, including SO series selection, cascading dropdown for open SO document numbers, item picking with batch/serial validation, and individual QR code label generation.
*   **Pick List Management:** Generation and processing of pick lists.
*   **Barcode Scanning:** Integrated camera-based scanning for various modules (GRPO, Bin Scanning, Pick List, Inventory Transfer, Barcode Reprint).
*   **Inventory Counting:** SAP B1 integrated inventory counting with local database storage for tracking, audit trails, user tracking, and timestamps, including a comprehensive history view.
*   **Branch Management:** Functionality for managing different warehouse branches.
*   **Quality Control Dashboard:** Provides a unified oversight for quality approval workflows across Multi GRN, Direct Transfer, and Sales Delivery modules, with SAP B1 posting integration upon approval.

### Multi GRN QC Approval Workflow (Updated Nov 22, 2025)
The Multi GRN module now implements a comprehensive line-by-line verification workflow:

**Status Flow:**
1. **Draft** → Items created by GRN team with batch/serial details
2. **Submitted** → Batch submitted for QC review
3. **QC Review** → QC team scans QR codes line-by-line to verify each item
   - Each batch/serial detail has status: `pending` → `verified`
   - QC review page shows real-time verification progress
   - "Approve & Post to SAP" button is disabled until ALL items are verified
4. **Approved & Posted** → All items verified, batch posted to SAP B1 as consolidated GRN

**Key Features:**
- Line-by-line QR code scanning verification using barcode scanner device (no camera-based scanning)
- **Quantity validation**: System compares scanned QR quantity with database record pack quantity before marking as verified
- Real-time progress tracking showing verified vs. total items
- Barcode scanner support with auto-focus input field
- Server-side validation ensures all items are verified before SAP posting
- QC notes captured for audit trail
- Verification status persists in database (multi_grn_batch_details.status, multi_grn_serial_details.status)
- Error handling for quantity mismatches to prevent incorrect verification

**Database Changes:**
- Added `status` column (VARCHAR(20), default 'pending') to:
  - `multi_grn_batch_details` table
  - `multi_grn_serial_details` table
- MySQL migration file updated accordingly

### Multi GRN Batch Label Linking Table (Added Nov 23, 2025)
To solve the issue where entering multiple packs (e.g., 3 packs) only generated 1 QR label, a new linking table `multi_grn_batch_details_label` was created to track each individual pack with unique GRN numbers.

**Problem Solved:**
- **Before**: Quantity=7, Packs=3 → Only 1 QR label generated ("1 of 1", Qty: 3)
- **After**: Quantity=7, Packs=3 → 3 unique QR labels (Pack 1/3: Qty 3, Pack 2/3: Qty 2, Pack 3/3: Qty 2)

**Database Structure:**
- `multi_grn_batch_details` stores the parent record (total quantity, no_of_packs)
- `multi_grn_batch_details_label` stores individual pack records with:
  - Unique `grn_number` per pack (e.g., MGN-19-43-1-1, MGN-19-43-1-2, MGN-19-43-1-3)
  - `pack_number` (1, 2, 3, etc.)
  - `qty_in_pack` (distributed integer quantity)
  - `barcode` and `qr_data` for each label
  - `printed` tracking for audit trail

**Key Features:**
- Automatic integer quantity distribution across packs (7 ÷ 3 = [3, 2, 2])
- Each pack gets a unique GRN number for tracking
- QR label generation reads from the label table
- Print tracking per label
- Cascading delete protection

**Migration Files:**
- PostgreSQL: Applied to development database
- MySQL: `migrations/mysql_multi_grn_batch_details_label_table.sql`

### Multi-GRN QR Labels with Bin Location (Added Nov 29, 2025)
QR code labels in the Multi-GRN module now include Bin Location information for better warehouse tracking.

**QR Code JSON Structure:**
- QR data now includes: `id`, `po`, `item`, `batch`, `qty`, `pack`, `grn_date`, `exp_date`, `bin` (new)
- Uses existing `line_selection.bin_location` field (no new database columns required)

**Updated Files:**
- `routes.py`: 8 QR data generation points updated to include `bin` field
- `view_batch.html`: QR label display shows "Bin: [location]" row
- `step3_detail.html`: QR label display shows "Bin: [location]" row

### SAP B1 Transfer Request Persistent Storage (Added Nov 27, 2025)
The Inventory Transfer module now permanently stores SAP B1 Transfer Request data in the local database for later posting back to SAP.

**Problem Solved:**
- Previously, the module fetched SAP data on every page load, which was slow and unreliable if SAP was unavailable
- Now SAP data is stored once when the transfer is created and used for all subsequent operations

**Database Changes:**
- `inventory_transfers` table - Added SAP header fields:
  - `sap_doc_entry` (INT) - SAP DocEntry identifier
  - `sap_doc_num` (INT) - SAP document number
  - `bpl_id` (INT) - Business Place ID
  - `bpl_name` (VARCHAR 100) - Business Place name
  - `sap_document_status` (VARCHAR 20) - Document status (bost_Open/bost_Close)
  - `doc_date` (DATETIME) - Document date from SAP
  - `due_date` (DATETIME) - Due date from SAP
  - `sap_raw_json` (LONGTEXT) - Complete SAP response JSON for reference

- `inventory_transfer_items` table - Added SAP line fields:
  - `sap_line_num` (INT) - SAP line number
  - `sap_doc_entry` (INT) - SAP document entry
  - `line_status` (VARCHAR 20) - Line status
  - `from_warehouse_code` (VARCHAR 20) - Source warehouse
  - `to_warehouse_code` (VARCHAR 20) - Destination warehouse

- NEW `inventory_transfer_request_lines` table - Stores SAP StockTransferLines exactly as received:
  - Foreign key to `inventory_transfers`
  - `line_num`, `sap_doc_entry`, `item_code`, `item_description`
  - `quantity`, `warehouse_code`, `from_warehouse_code`
  - `remaining_open_quantity`, `line_status`, `uom_code`
  - WMS tracking: `transferred_quantity`, `wms_remaining_quantity`

**Key Features:**
- SAP data stored on transfer creation (no duplicate API calls)
- Detail view uses stored database data first, falls back to SAP only if no records exist
- Complete SAP JSON preserved for future SAP B1 posting operations
- Line-level tracking of remaining quantities for accurate transfer management

**Migration Files:**
- PostgreSQL: Applied to development database via SQLAlchemy
- MySQL: Updated in `mysql_consolidated_migration.py`

### SO Against Invoice Module (Fixed Nov 26, 2025)
The SO Against Invoice module allows creating invoices against existing Sales Orders with SAP B1 integration.

**Blueprint Registration Fix:**
- Blueprint `so_invoice_bp` registered in `app.py`
- Models imported for auto table creation: `from modules.so_against_invoice import models as so_invoice_models`
- Template folder added to Jinja loader search path
- Templates moved to proper subfolder structure: `templates/so_against_invoice/`
- Routes updated to use correct template paths (e.g., `'so_against_invoice/index.html'`)
- URL prefix: `/so-against-invoice`

**Workflow:**
1. Create new SO Against Invoice document
2. Select SO Series from dropdown
3. Validate SO Number against SAP B1
4. Fetch SO details and open lines
5. Validate items (serial/batch/quantity)
6. Post invoice to SAP B1

**Database Tables:**
- `so_invoice_documents` - Document headers
- `so_invoice_items` - Line items
- `so_invoice_serials` - Serial numbers for items
- `so_series_cache` - SO series cache for faster lookup

**Migration Files:**
- MySQL: `migrations/mysql/changes/2025-11-26_so_against_invoice_module.sql`

## External Dependencies
*   **SAP B1 Service Layer API**: For all core inventory and document management functionalities (GRPO, pick lists, inventory transfers, serial numbers, business partners, inventory counts).
*   **PostgreSQL**: Primary relational database for production environments.
*   **SQLite**: Local relational database for development and initial setup.
*   **Gunicorn**: WSGI HTTP server for deploying the Flask application in production.
*   **Flask-Login**: Library for managing user sessions and authentication.