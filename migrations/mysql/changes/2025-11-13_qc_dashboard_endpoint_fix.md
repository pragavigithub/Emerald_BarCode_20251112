# QC Dashboard Endpoint Fix

**Date**: 2025-11-13  
**Type**: Bug Fix  
**Component**: QC Dashboard Template  

## Issue
The QC Dashboard was throwing a `BuildError` when trying to render the Direct Inventory Transfer review link.

### Error Details
```
werkzeug.routing.exceptions.BuildError: Could not build url for endpoint 'direct_inventory_transfer.transfer_detail' with values ['transfer_id']. Did you mean 'direct_inventory_transfer.detail' instead?
```

## Root Cause
The `qc_dashboard.html` template was using an incorrect endpoint name `direct_inventory_transfer.transfer_detail` when the actual endpoint defined in the blueprint is `direct_inventory_transfer.detail`.

## Changes Made

### File: `templates/qc_dashboard.html`
- **Line 268**: Changed URL endpoint from `direct_inventory_transfer.transfer_detail` to `direct_inventory_transfer.detail`

**Before:**
```html
<a href="{{ url_for('direct_inventory_transfer.transfer_detail', transfer_id=transfer.id) }}" class="btn btn-sm btn-outline-primary">
```

**After:**
```html
<a href="{{ url_for('direct_inventory_transfer.detail', transfer_id=transfer.id) }}" class="btn btn-sm btn-outline-primary">
```

## Impact
- Fixes the 500 error on the QC Dashboard page
- Allows QC approvers to properly review Direct Inventory Transfer requests
- No database schema changes required

## Testing
- Application restarted and verified responding with HTTP 200
- Endpoint naming now matches the blueprint definition in `modules/direct_inventory_transfer/routes.py`

## Notes
This was a template-only fix and did not require any database migrations.
