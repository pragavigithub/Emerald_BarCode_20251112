# Warehouse Management System (WMS)

## Overview
A Flask-based Warehouse Management System (WMS) designed to streamline inventory operations by integrating seamlessly with SAP. The system focuses on enhancing efficiency, accuracy, and control over warehouse logistics through functionalities such as barcode scanning, goods receipt, pick list generation, and inventory transfers. It aims to minimize manual errors and maximize throughput for small to medium-sized enterprises by providing real-time data synchronization with SAP.

## User Preferences
*   Keep MySQL migration files updated when database schema changes occur
*   SQL query validation should only run on initial startup, not on every application restart

## Recent Updates

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