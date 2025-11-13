# Warehouse Management System (WMS)

## Overview
A Flask-based Warehouse Management System (WMS) designed to streamline inventory operations by integrating seamlessly with SAP for functionalities such as barcode scanning, goods receipt, pick list generation, and inventory transfers. The system aims to enhance efficiency, accuracy, and control over warehouse logistics, ultimately minimizing manual errors and maximizing throughput for small to medium-sized enterprises.

## User Preferences
*   Keep MySQL migration files updated when database schema changes occur
*   SQL query validation should only run on initial startup, not on every application restart

## Recent Changes

### November 13, 2025 - Inventory Transfer QR Scanning Enhancement (Complete)
**Feature:** Production-ready database-backed QR code scanning for Inventory Transfer module
**Implementation:**
- Created `TransferScanState` model for persistent pack tracking (replaces Flask session to avoid 4KB cookie limit)
- Enhanced `api_scan_qr_label` endpoint to parse JSON QR labels with pack information (id, po, item, batch, qty, pack, grn_date, exp_date)
- Implemented duplicate pack prevention using database unique constraints
- Added quantity accumulation and overflow protection
- Built frontend UI with real-time scan progress, accumulated quantities, and remaining quantity display
- Added item code validation to prevent scanning QR codes for wrong items
- Made Available Quantity field readonly to preserve scan integrity
- Implemented modal reset functionality to clear scan state when closing

**Key Features:**
- ✅ Parses JSON QR format: `{"id":"GRN/xxx","po":"xxx","item":"xxx","batch":"xxx","qty":10,"pack":"1 of 5","grn_date":"xxx","exp_date":"xxx"}`
- ✅ Batch Number field populated with unique batches including quantity and expiry date
- ✅ Available Quantity field populated with accumulated quantities (readonly)
- ✅ Tracks each pack uniquely (prevents scanning "1 of 5" twice via database constraint)
- ✅ Accumulates quantities from multiple scans until equals requested quantity
- ✅ Validates against requested quantities with overflow protection
- ✅ Item code mismatch validation prevents scanning wrong QR codes
- ✅ Manages batch numbers for SAP B1 integration
- ✅ Database-backed state supports unlimited pack scans
- ✅ Modal auto-resets scan state when closed

**Files Modified:**
- `models.py`: Added TransferScanState model (lines 190-216)
- `modules/inventory_transfer/routes.py`: Enhanced api_scan_qr_label with item validation and api_reset_scan_state endpoints
- `templates/inventory_transfer_detail.html`: Complete frontend implementation with scan tracking, validation, and modal management

### November 12, 2025 - Multi GRN QR Labels Fix
**Issue:** Print Batch Labels button in Multi GRN module was not displaying QR labels
**Root Cause:** Users were attempting to print labels before adding item details (warehouse, bin location, batch/serial info)
**Solution:** 
- Added comprehensive logging to track label generation requests and identify missing data
- Implemented explicit error handling that returns clear message: "No batch details found for this item. Please add item details first before printing labels."
- Enhanced user workflow guidance in documentation

**Correct Workflow:**
1. Click "Add Item" button in Step 3: Line Item Details
2. Enter warehouse, bin location, and number of packs
3. Click "Generate QR Labels" and save
4. Then "Print Batch Labels" will work correctly

**Files Modified:**
- `modules/multi_grn_creation/routes.py`: Enhanced logging and validation in generate_barcode_labels_multi_grn()
- `MULTI_GRN_QR_LABELS_FIX.md`: Detailed documentation of the fix

## System Architecture
The system is built on a Flask web application backend, utilizing Jinja2 for server-side rendering. A core architectural decision is the deep integration with the SAP B1 Service Layer API for all critical warehouse operations, ensuring data consistency and real-time updates. PostgreSQL is the primary database target for cloud deployments, with SQLite serving as a fallback. User authentication uses Flask-Login with robust role-based access control. The application is designed for production deployment using Gunicorn with autoscale capabilities.

**UI/UX Decisions:**
*   Intuitive workflows for managing inventory, including serial number transfers and real-time validation against SAP B1.
*   Dynamic dropdowns for bin locations, populated from SAP B1.
*   Enhanced GRPO workflow with read-only warehouse fields automatically populated from Purchase Order data.
*   Comprehensive pagination, filtering, and search functionalities across key modules (GRPO Details, Sales Order Against Delivery Details, Inventory Counting History, Inventory Transfer).

**Technical Implementations:**
*   **SAP B1 Integration:** Utilizes a dedicated `SAPMultiGRNService` class for secure and robust communication with the SAP B1 Service Layer, including SSL/TLS verification and optimized OData filtering.
*   **Modular Design:** New features are implemented as modular blueprints with their own templates and services.
*   **Frontend:** Jinja2 templating with JavaScript libraries like Select2 for enhanced UI components.
*   **Error Handling:** Comprehensive validation and error logging for API communications and user inputs.
*   **Optimized SAP SQL Query Validation:** SQL query validation runs only on initial startup using a flag-based system to prevent repeated attempts.
*   **Database Migrations:** A comprehensive MySQL migration tracking system is in place for schema changes, complementing the primary PostgreSQL strategy.

**Feature Specifications:**
*   **User Management:** Comprehensive authentication, role-based access, and self-service profile management.
*   **GRPO Management:** Standard Goods Receipt PO processing, intelligent batch/serial field management, and a multi-GRN module for batch creation from multiple Purchase Orders via a 5-step workflow with SAP B1 integration and QR label generation.
*   **Inventory Transfer:** Enhanced module for creating inventory transfer requests with document series selection, SAP B1 validation, and QR label scanning.
*   **Direct Inventory Transfer:** Barcode-based inventory transfer module with automatic serial/batch detection, real-time SAP B1 validation, warehouse and bin selection, QC approval workflow, and direct posting to SAP B1 as StockTransfers. Includes QR code scanning with automatic item validation and camera-based scanning functionality using the integrated barcode scanner library.
*   **Sales Order Against Delivery:** Module for creating Delivery Notes against Sales Orders with SAP B1 integration, including SO series selection, cascading dropdown for open SO document numbers, item picking with batch/serial validation, and individual QR code label generation.
*   **Pick List Management:** Generation and processing of pick lists.
*   **Barcode Scanning:** Integrated camera-based scanning for various modules (GRPO, Bin Scanning, Pick List, Inventory Transfer, Barcode Reprint).
*   **Inventory Counting:** SAP B1 integrated inventory counting with local database storage for tracking, audit trails, user tracking, and timestamps. Includes a comprehensive history view.
*   **Branch Management:** Functionality for managing different warehouse branches.
*   **Quality Control Dashboard:** Provides oversight for quality processes.

## External Dependencies
*   **SAP B1 Service Layer API**: For all core inventory and document management functionalities (GRPO, pick lists, inventory transfers, serial numbers, business partners, inventory counts).
*   **PostgreSQL**: Primary relational database for production environments.
*   **SQLite**: Local relational database for development and initial setup.
*   **Gunicorn**: WSGI HTTP server for deploying the Flask application in production.
*   **Flask-Login**: Library for managing user sessions and authentication.