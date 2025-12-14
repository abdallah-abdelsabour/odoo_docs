# PACE2 Integration Module - Complete Documentation

**Module**: `pace2_integration`
**Version**: 17.0
**Author**: Youssef Mohamed
**Dependencies**: base, base_setup, product, sale, purchase, stock, external_sales_person, route_master, account_edi, l10n_sa_edi

---

## Table of Contents

1. [Module Overview](#1-module-overview)
2. [Architecture and Data Flow](#2-architecture-and-data-flow)
3. [Configuration Settings](#3-configuration-settings)
4. [Data Models and Records](#4-data-models-and-records)
5. [Cron Jobs (Scheduled Actions)](#5-cron-jobs-scheduled-actions)
6. [Integration Workflow](#6-integration-workflow)
7. [API Endpoints](#7-api-endpoints)
8. [Code Logic Explained](#8-code-logic-explained)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Module Overview

### Purpose
The PACE2 Integration module provides **bi-directional synchronization** between Odoo and the PACE2 ERP system (PepsiCo's distribution management system). It handles:

- Master data synchronization (Customers, Products, Routes)
- Transaction data exchange (Sales Orders, Invoices, Purchase Orders, Returns)
- Financial data (Collections/Payments)
- Stock movements (Van Loads, Transfers, Adjustments)
- ZATCA (Saudi Tax Authority) integration for e-invoicing

### Key Features
- **Automated sync** via scheduled cron jobs
- **Batch processing** with time limits to prevent timeout
- **State tracking** (pending → processed/failed)
- **Error logging** for failed records
- **Two-phase process**: Sync (fetch from PACE2) → Import (create in Odoo)

---

## 2. Architecture and Data Flow

### High-Level Architecture

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   PACE2 ERP     │◄───────►│  Odoo Instance   │◄───────►│  ZATCA System   │
│   (PepsiCo)     │  REST   │  (pace2_integ)   │  EDI    │  (Tax Auth)     │
└─────────────────┘   API   └──────────────────┘         └─────────────────┘
```

### Data Flow Pattern

**Two-Phase Processing:**

1. **SYNC Phase** (Pull from PACE2)
   ```
   PACE2 API → pace.data.master (staging) → state: pending
   PACE2 API → pace.transaction.move (staging) → state: pending
   PACE2 API → pace.collection (staging) → state: pending
   ```

2. **IMPORT Phase** (Process into Odoo)
   ```
   Staging Tables → Validation → Odoo Models → state: processed/failed
   ```

### Module File Structure

```
pace2_integration/
├── __init__.py
├── __manifest__.py
├── data/
│   ├── cron.xml           # 11 scheduled actions
│   └── data.xml           # Server actions
├── models/
│   ├── res_config_settings.py  # Main orchestration logic (3336 lines)
│   ├── pace2_records.py         # Staging models
│   ├── missing_data_log.py      # Error logging
│   ├── moves_sent_to_pace.py    # Outbound tracking
│   ├── account_move.py          # Invoice extensions
│   ├── account_payment.py       # Payment extensions
│   ├── sale_order.py            # Sales order extensions
│   ├── purchase_order.py        # Purchase order extensions
│   ├── stock_picking.py         # Delivery extensions
│   ├── res_partner.py           # Customer extensions
│   ├── product.py               # Product extensions
│   ├── uom_category.py          # Unit of measure
│   └── pace_specific_record.py  # Specific transactions
├── views/
│   ├── res_config_settings_views.xml
│   ├── [various model views].xml
└── security/
    └── ir.model.access.csv
```

---

## 3. Configuration Settings

### File: `res_config_settings.py`

This file extends `res.config.settings` and `res.company` to add PACE2-specific configurations.

### Configuration Fields

#### Company-Level Fields (stored in `res.company`)

| Field Name | Type | Description |
|-----------|------|-------------|
| `username` | Char | PACE2 API username |
| `user_password` | Char | PACE2 API password |
| `distributor_code` | Char | Unique distributor identifier |
| `token` | Char | OAuth access token (auto-generated) |
| `transaction_count_date_from` | Date | Start date for transaction queries |
| `transaction_count_date_to` | Date | End date for transaction queries |
| `transaction_list_date` | Date | Date for transaction list queries |
| `document_no` | Char | Specific document number for queries |
| `transaction_type` | Char | Transaction type filter |
| `collection_no` | Char | Collection reference number |

#### Import State Tracking Fields

These Boolean fields track whether sync/import operations are complete:

**Master Data:**
- `is_finished_customer_sync` / `is_finished_customer_import`
- `is_finished_product_sync` / `is_finished_product_import`
- `is_finished_route_sync` / `is_finished_route_import`

**Inventory Moves:**
- `is_finished_adjustment_sync` / `is_finished_adjustment_import`
- `is_finished_transfer_sync` / `is_finished_transfer_import`
- `is_finished_load_sync` / `is_finished_load_import`

**Transaction Moves:**
- `is_finished_sale_sync` / `is_finished_sale_import`
- `is_finished_sale_return_sync` / `is_finished_sale_return_import`
- `is_finished_debit_sync` / `is_finished_debit_import`
- `is_finished_credit_sync` / `is_finished_credit_import`
- `is_finished_company_invoice_sync` / `is_finished_company_invoice_import`
- `is_finished_company_return_sync` / `is_finished_company_return_import`

**Collections:**
- `is_finished_collection_sync` / `is_finished_collection_import`

### Key Configuration Methods

#### `generate_access_token()`
- **Purpose**: Obtains OAuth token from PACE2 API
- **Endpoint**: `https://ssfldms.pepsico.com/ZATCA_EInvoiceAPI/token`
- **Called by**: Cron job (daily)
- **Updates**: `company.token` field

---

## 4. Data Models and Records

### Staging Models (Temporary Storage)

#### 4.1 `pace.data.master`
Stores master data before import

| Field | Type | Description |
|-------|------|-------------|
| `code` | Char | Unique identifier |
| `data_type` | Selection | '1'=Customer, '2'=Product, '3'=Route |
| `object` | Text | JSON string of source data |
| `info` | Text | Error messages |
| `state` | Selection | pending/processed/failed |

**Creation**: Via `sync_customers()`, `update_products_from_pace2()`, `sync_rout()`

---

#### 4.2 `pace.transaction.move`
Stores business transactions before import

| Field | Type | Description |
|-------|------|-------------|
| `delivery_note` | Char | Delivery reference |
| `transaction_type` | Selection | See types below |
| `object` | Text | JSON transaction data |
| `info` | Text | Processing notes/errors |
| `state` | Selection | pending/processed/failed |
| `delivery_date` | Date | Computed from object |

**Transaction Types:**
- `'1'` = Sale Order & Invoices
- `'2'` = Sales Return
- `'3'` = Debit Note
- `'4'` = Credit Note
- `'7'` = Company Invoice (Purchase)
- `'8'` = Company Return (Purchase Return)

**Creation**: Via `sync_sales_order_and_invoices()`, `sync_sales_return()`, `sync_purchase_order_and_bills()`, `sync_company_return()`

**Methods**:
- `change_to_pending()`: Reset records to pending state

---

#### 4.3 `pace.inventory.move`
Stores inventory movements before import

| Field | Type | Description |
|-------|------|-------------|
| `transaction_number` | Char | Movement reference |
| `transaction_type` | Selection | See types below |
| `object` | Text | JSON movement data |
| `info` | Text | Processing notes |
| `state` | Selection | pending/processed/failed |
| `transaction_date` | Date | Computed from object |

**Transaction Types:**
- `'1'` = Warehouse Adjustment
- `'2'` = Inventory Transfer
- `'3'` = Van Load

**Creation**: Via `sync_adjustments()`, `sync_inventory_transfer()`, `sync_van_load()`

---

#### 4.4 `pace.collection`
Stores payment collections before import

| Field | Type | Description |
|-------|------|-------------|
| `transaction_number` | Char | Collection reference |
| `transaction_type` | Selection | '1'=Collection |
| `object` | Text | JSON collection data |
| `info` | Text | Processing notes |
| `state` | Selection | pending/processed/failed |
| `transaction_date` | Date | Computed from object |

**Creation**: Via `sync_collections()`

---

#### 4.5 `data.missing.log`
Logs errors during import

| Field | Type | Description |
|-------|------|-------------|
| `name` | Char | Error message |
| `object` | Text | Data that failed |

**Creation**: Automatically when import operations fail

---

### Extended Odoo Models

#### 4.6 `res.partner` Extensions
Additional fields for PACE2 customers:

| Field | Added Purpose |
|-------|---------------|
| `partner_code` | PACE2 customer code |
| `cust_type` | Set to 'pace2' for imported customers |

---

#### 4.7 `sale.order` Extensions

| Field | Purpose |
|-------|---------|
| `external_user_id` | Many2one to external.sales.person |
| `is_pace_record` | Boolean flag for PACE2-originated records |

---

#### 4.8 `account.move` Extensions

| Field | Purpose |
|-------|---------|
| `external_user_id` | External sales person reference |
| `is_pace_record` | Boolean flag |
| `pace_ref` | PACE2 document reference |
| `pace_doc_type` | Document type for ZATCA |

---

#### 4.9 `account.payment` Extensions

| Field | Purpose |
|-------|---------|
| `sales_man_code` | Salesman identifier |
| `sales_man_name` | Salesman name |
| `route_id` | Many2one to route.route |
| `pace_ref` | PACE2 reference |
| `is_pace_record` | Boolean flag |

---

#### 4.10 `stock.picking` Extensions

| Field | Purpose |
|-------|---------|
| `delivery_note` | PACE2 delivery note reference |

**Method**: `create_pace_purchase_return()` - Creates vendor credit notes for purchase returns

---

#### 4.11 `stock.picking.type` Extensions

| Field | Purpose |
|-------|---------|
| `is_sales_return` | Boolean to identify return picking types |

---

## 5. Cron Jobs (Scheduled Actions)

All crons are defined in `data/cron.xml` and call methods in `res_config_settings.py`.

### 5.1 Token Management

#### Cron: `auto_generate_access_token`
- **ID**: `pace2_integration.auto_generate_access_token`
- **Active**: YES (enabled by default)
- **Frequency**: Every 1 day
- **Method**: `generate_access_token()`
- **Purpose**: Refreshes OAuth token for API authentication
- **Critical**: Must run before other operations

---

### 5.2 Master Data Synchronization

#### Cron: `auto_pace_sync_and_import`
- **ID**: `pace2_integration.auto_pace_sync_and_import_cron`
- **Active**: NO (manual activation)
- **Frequency**: Every 1 day
- **Method**: `auto_pace_sync_and_import()`
- **Duration**: 25 seconds max
- **Purpose**: Syncs master data and transactions from PACE2

**What it syncs (in order):**
1. Products → `update_products_from_pace2()`
2. Sales Orders/Invoices → `sync_sales_order_and_invoices()`
3. Purchase Orders → `sync_purchase_order_and_bills()`
4. Sales Returns → `sync_sales_return()`
5. Company Returns → `sync_company_return()`
6. Collections → `sync_collections()`

**State Management**: Uses `ir.config_parameter` to track completion flags

---

### 5.3 Data Import Processing

#### Cron: `auto_pace_import`
- **ID**: `pace2_integration.auto_pace_import_cron`
- **Active**: NO (manual activation)
- **Frequency**: Every 1 day
- **Method**: `auto_pace_import()`
- **Duration**: 90 seconds max
- **Purpose**: Processes staged records into Odoo

**What it imports (in order):**
1. Products → `import_products()`
2. Sale Orders (type='1') → `import_sales_order_and_invoices()`
3. Purchase Orders (type='7') → `import_purchase_order_and_bills()`
4. Sales Returns (type='2') → `import_sales_return_new()`
5. Company Returns (type='8') → `import_company_return()`

---

### 5.4 Collections Processing

#### Cron: `import_pace_collection_cron`
- **ID**: `pace2_integration.import_pace_collection_cron`
- **Active**: YES
- **Frequency**: Every 5 minutes
- **Method**: `cron_import_pace_collection()`
- **Duration**: 90 seconds max
- **Purpose**: Imports payment collections continuously

**Logic**: Processes batches of 30 collections until none pending or time expires

---

#### Cron: `set_journal_id_collection_cron`
- **ID**: `pace2_integration.set_journal_id_collection_cron`
- **Active**: YES
- **Frequency**: Every 2 minutes
- **Method**: `cron_set_collection_journal_id()`
- **Purpose**: Updates journal_id on posted payments based on collection data

**Logic**:
- Finds payments with `journal_id=False`
- Maps to cash/bank journal via route and collection data
- Batch processes 1000-15000 records

---

### 5.5 Delivery Validation

#### Cron: `create_picking_validation_cron`
- **ID**: `pace2_integration.create_picking_validation_cron`
- **Active**: YES
- **Frequency**: Every 2 minutes
- **Method**: `cron_validate_pending_picking()`
- **Duration**: 40 seconds max
- **Purpose**: Auto-validates stock pickings and creates bills

**Processes**:
1. Purchase Order pickings (type='7')
2. Purchase Return pickings (type='8')

**Logic**:
- Validates all pickings for a PO
- Creates bills when all quantities match
- Handles purchase returns by creating credit notes

---

#### Cron: `create_sales_return_failure_cron`
- **ID**: `pace2_integration.create_sales_return_failure_cron`
- **Active**: YES
- **Frequency**: Every 5 minutes
- **Method**: `import_sales_return_new()`
- **Purpose**: Processes sales returns that require special handling

---

#### Cron: `validate_sales_return_failure_cron`
- **ID**: `pace2_integration.validate_sales_return_failure_cron`
- **Active**: YES
- **Frequency**: Every 5 minutes
- **Method**: `cron_validate_pending_picking_sales_return()`
- **Purpose**: Auto-validates sales return pickings and creates credit notes

**Logic**:
- Finds assigned pickings with `is_sales_return=True`
- Sets quantities and validates
- Creates credit notes via `_create_credit_notes_sales_failure()`

---

### 5.6 Purchase Order Processing

#### Cron: `create_pace_purchase_bill_cron`
- **ID**: `pace2_integration.create_pace_purchase_bill_cron`
- **Active**: YES
- **Frequency**: Every 5 hours
- **Method**: `cron_create_pace_po_bills()`
- **Purpose**: Creates bills for completed purchase orders

**Logic**:
- Finds processed PO transactions without bills
- Verifies all pickings are done
- Validates ordered quantity = received quantity
- Creates vendor bill

---

### 5.7 ZATCA Integration

#### Cron: `send_zatca_response_to_pace2_cron`
- **ID**: `pace2_integration.send_zatca_response_to_pace2_cron`
- **Active**: NO
- **Frequency**: Every 1 day
- **Method**: `send_zatca_response_to_pace2()`
- **Purpose**: Sends e-invoice data back to PACE2

**Syncs**: Invoice status, QR codes, approval dates from ZATCA

---

### 5.8 Special Processing

#### Cron: `custom_demo_sync_import_cron`
- **ID**: `pace2_integration.custom_demo_sync_import_cron`
- **Active**: NO
- **Frequency**: Every 1 day
- **Method**: `custom_demo_sync_import()`
- **Purpose**: Demo/testing synchronization

---

#### Cron: `sync_sales_order_and_invoices_cron`
- **ID**: `pace2_integration.sync_sales_order_and_invoices_cron`
- **Active**: NO
- **Frequency**: Every 1 day
- **Method**: `sync_sales_order_and_invoices()`
- **Purpose**: Manual trigger for sales sync

---

#### Cron: `sync_purchase_order_and_bills_cron`
- **ID**: `pace2_integration.sync_purchase_order_and_bills_cron`
- **Active**: NO
- **Frequency**: Every 1 day
- **Method**: `sync_purchase_order_and_bills()`
- **Purpose**: Manual trigger for purchase sync

---

## 6. Integration Workflow

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│              PACE2 INTEGRATION WORKFLOW                      │
└─────────────────────────────────────────────────────────────┘

PHASE 1: AUTHENTICATION
========================
[Daily: 00:00]
auto_generate_access_token
    ├─→ POST: /ZATCA_EInvoiceAPI/token
    └─→ Update company.token

PHASE 2: MASTER DATA SYNC
==========================
[On-Demand or Daily]
auto_pace_sync_and_import (25s loop)
    ├─→ sync_customers() → pace.data.master (type='1')
    ├─→ update_products_from_pace2() → pace.data.master (type='2')
    └─→ sync_rout() → pace.data.master (type='3')

PHASE 3: TRANSACTION SYNC
==========================
[Continuous: Every 1-5 minutes]
auto_pace_sync_and_import (25s loop)
    ├─→ sync_sales_order_and_invoices()
    │       └─→ pace.transaction.move (type='1')
    ├─→ sync_purchase_order_and_bills()
    │       └─→ pace.transaction.move (type='7')
    ├─→ sync_sales_return()
    │       └─→ pace.transaction.move (type='2')
    ├─→ sync_company_return()
    │       └─→ pace.transaction.move (type='8')
    └─→ sync_collections()
            └─→ pace.collection (type='1')

PHASE 4: DATA IMPORT
====================
[Continuous: Every 90s]
auto_pace_import (90s loop)
    ├─→ import_products()
    │       └─→ product.product
    ├─→ import_sales_order_and_invoices()
    │       ├─→ sale.order
    │       └─→ account.move (customer invoices)
    ├─→ import_purchase_order_and_bills()
    │       └─→ purchase.order
    ├─→ import_sales_return_new()
    │       ├─→ stock.picking (returns)
    │       └─→ account.move (credit notes)
    └─→ import_company_return()
            └─→ stock.picking (vendor returns)

PHASE 5: COLLECTION PROCESSING
===============================
[Every 5 minutes]
cron_import_pace_collection (90s loop)
    └─→ import_collections()
            └─→ account.payment

[Every 2 minutes]
cron_set_collection_journal_id
    └─→ Update payment journals (cash/bank)

PHASE 6: DELIVERY VALIDATION
=============================
[Every 2 minutes]
cron_validate_pending_picking (40s loop)
    ├─→ Validate PO pickings (type='7')
    │       └─→ Create vendor bills
    └─→ Validate PO return pickings (type='8')
            └─→ Create vendor credit notes

[Every 5 minutes]
cron_validate_pending_picking_sales_return
    └─→ Validate sales return pickings
            └─→ Create customer credit notes

PHASE 7: BILL CREATION
======================
[Every 5 hours]
cron_create_pace_po_bills
    └─→ Create bills for completed POs

PHASE 8: ZATCA SYNC (Optional)
==============================
[Daily or On-Demand]
send_zatca_response_to_pace2
    └─→ Send invoice status to PACE2
```

---

### Workflow Start-to-End Example: Sales Order

```
1. PACE2 creates sales order
2. [Cron] auto_pace_sync_and_import
   └─→ sync_sales_order_and_invoices()
       └─→ Creates pace.transaction.move (state=pending, type='1')
3. [Cron] auto_pace_import
   └─→ import_sales_order_and_invoices()
       ├─→ Validates customer exists (res.partner)
       ├─→ Validates products exist
       ├─→ Creates sale.order with pace2 flag
       ├─→ Confirms order
       ├─→ Validates delivery picking
       ├─→ Creates and posts account.move (invoice)
       └─→ Updates pace.transaction.move (state=processed)
4. [Optional] ZATCA generates QR code for invoice
5. [Cron] send_zatca_response_to_pace2
   └─→ Sends invoice status back to PACE2
```

---

## 7. API Endpoints

### Base URL
- **Production**: `https://ssfldms.pepsico.com/ZATCA_EInvoiceAPI`
- **Test**: `https://ksadmstestnew.pepsico.com/ZATCA_EInvoiceAPI` (commented out)

### Endpoint List

| Method | Endpoint | Purpose | Called By |
|--------|----------|---------|-----------|
| POST | `/token` | Get OAuth token | `generate_access_token()` |
| POST | `/ZATCA/api/Customerdata` | Fetch customers | `sync_customers()` |
| POST | `/ZATCA/api/MstRouteData` | Fetch routes | `sync_rout()` |
| POST | `/ZATCA/api/ProductData` | Fetch products | `update_products_from_pace2()` |
| POST | `/ZATCA/api/TxnInventoryData` | Fetch inventory moves | `sync_adjustments()`, `sync_inventory_transfer()`, `sync_van_load()` |
| POST | `/ZATCA/api/TransactionData` | Fetch sales transactions | `sync_sales_order_and_invoices()`, `sync_sales_return()` |
| POST | `/ZATCA/api/CompanyInvoice` | Fetch purchase invoices | `sync_purchase_order_and_bills()` |
| POST | `/ZATCA/api/CompanyReturns` | Fetch purchase returns | `sync_company_return()` |
| POST | `/ZATCA/api/TxnCollectiondata` | Fetch collections | `sync_collections()` |
| POST | `/ZATCA/api/TransactionCount` | Get transaction counts | `sync_transactions_count()` |
| POST | `/ZATCA/api/TransactionList` | Get transaction list | `sync_transactions_list()` |
| POST | `/ZATCA/api/TransactionDataSpecific` | Fetch specific transactions | `sync_specific_transactions()` |
| POST | `/ZATCA/api/TxnCollectiondataSpecific` | Fetch specific collections | `sync_specific_collections()` |
| POST | `/ZATCA/api/SyncZATCA` | Send ZATCA response | `sync_zatca_response()` |

### Authentication
All API calls (except `/token`) require:
```python
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {token}"
}
```

---

## 8. Code Logic Explained

### 8.1 Two-Phase Architecture

**Why Two Phases?**
- **Decoupling**: Separates API calls (unstable) from data processing (stable)
- **Batch Processing**: Allows processing records at different speeds
- **Error Recovery**: Failed imports don't affect API sync
- **State Tracking**: Can track what's synced vs. what's imported

### 8.2 State Machine

Every record follows this state machine:
```
pending → processed  (success)
pending → failed     (error logged)
failed  → pending    (manual retry)
```

### 8.3 Time-Limited Processing

All cron methods use time limits to prevent timeouts:
```python
time_limit = time.time() + 90  # 90 seconds
while time.time() < time_limit and has_pending_records:
    process_batch()
```

This ensures:
- No memory errors from infinite loops
- Predictable execution times
- Gradual processing of large datasets

### 8.4 Batch Size Strategy

Different operations use different batch sizes:
- Collections: 30 per batch (frequent, small)
- Products: 300 per batch (infrequent, large)
- Transactions: 50-100 per batch (medium)

### 8.5 Error Handling

Three-level error handling:
1. **Try-Catch in Cron**: Prevents one error from stopping entire cron
2. **Record-Level State**: Marks individual records as failed
3. **Missing Data Log**: Stores error details in `data.missing.log`

Example:
```python
try:
    Partner.create(vals)
    record.write({'state': 'processed'})
except Exception as e:
    record.write({'state': 'failed', 'info': e})
    self.env['data.missing.log'].create({
        'name': str(e),
        'object': vals,
    })
```

### 8.6 Idempotency

Import methods are designed to be idempotent:
- Check for existing records before create
- Use `write()` if record exists, `create()` if not
- Safe to re-run failed batches

Example:
```python
existing_partner = Partner.search([('partner_code', '=', customer['Cust_Code'])], limit=1)
if existing_partner:
    existing_partner.write(vals)
else:
    Partner.create(vals)
```

### 8.7 Relationship Management

**External Sales Person Creation:**
```python
def _create_or_get_external_sales_person(self, code, name):
    """Ensures external sales person exists before creating order"""
    external_sales_person = self.env['external.sales.person'].search([('code', '=', code)], limit=1)
    if not external_sales_person:
        external_sales_person = self.env['external.sales.person'].create({
            'name': name,
            'code': code
        })
    return external_sales_person
```

### 8.8 Date Handling

PACE2 dates come in ISO format and are converted:
```python
delivery_date_obj = datetime.strptime(
    purchase_orders_json.get('DeliveryDate'),
    '%Y-%m-%dT%H:%M:%S'
)
delivery_date_odoo_format = delivery_date_obj.strftime('%Y-%m-%d %H:%M:%S')
```

### 8.9 Unit of Measure Mapping

Complex UOM logic for products:
```python
uom_code = line.get('UOMCode', False)
if uom_code and uom_code == 'UN':
    quantities_per_uom = 1
    target_uom_id = # find 'unit' UOM
else:
    quantities_per_uom = int(float(product.uom)) if product.uom else 1
    target_uom_id = # find 'karton' UOM
```

### 8.10 ZATCA Integration Flow

```python
1. Invoice posted in Odoo
2. ZATCA generates QR code and approval
3. sync_zatca_response() sends to PACE2:
   - Invoice status
   - QR code
   - Document ID
   - Approval date
```

### 8.11 Purchase Order Bill Creation Logic

The `cron_create_pace_po_bills()` method has strict validation:
```python
1. Find processed PO transactions without bills
2. Check all pickings are done
3. Verify ordered qty == received qty for ALL products
4. Only create bill if quantities match exactly
```

This prevents partial billing and ensures data accuracy.

### 8.12 Collection Journal Assignment

Smart journal assignment based on payment method:
```python
if collection.get('Total Cash') > 0:
    journal = route_master.cash_journal_id
else:
    journal = route_master.bank_journal_id
```

Payments are set to draft, updated, then posted again.

---

## 9. Troubleshooting

### Common Issues and Solutions

#### Issue 1: Token Expired
**Symptom**: API calls return 401 Unauthorized
**Solution**:
- Check if `auto_generate_access_token` cron is active
- Manually run Settings → PACE2 → Generate Token

#### Issue 2: Records Stuck in Pending
**Symptom**: Staging tables have many pending records
**Solution**:
- Check if import crons are active
- Review `data.missing.log` for errors
- Manually run import methods from Settings

#### Issue 3: Duplicate Records
**Symptom**: Same customer/product created twice
**Solution**:
- Check `partner_code` / `default_code` uniqueness
- Records should use `search()` + `write()` or `create()`

#### Issue 4: Missing Products
**Symptom**: Sales order import fails due to missing products
**Solution**:
- Ensure product sync runs before transaction sync
- Check `pace.data.master` for pending products (type='2')

#### Issue 5: Picking Not Validating
**Symptom**: Deliveries stuck in assigned state
**Solution**:
- Check `cron_validate_pending_picking` is active
- Verify quantities are set correctly
- Check for backorder issues

#### Issue 6: Bills Not Created
**Symptom**: Purchase orders have done pickings but no bills
**Solution**:
- Check `cron_create_pace_po_bills` is active
- Verify ordered qty = received qty
- Look at transaction info field for "Without Transfer Validation"

#### Issue 7: Collection Journal Missing
**Symptom**: Payments have no journal
**Solution**:
- Check `cron_set_collection_journal_id` is active
- Verify route has cash_journal_id and bank_journal_id set
- Ensure collection data has route reference

### Debugging Tips

1. **Check Cron Logs:**
   ```python
   logger.error('---------------->po_transactions %s', po_transactions)
   ```
   Look in Odoo logs for these debug messages

2. **Monitor State Fields:**
   - Go to Settings → Technical → Parameters
   - Look for `pace2_integration.is_finished_*` parameters

3. **Review Staging Tables:**
   - Check record counts in pace.transaction.move
   - Filter by state and transaction_type
   - Review 'object' field JSON for data issues

4. **Use Action Menus:**
   - pace.transaction.move has "Change To Pending" action
   - Allows reprocessing failed records

5. **Check Dependencies:**
   - Ensure `external_sales_person` module is installed
   - Verify `route_master` data is populated

### Manual Operations

To manually trigger operations from Settings → General Settings → PACE2 Integration:

1. Generate Token
2. Sync Customers / Products / Routes
3. Import Customers / Products / Routes
4. Sync Transactions
5. Import Transactions

---

## Summary

### Integration Flow Order

**Must Start First:**
1. Token Generation (runs daily, required for all API calls)

**Then Master Data (can run in parallel):**
2. Sync Customers → Import Customers
3. Sync Products → Import Products
4. Sync Routes → Import Routes

**Then Transactions (sequential processing):**
5. Sync Sales/Purchase/Returns → staged in pace.transaction.move
6. Import Sales Orders → creates SO + invoices
7. Import Purchase Orders → creates PO (pickings validated by cron)
8. Import Sales Returns → creates return pickings (validated by cron → credit notes)
9. Import Purchase Returns → creates return pickings (validated by cron → credit notes)

**Continuous Processing:**
10. Collections (every 5 min) → account.payment
11. Journal Assignment (every 2 min) → updates payment journals
12. Picking Validation (every 2 min) → validates transfers, creates bills/credit notes
13. Bill Creation (every 5 hours) → creates vendor bills for completed POs

**Optional:**
14. ZATCA Sync (sends e-invoice data back to PACE2)

---

## Key Takeaways for Developers

1. **Always check state**: Before processing, ensure dependencies are in 'processed' state
2. **Use time limits**: All batch operations should respect time limits
3. **Log errors**: Use `data.missing.log` for import failures
4. **Test idempotency**: Import methods should be safe to re-run
5. **Handle dates carefully**: Convert ISO dates to Odoo format
6. **Validate relationships**: Ensure partners, products, routes exist before creating transactions
7. **Monitor crons**: Keep critical crons active (token, collections, validations)
8. **Review JSON data**: When debugging, check the 'object' field in staging tables

---

## Key Takeaways for Implementers

1. **Initial Setup:**
   - Configure username, password, distributor_code in Settings
   - Generate token manually first time
   - Activate token generation cron

2. **Master Data Import:**
   - Run sync → import for customers, products, routes
   - Verify data in Odoo before proceeding

3. **Transaction Processing:**
   - Activate auto_pace_sync_and_import cron
   - Activate auto_pace_import cron
   - Monitor first few imports for errors

4. **Continuous Operations:**
   - Keep collection crons active (every 5 min)
   - Keep validation crons active (every 2 min)
   - Keep bill creation cron active (every 5 hours)

5. **Monitoring:**
   - Check `data.missing.log` daily
   - Review staging table counts (pending should decrease)
   - Verify invoices/bills are created correctly

6. **Troubleshooting:**
   - Start with token issues first
   - Then check master data completeness
   - Finally debug transaction imports

---

**Document Version**: 1.0
**Last Updated**: 2025-12-10
**Contact**: For technical questions, review the code in `res_config_settings.py`
