---
description: 'Dataverse Python SDK: core concepts, modules, API reference and best practices'
applyTo: '**'
---

# Dataverse SDK for Python — Getting Started

- Install the Dataverse Python SDK and prerequisites.
- Configure environment variables for Dataverse tenant, client ID, secret, and resource URL.
- Use the SDK to authenticate via OAuth and perform CRUD operations.

## Setup
- Python 3.10+
- Recommended: virtual environment

## Install
```bash
pip install dataverse-sdk
```

## Auth Basics
- Use OAuth with Azure AD app registration.
- Store secrets in `.env` and load via `python-dotenv`.

## Common Tasks
- Query tables
- Create/update rows
- Batch operations
- Handle pagination and throttling

## Tips
- Reuse clients; avoid frequent re-auth.
- Add retries for transient failures.
- Log requests for troubleshooting.

---

# Dataverse SDK for Python — Official Quickstart

This instruction summarizes Microsoft Learn guidance for the Dataverse SDK for Python (preview) and provides copyable snippets.

## Prerequisites
- Dataverse environment with read/write
- Python 3.10+
- Network access to PyPI

## Install
```bash
pip install PowerPlatform-Dataverse-Client
```

## Connect
```python
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.config import DataverseConfig

cfg = DataverseConfig()  # defaults to language_code=1033
client = DataverseClient(
    base_url="https://<myorg>.crm.dynamics.com",
    credential=InteractiveBrowserCredential(),
    config=cfg,
)
```
- Optional HTTP settings: `cfg.http_retries`, `cfg.http_backoff`, `cfg.http_timeout`.

## CRUD Examples
```python
# Create returns list[str] of GUIDs
account_id = client.create("account", {"name": "Acme, Inc.", "telephone1": "555-0100"})[0]

# Retrieve single
account = client.get("account", account_id)

# Update (returns None)
client.update("account", account_id, {"telephone1": "555-0199"})

# Delete
client.delete("account", account_id)
```

## Bulk Operations
```python
# Broadcast patch to many IDs
ids = client.create("account", [{"name": "Contoso"}, {"name": "Fabrikam"}])
client.update("account", ids, {"telephone1": "555-0200"})

# 1:1 list of patches
client.update("account", ids, [{"telephone1": "555-1200"}, {"telephone1": "555-1300"}])

# Bulk create
payloads = [{"name": "Contoso"}, {"name": "Fabrikam"}, {"name": "Northwind"}]
ids = client.create("account", payloads)
```

## File Upload
```python
client.upload_file('account', record_id, 'sample_filecolumn', 'test.pdf')
client.upload_file('account', record_id, 'sample_filecolumn', 'test.pdf', mode='chunk', if_none_match=True)
```

## Paging Retrieve Multiple
```python
pages = client.get(
    "account",
    select=["accountid", "name", "createdon"],
    orderby=["name asc"],
    top=10,
    page_size=3,
)
for page in pages:
    print(len(page), page[:2])
```

## Table Metadata Quickstart
```python
info = client.create_table("SampleItem", {
    "code": "string",
    "count": "int",
    "amount": "decimal",
    "when": "datetime",
    "active": "bool",
})
logical = info["entity_logical_name"]
rec_id = client.create(logical, {f"{logical}name": "Sample A"})[0]
client.delete(logical, rec_id)
client.delete_table("SampleItem")
```

## References
- Getting started: https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/get-started
- Working with data: https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/work-data
- SDK source/examples: https://github.com/microsoft/PowerPlatform-DataverseClient-Python

---

# Dataverse SDK for Python — Complete Module Reference

## Package Hierarchy

```
PowerPlatform.Dataverse
├── client
│   └── DataverseClient
├── core
│   ├── config (DataverseConfig)
│   └── errors (DataverseError, ValidationError, MetadataError, HttpError, SQLParseError)
├── data (OData operations, metadata, SQL, file upload)
├── extensions (placeholder for future extensions)
├── models (placeholder for data models and types)
└── utils (placeholder for utilities and adapters)
```

## core.config Module

Manage client connection and behavior settings.

### DataverseConfig Class

Container for language, timeouts, retries. Immutable.

```python
from PowerPlatform.Dataverse.core.config import DataverseConfig

cfg = DataverseConfig(
    language_code=1033,        # Default English (US)
    http_retries=None,         # Reserved for future
    http_backoff=None,         # Reserved for future
    http_timeout=None          # Reserved for future
)

# Or use default static builder
cfg_default = DataverseConfig.from_env()
```

**Key attributes:**
- `language_code: int = 1033` — LCID for localized labels and messages.
- `http_retries: int | None` — (Reserved) Maximum retry attempts for transient errors.
- `http_backoff: float | None` — (Reserved) Backoff multiplier between retries.
- `http_timeout: float | None` — (Reserved) Request timeout in seconds.

## core.errors Module

Structured exception hierarchy for SDK operations.

### DataverseError (Base)

Base exception for SDK errors.

```python
from PowerPlatform.Dataverse.core.errors import DataverseError

try:
    # SDK call
    pass
except DataverseError as e:
    print(f"Code: {e.code}")                # Error category
    print(f"Subcode: {e.subcode}")          # Specific error
    print(f"Message: {e.message}")          # Human-readable
    print(f"Status: {e.status_code}")       # HTTP status (if applicable)
    print(f"Transient: {e.is_transient}")   # Retry-worthy?
    details = e.to_dict()                  # Convert to dict
```

### ValidationError

Validation failures during data operations.

```python
from PowerPlatform.Dataverse.core.errors import ValidationError
```

### MetadataError

Table/column creation, deletion, or inspection failures.

```python
from PowerPlatform.Dataverse.core.errors import MetadataError

try:
    client.create_table("MyTable", {...})
except MetadataError as e:
    print(f"Metadata issue: {e.message}")
```

### HttpError

Web API HTTP request failures (4xx, 5xx, etc.).

```python
from PowerPlatform.Dataverse.core.errors import HttpError

try:
    client.get("account", record_id)
except HttpError as e:
    print(f"HTTP {e.status_code}: {e.message}")
    print(f"Service error code: {e.service_error_code}")
    print(f"Correlation ID: {e.correlation_id}")
    print(f"Request ID: {e.request_id}")
    print(f"Retry-After: {e.retry_after} seconds")
    print(f"Transient (retry?): {e.is_transient}")  # 429, 503, 504
```

### SQLParseError

SQL query syntax errors when using `query_sql()`.

```python
from PowerPlatform.Dataverse.core.errors import SQLParseError

try:
    client.query_sql("INVALID SQL HERE")
except SQLParseError as e:
    print(f"SQL parse error: {e.message}")
```

## data Package

Low-level OData protocol, metadata, SQL, and file operations (internal delegation).

The `data` package is primarily internal; the high-level `DataverseClient` in the `client` module wraps and exposes:
- CRUD operations via OData
- Metadata management (create/update/delete tables and columns)
- SQL query execution
- File upload handling

Users interact with these via `DataverseClient` methods (e.g., `create()`, `get()`, `update()`, `delete()`, `create_table()`, `query_sql()`, `upload_file()`).

## extensions Package (Placeholder)

Reserved for future extension points (e.g., custom adapters, middleware).

Currently empty; use core and client modules for current functionality.

## models Package (Placeholder)

Reserved for future data model definitions and type definitions.

Currently empty. Data structures return as `dict` (OData) and are JSON-serializable.

## utils Package (Placeholder)

Reserved for utility adapters and helpers.

Currently empty. Helper functions may be added in future releases.

## client Module

Main user-facing API.

### DataverseClient Class

High-level client for all Dataverse operations.

```python
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.config import DataverseConfig

# Create credential
credential = InteractiveBrowserCredential()

# Optionally configure
cfg = DataverseConfig(language_code=1033)

# Create client
client = DataverseClient(
    base_url="https://org.crm.dynamics.com",
    credential=credential,
    config=cfg  # optional
)
```

#### CRUD Methods

- `create(table_schema_name, records)` → `list[str]` — Create records, return GUIDs.
- `get(table_schema_name, record_id=None, select, filter, orderby, top, expand, page_size)` → Record(s).
- `update(table_schema_name, ids, changes)` → `None` — Update records.
- `delete(table_schema_name, ids, use_bulk_delete=True)` → `str | None` — Delete records.

#### Metadata Methods

- `create_table(table_schema_name, columns, solution_unique_name, primary_column_schema_name)` → Metadata dict.
- `create_columns(table_schema_name, columns)` → `list[str]`.
- `delete_columns(table_schema_name, columns)` → `list[str]`.
- `delete_table(table_schema_name)` → `None`.
- `get_table_info(table_schema_name)` → Metadata dict or `None`.
- `list_tables()` → `list[str]`.

#### SQL & Utilities

- `query_sql(sql)` → `list[dict]` — Execute read-only SQL.
- `upload_file(table_schema_name, record_id, file_name_attribute, path, mode, mime_type, if_none_match)` → `None` — Upload to file column.
- `flush_cache(kind)` → `int` — Clear SDK caches (e.g., `"picklist"`).

## Imports Summary

```python
# Main client
from PowerPlatform.Dataverse.client import DataverseClient

# Configuration
from PowerPlatform.Dataverse.core.config import DataverseConfig

# Errors
from PowerPlatform.Dataverse.core.errors import (
    DataverseError,
    ValidationError,
    MetadataError,
    HttpError,
    SQLParseError,
)
```

## References

- Module docs: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/
- Core: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.core
- Data: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.data
- Extensions: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.extensions
- Models: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.models
- Utils: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.utils
- Client: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.client

---

# Dataverse SDK for Python - Best Practices Guide

## Overview
Production-ready patterns and best practices extracted from Microsoft's official PowerPlatform-DataverseClient-Python repository, examples, and recommended workflows.

## 1. Installation & Environment Setup

### Production Installation
```bash
# Install the published SDK from PyPI
pip install PowerPlatform-Dataverse-Client

# Install Azure Identity for authentication
pip install azure-identity

# Optional: pandas integration for data manipulation
pip install pandas
```

### Development Installation
```bash
# Clone the repository
git clone https://github.com/microsoft/PowerPlatform-DataverseClient-Python.git
cd PowerPlatform-DataverseClient-Python

# Install in editable mode for live development
pip install -e .

# Install development dependencies
pip install pytest pytest-cov black isort mypy ruff
```

### Python Version Support
- **Minimum**: Python 3.10
- **Recommended**: Python 3.11+ for best performance
- **Supported**: Python 3.10, 3.11, 3.12, 3.13, 3.14

### Verify Installation
```python
from PowerPlatform.Dataverse import __version__
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import InteractiveBrowserCredential

print(f"SDK Version: {__version__}")
print("Installation successful!")
```

---

## 2. Authentication Patterns

### Interactive Development (Browser-Based)
```python
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient

credential = InteractiveBrowserCredential()
client = DataverseClient("https://yourorg.crm.dynamics.com", credential)
```

**When to use:** Local development, interactive testing, single-user scenarios.

### Production (Client Secret)
```python
from azure.identity import ClientSecretCredential
from PowerPlatform.Dataverse.client import DataverseClient

credential = ClientSecretCredential(
    tenant_id="your-tenant-id",
    client_id="your-client-id",
    client_secret="your-client-secret"
)
client = DataverseClient("https://yourorg.crm.dynamics.com", credential)
```

**When to use:** Server-side applications, Azure automation, scheduled jobs.

### Certificate-Based Authentication
```python
from azure.identity import ClientCertificateCredential
from PowerPlatform.Dataverse.client import DataverseClient

credential = ClientCertificateCredential(
    tenant_id="your-tenant-id",
    client_id="your-client-id",
    certificate_path="path/to/certificate.pem"
)
client = DataverseClient("https://yourorg.crm.dynamics.com", credential)
```

**When to use:** Highly secure environments, certificate-pinning requirements.

### Azure CLI Authentication
```python
from azure.identity import AzureCliCredential
from PowerPlatform.Dataverse.client import DataverseClient

credential = AzureCliCredential()
client = DataverseClient("https://yourorg.crm.dynamics.com", credential)
```

**When to use:** Local testing with Azure CLI installed, Azure DevOps pipelines.

---

## 3. Singleton Client Pattern

**Best Practice**: Create one `DataverseClient` instance and reuse it throughout your application.

```python
# ❌ ANTI-PATTERN: Creating new clients repeatedly
def fetch_account(account_id):
    credential = InteractiveBrowserCredential()
    client = DataverseClient("https://yourorg.crm.dynamics.com", credential)
    return client.get("account", account_id)

# ✅ PATTERN: Singleton client
class DataverseService:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            credential = InteractiveBrowserCredential()
            cls._instance = DataverseClient(
                "https://yourorg.crm.dynamics.com", 
                credential
            )
        return cls._instance

# Usage
service = DataverseService()
account = service.get("account", account_id)
```

---

## 4. Configuration Optimization

### Connection Settings
```python
from PowerPlatform.Dataverse.core.config import DataverseConfig
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import ClientSecretCredential

config = DataverseConfig(
    language_code=1033,  # English (US)
    # Note: http_retries, http_backoff, http_timeout are reserved for internal use
)

credential = ClientSecretCredential(tenant_id, client_id, client_secret)
client = DataverseClient("https://yourorg.crm.dynamics.com", credential, config)
```

**Key configuration options:**
- `language_code`: Language for API responses (default: 1033 for English)

---

## 5. CRUD Operations Best Practices

### Create Operations

#### Single Record
```python
record_data = {
    "name": "Contoso Ltd",
    "telephone1": "555-0100",
    "creditlimit": 100000.00,
}
created_ids = client.create("account", record_data)
record_id = created_ids[0]
print(f"Created: {record_id}")
```

#### Bulk Create (Automatically Optimized)
```python
# SDK automatically uses CreateMultiple for arrays > 1 record
records = [
    {"name": f"Company {i}", "creditlimit": 50000 + (i * 1000)}
    for i in range(100)
]
created_ids = client.create("account", records)
print(f"Created {len(created_ids)} records")
```

**Performance**: Bulk create is optimized internally; no manual batching required.

### Read Operations

#### Single Record by ID
```python
account = client.get("account", "account-guid-here")
print(account.get("name"))
```

#### Query with Filtering & Selection
```python
# Returns paginated results (generator)
for page in client.get(
    "account",
    filter="creditlimit gt 50000",
    select=["name", "creditlimit", "telephone1"],
    orderby="name",
    top=100
):
    for account in page:
        print(f"{account['name']}: ${account['creditlimit']}")
```

**Key parameters:**
- `filter`: OData filter (must use **lowercase** logical names)
- `select`: Fields to retrieve (improves performance)
- `orderby`: Sort results
- `top`: Max records per page (default: 5000)
- `page_size`: Override page size for pagination

#### SQL Queries (Read-Only)
```python
# SQL queries are read-only; use for complex analytics
results = client.query_sql("""
    SELECT TOP 10 name, creditlimit 
    FROM account 
    WHERE creditlimit > 50000
    ORDER BY name
""")

for row in results:
    print(f"{row['name']}: ${row['creditlimit']}")
```

**Limitations:**
- Read-only (SELECT only, no DML)
- Useful for complex joins and analytics
- May be disabled by org policy

### Update Operations

#### Single Record
```python
client.update("account", "account-guid", {
    "creditlimit": 150000.00,
    "name": "Updated Company Name"
})
```

#### Bulk Update (Broadcast Same Change)
```python
# Update all selected records with same data
account_ids = ["id1", "id2", "id3"]
client.update("account", account_ids, {
    "industrycode": 1,  # Retail
    "accountmanagerid": "manager-guid"
})
```

#### Paired Updates (1:1 Record Updates)
```python
# For different updates per record, send multiple calls
updates = {
    "id1": {"creditlimit": 100000},
    "id2": {"creditlimit": 200000},
    "id3": {"creditlimit": 300000},
}
for record_id, data in updates.items():
    client.update("account", record_id, data)
```

### Delete Operations

#### Single Record
```python
client.delete("account", "account-guid")
```

#### Bulk Delete (Optimized)
```python
# SDK automatically uses BulkDelete for large lists
record_ids = ["id1", "id2", "id3", ...]
client.delete("account", record_ids, use_bulk_delete=True)
```

---

## 6. Error Handling & Recovery

### Exception Hierarchy
```python
from PowerPlatform.Dataverse.core.errors import (
    DataverseError,           # Base class
    ValidationError,          # Validation failures
    MetadataError,           # Table/column operations
    HttpError,               # HTTP-level errors
    SQLParseError            # SQL query syntax errors
)

try:
    client.create("account", {"name": None})  # Invalid
except ValidationError as e:
    print(f"Validation failed: {e}")
    # Handle validation-specific logic
except DataverseError as e:
    print(f"General SDK error: {e}")
    # Handle other SDK errors
```

### Retry Logic Pattern
```python
import time
from PowerPlatform.Dataverse.core.errors import HttpError

def create_with_retry(table_name, record_data, max_retries=3):
    """Create record with exponential backoff retry logic."""
    for attempt in range(max_retries):
        try:
            return client.create(table_name, record_data)
        except HttpError as e:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff: 1s, 2s, 4s
            backoff_seconds = 2 ** attempt
            print(f"Attempt {attempt + 1} failed. Retrying in {backoff_seconds}s...")
            time.sleep(backoff_seconds)

# Usage
created_ids = create_with_retry("account", {"name": "Contoso"})
```

### 429 (Request Rate Limit) Handling
```python
import time
from PowerPlatform.Dataverse.core.errors import HttpError

try:
    accounts = client.get("account", top=5000)
except HttpError as e:
    if "429" in str(e):
        # Rate limited; wait and retry
        print("Rate limited. Waiting 60 seconds...")
        time.sleep(60)
        accounts = client.get("account", top=5000)
    else:
        raise
```

---

## 7. Table & Column Management

### Create Custom Table
```python
from enum import IntEnum

class Priority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

# Define columns with types
columns = {
    "new_Title": "string",
    "new_Quantity": "int",
    "new_Amount": "decimal",
    "new_Completed": "bool",
    "new_Priority": Priority,  # Creates option set/picklist
    "new_CreatedDate": "datetime"
}

table_info = client.create_table(
    "new_CustomTable",
    primary_column_schema_name="new_Name",
    columns=columns
)

print(f"Created table: {table_info['table_schema_name']}")
```

### Get Table Metadata
```python
table_info = client.get_table_info("account")
print(f"Schema Name: {table_info['table_schema_name']}")
print(f"Logical Name: {table_info['table_logical_name']}")
print(f"Entity Set: {table_info['entity_set_name']}")
print(f"Primary ID: {table_info['primary_id_attribute']}")
```

### List All Tables
```python
tables = client.list_tables()
for table in tables:
    print(f"{table['table_schema_name']} ({table['table_logical_name']})")
```

### Column Management
```python
# Add columns to existing table
client.create_columns("new_CustomTable", {
    "new_Status": "string",
    "new_Priority": "int"
})

# Delete columns
client.delete_columns("new_CustomTable", ["new_Status", "new_Priority"])

# Delete table
client.delete_table("new_CustomTable")
```

---

## 8. Paging & Large Result Sets

### Pagination Pattern
```python
# Retrieve all accounts in pages
all_accounts = []
for page in client.get(
    "account",
    top=500,      # Records per page
    page_size=500
):
    all_accounts.extend(page)
    print(f"Retrieved page with {len(page)} records")

print(f"Total: {len(all_accounts)} records")
```

### Manual Paging with Continuation Tokens
```python
# For complex paging scenarios
skip_count = 0
page_size = 1000

while True:
    page = client.get("account", top=page_size, skip=skip_count)
    if not page:
        break
    
    print(f"Page {skip_count // page_size + 1}: {len(page)} records")
    skip_count += page_size
```

---

## 9. File Operations

### Upload Small Files (< 128 MB)
```python
from pathlib import Path

file_path = Path("document.pdf")
record_id = "account-guid"

# Single PATCH upload
response = client.upload_file(
    table_name="account",
    record_id=record_id,
    file_column_name="new_documentfile",
    file_path=file_path
)
print(f"Upload successful: {response}")
```

### Upload Large Files with Chunking
```python
from pathlib import Path

file_path = Path("large_video.mp4")
record_id = "account-guid"

# SDK automatically chunks large files
response = client.upload_file(
    table_name="account",
    record_id=record_id,
    file_column_name="new_videofile",
    file_path=file_path,
    chunk_size=4 * 1024 * 1024  # 4 MB chunks
)
print(f"Chunked upload complete")
```

---

## 10. OData Filter Optimization

### Case Sensitivity Rules
```python
# ❌ WRONG: Uppercase logical names
results = client.get("account", filter="Name eq 'Contoso'")

# ✅ CORRECT: Lowercase logical names
results = client.get("account", filter="name eq 'Contoso'")

# ✅ Values ARE case-sensitive when needed
results = client.get("account", filter="name eq 'Contoso Ltd'")
```

### Filter Expression Examples
```python
# Equality
client.get("account", filter="name eq 'Contoso'")

# Greater than / Less than
client.get("account", filter="creditlimit gt 50000")
client.get("account", filter="createdon lt 2024-01-01")

# String contains
client.get("account", filter="contains(name, 'Ltd')")

# AND/OR operations
client.get("account", filter="(name eq 'Contoso') and (creditlimit gt 50000)")
client.get("account", filter="(industrycode eq 1) or (industrycode eq 2)")

# NOT operation
client.get("account", filter="not(statecode eq 1)")
```

### Select & Expand
```python
# Select specific columns (improves performance)
client.get("account", select=["name", "creditlimit", "telephone1"])

# Expand related records
client.get(
    "account",
    expand=["parentaccountid($select=name)"],
    select=["name", "parentaccountid"]
)
```

---

## 11. Cache Management

### Flushing Cache
```python
# Clear SDK internal cache after bulk operations
client.flush_cache()

# Useful after:
# - Metadata changes (table/column creation)
# - Bulk deletes
# - Metadata synchronization
```

---

## 12. Performance Best Practices

### Do's ✅
1. **Use `select` parameter**: Only fetch needed columns
   ```python
   client.get("account", select=["name", "creditlimit"])
   ```

2. **Batch operations**: Create/update multiple records at once
   ```python
   ids = client.create("account", [record1, record2, record3])
   ```

3. **Use paging**: Don't load all records at once
   ```python
   for page in client.get("account", top=1000):
       process_page(page)
   ```

4. **Reuse client instance**: Create once, use many times
   ```python
   client = DataverseClient(url, credential)  # Once
   # Reuse throughout app
   ```

5. **Apply filters on server**: Let Dataverse filter before returning
   ```python
   client.get("account", filter="creditlimit gt 50000")
   ```

### Don'ts ❌
1. **Don't fetch all columns**: Specify what you need
   ```python
   # Slow
   client.get("account")
   ```

2. **Don't create records in loops**: Batch them
   ```python
   # Slow
   for record in records:
       client.create("account", record)
   ```

3. **Don't load all results at once**: Use pagination
   ```python
   # Slow
   all_accounts = list(client.get("account"))
   ```

4. **Don't create new clients repeatedly**: Reuse singleton
   ```python
   # Inefficient
   for i in range(100):
       client = DataverseClient(url, credential)
   ```

---

## 13. Common Patterns Summary

### Pattern: Upsert (Create or Update)
```python
def upsert_account(name, data):
    """Create account or update if exists."""
    try:
        # Try to find existing
        results = list(client.get("account", filter=f"name eq '{name}'"))
        if results:
            account_id = results[0]['accountid']
            client.update("account", account_id, data)
            return account_id, "updated"
        else:
            ids = client.create("account", {"name": name, **data})
            return ids[0], "created"
    except Exception as e:
        print(f"Upsert failed: {e}")
        raise
```

### Pattern: Bulk Operation with Error Recovery
```python
def create_with_recovery(records):
    """Create records with per-record error tracking."""
    results = {"success": [], "failed": []}
    
    try:
        ids = client.create("account", records)
        results["success"] = ids
    except Exception as e:
        # If bulk fails, try individual records
        for i, record in enumerate(records):
            try:
                ids = client.create("account", record)
                results["success"].append(ids[0])
            except Exception as e:
                results["failed"].append({"index": i, "record": record, "error": str(e)})
    
    return results
```

---

## 14. Dependencies & Versions

### Core Dependencies
- **azure-identity** >= 1.17.0 (Authentication)
- **azure-core** >= 1.30.2 (HTTP client)
- **requests** >= 2.32.0 (HTTP requests)
- **Python** >= 3.10

### Optional Dependencies
- **pandas** (Data manipulation)
- **reportlab** (PDF generation for file examples)

### Development Tools
- **pytest** >= 7.0.0 (Testing)
- **black** >= 23.0.0 (Code formatting)
- **mypy** >= 1.0.0 (Type checking)
- **ruff** >= 0.1.0 (Linting)

---

## 15. Troubleshooting Common Issues

### ImportError: No module named 'PowerPlatform'
```bash
# Verify installation
pip show PowerPlatform-Dataverse-Client

# Reinstall
pip install --upgrade PowerPlatform-Dataverse-Client

# Check virtual environment is activated
which python  # Should show venv path
```

### Authentication Failed
```python
# Verify credentials have Dataverse access
# Try interactive auth first for testing
from azure.identity import InteractiveBrowserCredential
credential = InteractiveBrowserCredential(
    tenant_id="your-tenant-id"  # Specify if multiple tenants
)

# Check org URL format
# ✓ https://yourorg.crm.dynamics.com
# ❌ https://yourorg.crm.dynamics.com/
# ❌ https://yourorg.crm4.dynamics.com (regional)
```

### HTTP 429 Rate Limiting
```python
# Reduce request frequency
# Implement exponential backoff (see Error Handling section)
# Reduce page size
client.get("account", top=500)  # Instead of 5000
```

### MetadataError: Table Not Found
```python
# Verify table exists (schema name is case-insensitive for existence, but case-sensitive for API)
tables = client.list_tables()
print([t['table_schema_name'] for t in tables])

# Use exact schema name
table_info = client.get_table_info("new_customprefixed_table")
```

### SQL Query Not Enabled
```python
# query_sql() requires org config
# If disabled, fallback to OData
try:
    results = client.query_sql("SELECT * FROM account")
except Exception:
    # Fallback to OData
    results = client.get("account")
```

---

## Reference Links
- [Official Repository](https://github.com/microsoft/PowerPlatform-DataverseClient-Python)
- [PyPI Package](https://pypi.org/project/PowerPlatform-Dataverse-Client/)
- [Azure Identity Documentation](https://learn.microsoft.com/en-us/python/api/overview/azure/identity-readme)
- [Dataverse Web API Documentation](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview)

---

# Dataverse SDK for Python — API Reference Guide

## DataverseClient Class
Main client for interacting with Dataverse. Initialize with base URL and Azure credentials.

### Key Methods

#### create(table_schema_name, records)
Create single or bulk records. Returns list of GUIDs.

```python
# Single record
ids = client.create("account", {"name": "Acme"})
print(ids[0])  # First GUID

# Bulk create
ids = client.create("account", [{"name": "Contoso"}, {"name": "Fabrikam"}])
```

#### get(table_schema_name, record_id=None, select, filter, orderby, top, expand, page_size)
Fetch single record or query multiple with OData options.

```python
# Single record
record = client.get("account", record_id="guid-here")

# Query with filter and paging
for batch in client.get(
    "account",
    filter="statecode eq 0",
    select=["name", "telephone1"],
    orderby=["createdon desc"],
    top=100,
    page_size=50
):
    for record in batch:
        print(record["name"])
```

#### update(table_schema_name, ids, changes)
Update single or bulk records.

```python
# Single update
client.update("account", "guid-here", {"telephone1": "555-0100"})

# Broadcast: apply same changes to many IDs
client.update("account", [id1, id2, id3], {"statecode": 1})

# Paired: one-to-one mapping
client.update("account", [id1, id2], [{"name": "A"}, {"name": "B"}])
```

#### delete(table_schema_name, ids, use_bulk_delete=True)
Delete single or bulk records.

```python
# Single delete
client.delete("account", "guid-here")

# Bulk delete (async)
job_id = client.delete("account", [id1, id2, id3])
```

#### create_table(table_schema_name, columns, solution_unique_name=None, primary_column_schema_name=None)
Create custom table.

```python
from enum import IntEnum

class ItemStatus(IntEnum):
    ACTIVE = 1
    INACTIVE = 2
    __labels__ = {
        1033: {"ACTIVE": "Active", "INACTIVE": "Inactive"}
    }

info = client.create_table("new_MyTable", {
    "new_Title": "string",
    "new_Quantity": "int",
    "new_Price": "decimal",
    "new_Active": "bool",
    "new_Status": ItemStatus
})
print(info["entity_logical_name"])
```

#### create_columns(table_schema_name, columns)
Add columns to existing table.

```python
created = client.create_columns("new_MyTable", {
    "new_Notes": "string",
    "new_Count": "int"
})
```

#### delete_columns(table_schema_name, columns)
Remove columns from table.

```python
removed = client.delete_columns("new_MyTable", ["new_Notes", "new_Count"])
```

#### delete_table(table_schema_name)
Delete custom table (irreversible).

```python
client.delete_table("new_MyTable")
```

#### get_table_info(table_schema_name)
Retrieve table metadata.

```python
info = client.get_table_info("new_MyTable")
if info:
    print(info["table_logical_name"])
    print(info["entity_set_name"])
```

#### list_tables()
List all custom tables.

```python
tables = client.list_tables()
for table in tables:
    print(table)
```

#### flush_cache(kind)
Clear SDK caches (e.g., picklist labels).

```python
removed = client.flush_cache("picklist")
```

## DataverseConfig Class
Configure client behavior (timeouts, retries, language).

```python
from PowerPlatform.Dataverse.core.config import DataverseConfig

cfg = DataverseConfig()
cfg.http_retries = 3
cfg.http_backoff = 1.0
cfg.http_timeout = 30
cfg.language_code = 1033  # English

client = DataverseClient(base_url=url, credential=cred, config=cfg)
```

## Error Handling
Catch `DataverseError` for SDK-specific exceptions. Check `is_transient` to decide retry.

```python
from PowerPlatform.Dataverse.core.errors import DataverseError

try:
    client.create("account", {"name": "Test"})
except DataverseError as e:
    print(f"Code: {e.code}")
    print(f"Message: {e.message}")
    print(f"Transient: {e.is_transient}")
    print(f"Details: {e.to_dict()}")
```

## OData Filter Tips
- Use exact logical names (lowercase) in filter expressions
- Column names in `select` are auto-lowercased
- Navigation property names in `expand` are case-sensitive

## References
- API docs: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.client.dataverseclient
- Config docs: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.core.config.dataverseconfig
- Errors: https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.core.errors