---
description: 'Dataverse Python: advanced features, agentic workflows, performance and real-world examples'
applyTo: '**'
---

# Dataverse SDK for Python - Advanced Features Guide

## Overview
Comprehensive guide to advanced Dataverse SDK features including enums, complex filtering, SQL queries, metadata operations, and production patterns. Based on official Microsoft walkthrough examples.

## 1. Working with Option Sets & Picklists

### Using IntEnum for Type Safety
```python
from enum import IntEnum
from PowerPlatform.Dataverse.client import DataverseClient

# Define enum for picklist
class Priority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

class Priority(IntEnum):
    COLD = 1
    WARM = 2
    HOT = 3

# Create record with enum value
record_data = {
    "new_title": "Important Task",
    "new_priority": Priority.HIGH,  # Automatically converted to int
}

ids = client.create("new_tasktable", record_data)
```

### Handling Formatted Values
```python
# When retrieving records, picklist values are returned as integers
record = client.get("new_tasktable", record_id)

priority_int = record.get("new_priority")  # Returns: 3
priority_formatted = record.get("new_priority@OData.Community.Display.V1.FormattedValue")  # Returns: "High"

print(f"Priority (Raw): {priority_int}")
print(f"Priority (Formatted): {priority_formatted}")
```

### Creating Tables with Enum Columns
```python
from enum import IntEnum

class TaskStatus(IntEnum):
    NOT_STARTED = 0
    IN_PROGRESS = 1
    COMPLETED = 2

class TaskPriority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

# Pass enum classes as column types
columns = {
    "new_Title": "string",
    "new_Description": "string",
    "new_Status": TaskStatus,      # Creates option set column
    "new_Priority": TaskPriority,  # Creates option set column
    "new_Amount": "decimal",
    "new_DueDate": "datetime"
}

table_info = client.create_table(
    "new_TaskManagement",
    primary_column_schema_name="new_Title",
    columns=columns
)

print(f"Created table with {len(columns)} columns including enums")
```

---

## 2. Advanced Filtering & Querying

### Complex OData Filters
```python
# Simple equality
filter1 = "name eq 'Contoso'"

# Comparison operators
filter2 = "creditlimit gt 50000"
filter3 = "createdon lt 2024-01-01"

# String operations
filter4 = "contains(name, 'Ltd')"
filter5 = "startswith(name, 'Con')"
filter6 = "endswith(name, 'Ltd')"

# Multiple conditions with AND
filter7 = "(name eq 'Contoso') and (creditlimit gt 50000)"

# Multiple conditions with OR
filter8 = "(industrycode eq 1) or (industrycode eq 2)"

# Negation
filter9 = "not(statecode eq 1)"

# Complex nested conditions
filter10 = "(creditlimit gt 50000) and ((industrycode eq 1) or (industrycode eq 2))"

# Using in get() calls
results = client.get("account", filter=filter10, select=["name", "creditlimit"])
```

### Retrieve with Related Records (Expand)
```python
# Expand parent account information
accounts = client.get(
    "account",
    filter="creditlimit gt 100000",
    expand=["parentaccountid($select=name,creditlimit)"],
    select=["accountid", "name", "creditlimit", "parentaccountid"]
)

for page in accounts:
    for account in page:
        parent_name = account.get("_parentaccountid_value")
        print(f"Account: {account['name']}, Parent: {parent_name}")
```

### SQL Queries for Complex Analysis
```python
# SQL queries are read-only but powerful for analytics
sql = """
SELECT 
    a.name as AccountName,
    a.creditlimit,
    COUNT(c.contactid) as ContactCount
FROM account a
LEFT JOIN contact c ON a.accountid = c.parentcustomerid
WHERE a.creditlimit > 50000
GROUP BY a.accountid, a.name, a.creditlimit
ORDER BY ContactCount DESC
"""

results = client.query_sql(sql)
for row in results:
    print(f"{row['AccountName']}: {row['ContactCount']} contacts")
```

### Paging with SQL Queries
```python
# SQL queries return paginated results by default
sql = "SELECT TOP 10000 name, creditlimit FROM account ORDER BY name"

all_results = []
for page in client.query_sql(sql):
    all_results.extend(page)
    print(f"Retrieved {len(page)} rows")

print(f"Total: {len(all_results)} rows")
```

---

## 3. Metadata Operations

### Creating Complex Tables
```python
from enum import IntEnum
from datetime import datetime

class TaskStatus(IntEnum):
    NEW = 1
    OPEN = 2
    CLOSED = 3

# Create table with diverse column types
columns = {
    "new_Subject": "string",
    "new_Description": "string",
    "new_Category": "string",
    "new_Priority": "int",
    "new_Status": TaskStatus,
    "new_EstimatedHours": "decimal",
    "new_DueDate": "datetime",
    "new_IsOverdue": "bool",
    "new_Notes": "string"
}

table_info = client.create_table(
    "new_WorkItem",
    primary_column_schema_name="new_Subject",
    columns=columns
)

print(f"✓ Created table: {table_info['table_schema_name']}")
print(f"  Primary Key: {table_info['primary_id_attribute']}")
print(f"  Columns: {', '.join(table_info.get('columns_created', []))}")
```

### Inspecting Table Metadata
```python
# Get detailed table information
table_info = client.get_table_info("account")

print(f"Schema Name: {table_info.get('table_schema_name')}")
print(f"Logical Name: {table_info.get('table_logical_name')}")
print(f"Display Name: {table_info.get('table_display_name')}")
print(f"Entity Set: {table_info.get('entity_set_name')}")
print(f"Primary ID: {table_info.get('primary_id_attribute')}")
print(f"Primary Name: {table_info.get('primary_name_attribute')}")
```

### Listing All Tables in Organization
```python
# Retrieve all tables (may be large result set)
all_tables = []
for page in client.list_tables():
    all_tables.extend(page)
    print(f"Retrieved {len(page)} tables in this page")

print(f"\nTotal tables: {len(all_tables)}")

# Filter for custom tables
custom_tables = [t for t in all_tables if t['table_schema_name'].startswith('new_')]
print(f"Custom tables: {len(custom_tables)}")
for table in custom_tables[:5]:
    print(f"  - {table['table_schema_name']}")
```

### Managing Columns Dynamically
```python
# Add columns to existing table
client.create_columns("new_TaskTable", {
    "new_Department": "string",
    "new_Budget": "decimal",
    "new_ApprovedDate": "datetime"
})

# Delete specific columns
client.delete_columns("new_TaskTable", [
    "new_OldField1",
    "new_OldField2"
])

# Delete entire table
client.delete_table("new_TaskTable")
```

---

## 4. Single vs. Multiple Record Operations

### Single Record Operations
```python
# Create single
record_id = client.create("account", {"name": "Contoso"})[0]

# Get single by ID
account = client.get("account", record_id)

# Update single
client.update("account", record_id, {"creditlimit": 100000})

# Delete single
client.delete("account", record_id)
```

### Multiple Record Operations

#### Create Multiple Records
```python
# Create list of records
records = [
    {"name": "Company A", "creditlimit": 50000},
    {"name": "Company B", "creditlimit": 75000},
    {"name": "Company C", "creditlimit": 100000},
]

created_ids = client.create("account", records)
print(f"Created {len(created_ids)} records: {created_ids}")
```

#### Update Multiple Records (Broadcast)
```python
# Apply same update to multiple records
account_ids = ["id1", "id2", "id3"]
client.update("account", account_ids, {
    "industrycode": 1,  # Retail
    "accountmanagerid": "manager-guid"
})
print(f"Updated {len(account_ids)} records with same data")
```

#### Delete Multiple Records
```python
# Delete multiple records with optimized bulk delete
record_ids = ["id1", "id2", "id3", "id4", "id5"]
client.delete("account", record_ids, use_bulk_delete=True)
print(f"Deleted {len(record_ids)} records")
```

---

## 5. Data Manipulation Patterns

### Retrieve, Modify, Update Pattern
```python
# Retrieve single record
account = client.get("account", record_id)

# Modify locally
original_amount = account.get("creditlimit", 0)
new_amount = original_amount + 10000

# Update back
client.update("account", record_id, {"creditlimit": new_amount})
print(f"Updated creditlimit: {original_amount} → {new_amount}")
```

### Batch Processing Pattern
```python
# Retrieve in batches with paging
batch_size = 100
processed = 0

for page in client.get("account", top=batch_size, filter="statecode eq 0"):
    # Process each page
    batch_updates = []
    for account in page:
        if account.get("creditlimit", 0) > 100000:
            batch_updates.append({
                "id": account['accountid'],
                "accountmanagerid": "senior-manager-guid"
            })
    
    # Batch update
    for update in batch_updates:
        client.update("account", update['id'], {"accountmanagerid": update['accountmanagerid']})
        processed += 1

print(f"Processed {processed} accounts")
```

### Conditional Operations Pattern
```python
from PowerPlatform.Dataverse.core.errors import DataverseError

def safe_update(table, record_id, data, check_field=None, check_value=None):
    """Update with pre-condition check."""
    try:
        if check_field and check_value:
            # Verify condition before updating
            record = client.get(table, record_id, select=[check_field])
            if record.get(check_field) != check_value:
                print(f"Condition not met: {check_field} != {check_value}")
                return False
        
        client.update(table, record_id, data)
        return True
    except DataverseError as e:
        print(f"Update failed: {e}")
        return False

# Usage
safe_update("account", account_id, {"creditlimit": 100000}, "statecode", 0)
```

---

## 6. Formatted Values & Display

### Retrieving Formatted Values
```python
# When you retrieve a record with option set or money fields,
# you can request formatted values for display

record = client.get(
    "account",
    record_id,
    select=["name", "creditlimit", "industrycode"]
)

# Raw values
name = record.get("name")  # "Contoso Ltd"
limit = record.get("creditlimit")  # 100000.00
industry = record.get("industrycode")  # 1

# Formatted values (returned in OData response)
limit_formatted = record.get("creditlimit@OData.Community.Display.V1.FormattedValue")
industry_formatted = record.get("industrycode@OData.Community.Display.V1.FormattedValue")

print(f"Name: {name}")
print(f"Credit Limit: {limit_formatted or limit}")  # "100,000.00" or 100000.00
print(f"Industry: {industry_formatted or industry}")  # "Technology" or 1
```

---

## 7. Performance Optimization

### Column Selection Strategy
```python
# ❌ Retrieve all columns (slow, uses more bandwidth)
account = client.get("account", record_id)

# ✅ Retrieve only needed columns (fast, efficient)
account = client.get(
    "account",
    record_id,
    select=["accountid", "name", "creditlimit", "telephone1"]
)
```

### Filtering on Server
```python
# ❌ Retrieve all, filter locally (inefficient)
all_accounts = []
for page in client.get("account"):
    all_accounts.extend(page)
large_accounts = [a for a in all_accounts if a.get("creditlimit", 0) > 100000]

# ✅ Filter on server, retrieve only matches (efficient)
large_accounts = []
for page in client.get("account", filter="creditlimit gt 100000"):
    large_accounts.extend(page)
```

### Paging Large Result Sets
```python
# ❌ Load all results at once (memory intensive)
all_accounts = list(client.get("account"))

# ✅ Process in pages (memory efficient)
processed = 0
for page in client.get("account", top=1000):
    for account in page:
        process_account(account)
        processed += 1
    print(f"Processed: {processed}")
```

### Batch Operations
```python
# ❌ Individual creates in loop (slow)
for account_data in accounts:
    client.create("account", account_data)

# ✅ Batch create (fast, optimized)
created_ids = client.create("account", accounts)
```

---

## 8. Error Handling in Advanced Scenarios

### Handling Metadata Errors
```python
from PowerPlatform.Dataverse.core.errors import MetadataError

try:
    table_info = client.create_table("new_CustomTable", {"name": "string"})
except MetadataError as e:
    print(f"Metadata operation failed: {e}")
    # Handle table creation specific errors
```

### Handling Validation Errors
```python
from PowerPlatform.Dataverse.core.errors import ValidationError

try:
    client.create("account", {"name": None})  # Invalid: name required
except ValidationError as e:
    print(f"Validation error: {e}")
    # Handle validation specific errors
```

### Handling HTTP Errors
```python
from PowerPlatform.Dataverse.core.errors import HttpError

try:
    client.get("account", "invalid-guid")
except HttpError as e:
    if "404" in str(e):
        print("Record not found")
    elif "403" in str(e):
        print("Access denied")
    else:
        print(f"HTTP error: {e}")
```

### Handling SQL Errors
```python
from PowerPlatform.Dataverse.core.errors import SQLParseError

try:
    results = client.query_sql("SELECT INVALID SYNTAX")
except SQLParseError as e:
    print(f"SQL parse error: {e}")
```

---

## 9. Working with Relationships

### Creating Related Records
```python
# Create parent account
parent_ids = client.create("account", {
    "name": "Parent Company",
    "creditlimit": 500000
})
parent_id = parent_ids[0]

# Create child accounts with parent reference
children = [
    {"name": "Subsidiary A", "parentaccountid": parent_id},
    {"name": "Subsidiary B", "parentaccountid": parent_id},
    {"name": "Subsidiary C", "parentaccountid": parent_id},
]
child_ids = client.create("account", children)
print(f"Created {len(child_ids)} child accounts")
```

### Querying Related Records
```python
# Get account with child accounts
account = client.get("account", account_id)

# Query child accounts
children = client.get(
    "account",
    filter=f"parentaccountid eq {account_id}",
    select=["accountid", "name", "creditlimit"]
)

for page in children:
    for child in page:
        print(f"  - {child['name']}: ${child['creditlimit']}")
```

---

## 10. Cleanup & Housekeeping

### Clearing SDK Cache
```python
# After bulk operations, clear metadata cache
client.flush_cache()

# Useful after:
# - Massive delete operations
# - Table/column creation or deletion
# - Metadata synchronization across environments
```

### Safe Table Deletion
```python
from PowerPlatform.Dataverse.core.errors import MetadataError

def delete_table_safe(table_name):
    """Delete table with error handling."""
    try:
        # Verify table exists
        table_info = client.get_table_info(table_name)
        if not table_info:
            print(f"Table {table_name} not found")
            return False
        
        # Delete
        client.delete_table(table_name)
        print(f"✓ Deleted table: {table_name}")
        
        # Clear cache
        client.flush_cache()
        return True
        
    except MetadataError as e:
        print(f"❌ Failed to delete table: {e}")
        return False

delete_table_safe("new_TempTable")
```

---

## 11. Comprehensive Example: Full Workflow

```python
from enum import IntEnum
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.errors import DataverseError, MetadataError

class TaskStatus(IntEnum):
    NEW = 1
    IN_PROGRESS = 2
    COMPLETED = 3

class TaskPriority(IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3

# Setup
credential = InteractiveBrowserCredential()
client = DataverseClient("https://yourorg.crm.dynamics.com", credential)

try:
    # 1. Create table
    print("Creating table...")
    table_info = client.create_table(
        "new_ProjectTask",
        primary_column_schema_name="new_Title",
        columns={
            "new_Description": "string",
            "new_Status": TaskStatus,
            "new_Priority": TaskPriority,
            "new_DueDate": "datetime",
            "new_EstimatedHours": "decimal"
        }
    )
    print(f"✓ Created table: {table_info['table_schema_name']}")
    
    # 2. Create records
    print("\nCreating tasks...")
    tasks = [
        {
            "new_Title": "Design system",
            "new_Description": "Create design system architecture",
            "new_Status": TaskStatus.NEW,
            "new_Priority": TaskPriority.HIGH,
            "new_EstimatedHours": 40.0
        },
        {
            "new_Title": "Implement UI",
            "new_Description": "Build React components",
            "new_Status": TaskStatus.IN_PROGRESS,
            "new_Priority": TaskPriority.HIGH,
            "new_EstimatedHours": 80.0
        },
        {
            "new_Title": "Write tests",
            "new_Description": "Unit and integration tests",
            "new_Status": TaskStatus.NEW,
            "new_Priority": TaskPriority.MEDIUM,
            "new_EstimatedHours": 30.0
        }
    ]
    task_ids = client.create("new_ProjectTask", tasks)
    print(f"✓ Created {len(task_ids)} tasks")
    
    # 3. Query and filter
    print("\nQuerying high-priority tasks...")
    high_priority = client.get(
        "new_ProjectTask",
        filter="new_priority eq 3",
        select=["new_Title", "new_Priority", "new_EstimatedHours"]
    )
    for page in high_priority:
        for task in page:
            print(f"  - {task['new_title']}: {task['new_estimatedhours']} hours")
    
    # 4. Update records
    print("\nUpdating task status...")
    client.update("new_ProjectTask", task_ids[1], {
        "new_Status": TaskStatus.COMPLETED,
        "new_EstimatedHours": 85.5
    })
    print("✓ Updated task status")
    
    # 5. Cleanup
    print("\nCleaning up...")
    client.delete_table("new_ProjectTask")
    print("✓ Deleted table")
    
    # Clear cache
    client.flush_cache()
    
except (MetadataError, DataverseError) as e:
    print(f"❌ Error: {e}")
```

---

## Reference
- [Official Walkthrough Example](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/blob/main/examples/advanced/walkthrough.py)
- [OData Filter Syntax](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/query-data-web-api)
- [Table/Column Metadata](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/create-update-entity-definitions-using-web-api)

---

# Dataverse SDK for Python - Agentic Workflows Guide

## ⚠️ PREVIEW FEATURE NOTICE

**Status**: This feature is in **Public Preview** as of December 2025  
**Availability**: General Availability (GA) date TBD  
**Documentation**: Complete implementation details forthcoming  

This guide covers the conceptual framework and planned capabilities for building agentic workflows with the Dataverse SDK for Python. Specific APIs and implementations may change before general availability.

---

## 1. Overview: Agentic Workflows with Dataverse

### What are Agentic Workflows?

Agentic workflows are autonomous, intelligent processes where:
- **Agents** make decisions and take actions based on data and rules
- **Workflows** orchestrate complex, multi-step operations
- **Dataverse** serves as the central source of truth for enterprise data

The Dataverse SDK for Python is designed to enable data scientists and developers to build these intelligent systems without .NET expertise.

### Key Capabilities (Planned)

The SDK is strategically positioned to support:

1. **Autonomous Data Agents** - Query, update, and evaluate data quality independently
2. **Form Prediction & Autofill** - Pre-fill forms based on data patterns and context
3. **Model Context Protocol (MCP)** Support - Enable standardized agent-to-tool communication
4. **Agent-to-Agent (A2A)** Collaboration - Multiple agents working together on complex tasks
5. **Semantic Modeling** - Natural language understanding of data relationships
6. **Secure Impersonation** - Run operations on behalf of specific users with audit trails
7. **Compliance Built-in** - Data governance and retention policies enforced

---

## 2. Architecture Patterns for Agentic Systems

### Multi-Agent Pattern
```python
# Conceptual pattern - specific APIs pending GA
class DataQualityAgent:
    """Autonomous agent that monitors and improves data quality."""
    
    def __init__(self, client):
        self.client = client
    
    async def evaluate_data_quality(self, table_name):
        """Evaluate data quality metrics for a table."""
        records = await self.client.get(table_name)
        
        metrics = {
            'total_records': len(records),
            'null_values': sum(1 for r in records if None in r.values()),
            'duplicate_records': await self._find_duplicates(table_name)
        }
        return metrics
    
    async def auto_remediate(self, issues):
        """Automatically fix identified data quality issues."""
        # Agent autonomously decides on remediation actions
        pass

class DataEnrichmentAgent:
    """Autonomous agent that enriches data from external sources."""
    
    async def enrich_accounts(self):
        """Enrich account data with market information."""
        accounts = await self.client.get("account")
        
        for account in accounts:
            enrichment = await self._lookup_market_data(account['name'])
            await self.client.update("account", account['id'], enrichment)
```

### Agent Orchestration Pattern
```python
# Conceptual pattern - specific APIs pending GA
class DataPipeline:
    """Orchestrates multiple agents working together."""
    
    def __init__(self, client):
        self.quality_agent = DataQualityAgent(client)
        self.enrichment_agent = DataEnrichmentAgent(client)
        self.sync_agent = SyncAgent(client)
    
    async def run(self, table_name):
        """Execute multi-agent workflow."""
        # Step 1: Quality check
        print("Running quality checks...")
        issues = await self.quality_agent.evaluate_data_quality(table_name)
        
        # Step 2: Enrich data
        print("Enriching data...")
        await self.enrichment_agent.enrich_accounts()
        
        # Step 3: Sync to external systems
        print("Syncing to external systems...")
        await self.sync_agent.sync_to_external_db(table_name)
```

---

## 3. Model Context Protocol (MCP) Support (Planned)

### What is MCP?

The Model Context Protocol (MCP) is an open standard for:
- **Tool Definition** - Describe what tools/capabilities are available
- **Tool Invocation** - Allow LLMs to call tools with parameters
- **Context Management** - Manage context between agent and tools
- **Error Handling** - Standardized error responses

### MCP Integration Pattern (Conceptual)

```python
# Conceptual pattern - specific APIs pending GA
from dataverse_mcp import DataverseMCPServer

# Define available tools
tools = [
    {
        "name": "query_accounts",
        "description": "Query accounts with filters",
        "parameters": {
            "filter": "OData filter expression",
            "select": "Columns to retrieve",
            "top": "Maximum records"
        }
    },
    {
        "name": "create_account",
        "description": "Create a new account",
        "parameters": {
            "name": "Account name",
            "credit_limit": "Credit limit amount"
        }
    },
    {
        "name": "update_account",
        "description": "Update account fields",
        "parameters": {
            "account_id": "Account GUID",
            "updates": "Dictionary of field updates"
        }
    }
]

# Create MCP server
server = DataverseMCPServer(client, tools=tools)

# LLMs can now use Dataverse tools
await server.handle_tool_call("query_accounts", {
    "filter": "creditlimit gt 100000",
    "select": ["name", "creditlimit"]
})
```

---

## 4. Agent-to-Agent (A2A) Collaboration (Planned)

### A2A Communication Pattern

```python
# Conceptual pattern - specific APIs pending GA
class DataValidationAgent:
    """Validates data before downstream agents process it."""
    
    async def validate_and_notify(self, data):
        """Validate data and notify other agents."""
        if await self._is_valid(data):
            # Publish event that other agents can subscribe to
            await self.publish_event("data_validated", data)
        else:
            await self.publish_event("validation_failed", data)

class DataProcessingAgent:
    """Waits for valid data from validation agent."""
    
    async def __init__(self):
        self.subscribe("data_validated", self.process_data)
    
    async def process_data(self, data):
        """Process already-validated data."""
        # Agent can safely assume data is valid
        result = await self._transform(data)
        await self.publish_event("processing_complete", result)
```

---

## 5. Building Autonomous Data Agents

### Data Quality Agent Example
```python
# Working example with current SDK features
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import InteractiveBrowserCredential
import json

class DataQualityAgent:
    """Monitor and report on data quality."""
    
    def __init__(self, org_url, credential):
        self.client = DataverseClient(org_url, credential)
    
    def analyze_completeness(self, table_name, required_fields):
        """Analyze field completeness."""
        records = self.client.get(
            table_name,
            select=required_fields
        )
        
        missing_by_field = {field: 0 for field in required_fields}
        total = 0
        
        for page in records:
            for record in page:
                total += 1
                for field in required_fields:
                    if field not in record or record[field] is None:
                        missing_by_field[field] += 1
        
        # Calculate completeness percentage
        completeness = {
            field: ((total - count) / total * 100) 
            for field, count in missing_by_field.items()
        }
        
        return {
            'table': table_name,
            'total_records': total,
            'completeness': completeness,
            'missing_counts': missing_by_field
        }
    
    def detect_duplicates(self, table_name, key_fields):
        """Detect potential duplicate records."""
        records = self.client.get(table_name, select=key_fields)
        
        all_records = []
        for page in records:
            all_records.extend(page)
        
        seen = {}
        duplicates = []
        
        for record in all_records:
            key = tuple(record.get(f) for f in key_fields)
            if key in seen:
                duplicates.append({
                    'original_id': seen[key],
                    'duplicate_id': record.get('id'),
                    'key': key
                })
            else:
                seen[key] = record.get('id')
        
        return {
            'table': table_name,
            'duplicate_count': len(duplicates),
            'duplicates': duplicates
        }
    
    def generate_quality_report(self, table_name):
        """Generate comprehensive quality report."""
        completeness = self.analyze_completeness(
            table_name,
            ['name', 'telephone1', 'emailaddress1']
        )
        
        duplicates = self.detect_duplicates(
            table_name,
            ['name', 'emailaddress1']
        )
        
        return {
            'timestamp': pd.Timestamp.now().isoformat(),
            'table': table_name,
            'completeness': completeness,
            'duplicates': duplicates
        }

# Usage
client = DataverseClient("https://<org>.crm.dynamics.com", InteractiveBrowserCredential())
agent = DataQualityAgent("https://<org>.crm.dynamics.com", InteractiveBrowserCredential())

report = agent.generate_quality_report("account")
print(json.dumps(report, indent=2))
```

### Form Prediction Agent Example
```python
# Conceptual pattern using current SDK capabilities
from sklearn.ensemble import RandomForestRegressor
import pandas as pd

class FormPredictionAgent:
    """Predict and autofill form values."""
    
    def __init__(self, org_url, credential):
        self.client = DataverseClient(org_url, credential)
        self.model = None
    
    def train_on_historical_data(self, table_name, features, target):
        """Train prediction model on historical data."""
        # Collect training data
        records = []
        for page in self.client.get(table_name, select=features + [target]):
            records.extend(page)
        
        df = pd.DataFrame(records)
        
        # Train model
        X = df[features].fillna(0)
        y = df[target]
        
        self.model = RandomForestRegressor()
        self.model.fit(X, y)
        
        return self.model.score(X, y)
    
    def predict_field_values(self, table_name, record_id, features_data):
        """Predict missing field values."""
        if self.model is None:
            raise ValueError("Model not trained. Call train_on_historical_data first.")
        
        # Predict
        prediction = self.model.predict([features_data])[0]
        
        # Return prediction with confidence
        return {
            'record_id': record_id,
            'predicted_value': prediction,
            'confidence': self.model.score([features_data], [prediction])
        }
```

---

## 6. Integration with AI/ML Services

### LLM Integration Pattern
```python
# Using LLM to interpret Dataverse data
from openai import OpenAI

class DataInsightAgent:
    """Use LLM to generate insights from Dataverse data."""
    
    def __init__(self, org_url, credential, openai_key):
        self.client = DataverseClient(org_url, credential)
        self.llm = OpenAI(api_key=openai_key)
    
    def analyze_with_llm(self, table_name, sample_size=100):
        """Analyze data using LLM."""
        # Get sample data
        records = []
        count = 0
        for page in self.client.get(table_name):
            records.extend(page)
            count += len(page)
            if count >= sample_size:
                break
        
        # Create summary for LLM
        summary = f"""
        Table: {table_name}
        Total records sampled: {len(records)}
        
        Sample data:
        {json.dumps(records[:5], indent=2, default=str)}
        
        Provide insights about this data.
        """
        
        # Ask LLM
        response = self.llm.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": summary}]
        )
        
        return response.choices[0].message.content
```

---

## 7. Secure Impersonation & Audit Trails

### Planned Capabilities

The SDK will support running operations on behalf of specific users:

```python
# Conceptual pattern - specific APIs pending GA
from dataverse_security import ImpersonationContext

# Run as different user
with ImpersonationContext(client, user_id="user-guid"):
    # All operations run as this user
    client.create("account", {"name": "New Account"})
    # Audit trail: Created by [user-guid] at [timestamp]

# Retrieve audit trail
audit_log = client.get_audit_trail(
    table="account",
    record_id="record-guid",
    action="create"
)
```

---

## 8. Compliance and Data Governance

### Planned Governance Features

```python
# Conceptual pattern - specific APIs pending GA
from dataverse_governance import DataGovernance

# Define retention policy
governance = DataGovernance(client)
governance.set_retention_policy(
    table="account",
    retention_days=365
)

# Define data classification
governance.classify_columns(
    table="account",
    classifications={
        "name": "Public",
        "telephone1": "Internal",
        "creditlimit": "Confidential"
    }
)

# Enforce policies
governance.enforce_all_policies()
```

---

## 9. Current SDK Capabilities Supporting Agentic Workflows

While full agentic features are in preview, current SDK capabilities already support agent building:

### ✅ Available Now
- **CRUD Operations** - Create, retrieve, update, delete data
- **Bulk Operations** - Process large datasets efficiently
- **Query Capabilities** - OData and SQL for flexible data retrieval
- **Metadata Operations** - Work with table and column definitions
- **Error Handling** - Structured exception hierarchy
- **Pagination** - Handle large result sets
- **File Upload** - Manage document attachments

### 🔜 Coming in GA
- Full MCP integration
- A2A collaboration primitives
- Enhanced authentication/impersonation
- Governance policy enforcement
- Native async/await support
- Advanced caching strategies

---

## 10. Getting Started: Build Your First Agent Today

```python
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import InteractiveBrowserCredential
import json

class SimpleDataAgent:
    """Your first Dataverse agent."""
    
    def __init__(self, org_url):
        credential = InteractiveBrowserCredential()
        self.client = DataverseClient(org_url, credential)
    
    def check_health(self, table_name):
        """Agent function: Check table health."""
        try:
            tables = self.client.list_tables()
            matching = [t for t in tables if t['LogicalName'] == table_name]
            
            if not matching:
                return {"status": "error", "message": f"Table {table_name} not found"}
            
            # Get record count
            records = []
            for page in self.client.get(table_name):
                records.extend(page)
                if len(records) > 1000:
                    break
            
            return {
                "status": "healthy",
                "table": table_name,
                "record_count": len(records),
                "timestamp": pd.Timestamp.now().isoformat()
            }
        
        except Exception as e:
            return {"status": "error", "message": str(e)}

# Usage
agent = SimpleDataAgent("https://<org>.crm.dynamics.com")
health = agent.check_health("account")
print(json.dumps(health, indent=2))
```

---

## 11. Resources & Documentation

### Official Documentation
- [Dataverse SDK for Python Overview](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/overview)
- [Working with Data](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/work-data)
- [Release Plan: Agentic Workflows](https://learn.microsoft.com/en-us/power-platform/release-plan/2025wave2/data-platform/build-agentic-flows-dataverse-sdk-python)

### External Resources
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Azure AI Services](https://learn.microsoft.com/en-us/azure/ai-services/)
- [Python async/await](https://docs.python.org/3/library/asyncio.html)

### Repository
- [SDK Source Code](https://github.com/microsoft/PowerPlatform-DataverseClient-Python)
- [Issues & Feature Requests](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/issues)

---

## 12. FAQ: Agentic Workflows

**Q: Can I use agents today with the current SDK?**  
A: Yes! Use the current capabilities to build agent-like systems. Full MCP/A2A support coming in GA.

**Q: What's the difference between current SDK and agentic features?**  
A: Current: Synchronous CRUD; Agentic: Async, autonomous decision-making, agent collaboration.

**Q: Will there be breaking changes from preview to GA?**  
A: Possibly. This is a preview feature; expect API refinements before general availability.

**Q: How do I prepare for agentic workflows today?**  
A: Build agents using current CRUD operations, design with async patterns in mind, use MCP specs for future compatibility.

**Q: Is there a cost difference for agentic features?**  
A: Unknown at this time. Check release notes closer to GA.

---

## 13. Next Steps

1. **Build a prototype** using current SDK capabilities
2. **Join preview** when MCP integration becomes available
3. **Provide feedback** via GitHub issues
4. **Watch for GA announcement** with full API documentation
5. **Migrate to full agentic** features when ready

The Dataverse SDK for Python is positioning itself as the go-to platform for building intelligent, autonomous data systems on the Microsoft Power Platform.

---


# Dataverse SDK for Python — Performance & Optimization Guide

Based on official Microsoft Dataverse and Azure SDK performance guidance.

## 1. Performance Overview

The Dataverse SDK for Python is optimized for Python developers but has some limitations in preview:
- **Minimal retry policy**: Only network errors are retried by default
- **No DeleteMultiple**: Use individual deletes or update status instead
- **Limited OData batching**: General-purpose OData batching not supported
- **SQL limitations**: No JOINs, limited WHERE/TOP/ORDER BY

Workarounds and optimization strategies address these limitations.

---

## 2. Query Optimization

### Use Select to Limit Columns

```python
# ❌ SLOW - Retrieves all columns
accounts = client.get("account", top=100)

# ✅ FAST - Only retrieve needed columns
accounts = client.get(
    "account",
    select=["accountid", "name", "telephone1", "creditlimit"],
    top=100
)
```

**Impact**: Reduces payload size and memory usage by 30-50%.

---

### Use Filters Efficiently

```python
# ❌ SLOW - Fetch all, filter in Python
all_accounts = client.get("account")
active_accounts = [a for a in all_accounts if a.get("statecode") == 0]

# ✅ FAST - Filter server-side
accounts = client.get(
    "account",
    filter="statecode eq 0",
    top=100
)
```

**OData filter examples**:
```python
# Equals
filter="statecode eq 0"

# String contains
filter="contains(name, 'Acme')"

# Multiple conditions
filter="statecode eq 0 and createdon gt 2025-01-01Z"

# Not equals
filter="statecode ne 2"
```

---

### Order by for Predictable Paging

```python
# Ensure consistent order for pagination
accounts = client.get(
    "account",
    orderby=["createdon desc", "name asc"],
    page_size=100
)

for page in accounts:
    process_page(page)
```

---

## 3. Pagination Best Practices

### Lazy Pagination (Recommended)

```python
# ✅ BEST - Generator yields one page at a time
pages = client.get(
    "account",
    top=5000,              # Total limit
    page_size=200          # Per-page size (hint)
)

for page in pages:  # Each iteration fetches one page
    for record in page:
        process_record(record)  # Process immediately
```

**Benefits**:
- Memory efficient (pages loaded on-demand)
- Fast time-to-first-result
- Can stop early if needed

### Avoid Loading Everything into Memory

```python
# ❌ SLOW - Loads all 100,000 records at once
all_records = list(client.get("account", top=100000))
process(all_records)

# ✅ FAST - Process as you go
for page in client.get("account", top=100000, page_size=5000):
    process(page)
```

---

## 4. Batch Operations

### Bulk Create (Recommended)

```python
# ✅ BEST - Single call with multiple records
payloads = [
    {"name": f"Account {i}", "telephone1": f"555-{i:04d}"}
    for i in range(1000)
]
ids = client.create("account", payloads)  # One API call for many records
```

### Bulk Update - Broadcast Mode

```python
# ✅ FAST - Same update applied to many records
account_ids = ["id1", "id2", "id3", "..."]
client.update("account", account_ids, {"statecode": 1})  # One call
```

### Bulk Update - Per-Record Mode

```python
# ✅ ACCEPTABLE - Different updates for each record
account_ids = ["id1", "id2", "id3"]
updates = [
    {"telephone1": "555-0100"},
    {"telephone1": "555-0200"},
    {"telephone1": "555-0300"},
]
client.update("account", account_ids, updates)
```

### Batch Size Tuning

Based on table complexity (per Microsoft guidance):

| Table Type | Batch Size | Max Threads |
|------------|-----------|-------------|
| OOB (Account, Contact, Lead) | 200-300 | 30 |
| Simple (few lookups) | ≤10 | 50 |
| Moderately complex | ≤100 | 30 |
| Large/complex (>100 cols, >20 lookups) | 10-20 | 10-20 |

```python
def bulk_create_optimized(client, table_name, payloads, batch_size=200):
    """Create records in optimal batch size."""
    for i in range(0, len(payloads), batch_size):
        batch = payloads[i:i + batch_size]
        ids = client.create(table_name, batch)
        print(f"Created {len(ids)} records")
        yield ids
```

---

## 5. Connection Management

### Reuse Client Instance

```python
# ❌ BAD - Creates new connection each time
def process_batch():
    for batch in batches:
        client = DataverseClient(...)  # Expensive!
        client.create("account", batch)

# ✅ GOOD - Reuse connection
client = DataverseClient(...)  # Create once

def process_batch():
    for batch in batches:
        client.create("account", batch)  # Reuse
```

### Global Client Instance

```python
# singleton_client.py
from azure.identity import DefaultAzureCredential
from PowerPlatform.Dataverse.client import DataverseClient

_client = None

def get_client():
    global _client
    if _client is None:
        _client = DataverseClient(
            base_url="https://myorg.crm.dynamics.com",
            credential=DefaultAzureCredential()
        )
    return _client

# main.py
from singleton_client import get_client

client = get_client()
records = client.get("account")
```

### Connection Timeout Configuration

```python
from PowerPlatform.Dataverse.core.config import DataverseConfig

cfg = DataverseConfig()
cfg.http_timeout = 30         # Request timeout
cfg.connection_timeout = 5    # Connection timeout

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential,
    config=cfg
)
```

---

## 6. Async Operations (Future Capability)

Currently synchronous, but prepare for async:

```python
# Recommended pattern for future async support
import asyncio

async def get_accounts_async(client):
    """Pattern for future async SDK."""
    # When SDK supports async:
    # accounts = await client.get("account")
    # For now, use sync with executor
    loop = asyncio.get_event_loop()
    accounts = await loop.run_in_executor(
        None, 
        lambda: list(client.get("account"))
    )
    return accounts

# Usage
accounts = asyncio.run(get_accounts_async(client))
```

---

## 7. File Upload Optimization

### Small Files (<128 MB)

```python
# ✅ FAST - Single request
client.upload_file(
    table_name="account",
    record_id=record_id,
    column_name="document_column",
    file_path="small_file.pdf"
)
```

### Large Files (>128 MB)

```python
# ✅ OPTIMIZED - Chunked upload
client.upload_file(
    table_name="account",
    record_id=record_id,
    column_name="document_column",
    file_path="large_file.pdf",
    mode='chunk',
    if_none_match=True
)

# SDK automatically:
# 1. Splits file into 4MB chunks
# 2. Uploads chunks in parallel
# 3. Assembles on server
```

---

## 8. OData Query Optimization

### SQL Alternative (Simple Queries)

```python
# ✅ SOMETIMES FASTER - Direct SQL for SELECT only
# Limited support: single SELECT, optional WHERE/TOP/ORDER BY
records = client.get(
    "account",
    sql="SELECT accountid, name FROM account WHERE statecode = 0 ORDER BY name"
)
```

### Complex Queries

```python
# ❌ NOT SUPPORTED - JOINs, complex WHERE
sql="SELECT a.accountid, c.fullname FROM account a JOIN contact c ON a.accountid = c.parentcustomerid"

# ✅ WORKAROUND - Get accounts, then contacts for each
accounts = client.get("account", select=["accountid", "name"])
for account in accounts:
    contacts = client.get(
        "contact",
        filter=f"parentcustomerid eq '{account['accountid']}'"
    )
    process(account, contacts)
```

---

## 9. Memory Management

### Process Large Datasets Incrementally

```python
import gc

def process_large_table(client, table_name):
    """Process millions of records without memory issues."""
    
    for page in client.get(table_name, page_size=5000):
        for record in page:
            result = process_record(record)
            save_result(result)
        
        # Force garbage collection between pages
        gc.collect()
```

### DataFrame Integration with Chunking

```python
import pandas as pd

def load_to_dataframe_chunked(client, table_name, chunk_size=10000):
    """Load data to DataFrame in chunks."""
    
    dfs = []
    for page in client.get(table_name, page_size=1000):
        df_chunk = pd.DataFrame(page)
        dfs.append(df_chunk)
        
        # Combine when chunk threshold reached
        if len(dfs) >= chunk_size // 1000:
            df = pd.concat(dfs, ignore_index=True)
            process_chunk(df)
            dfs = []
    
    # Process remaining
    if dfs:
        df = pd.concat(dfs, ignore_index=True)
        process_chunk(df)
```

---

## 10. Rate Limiting Handling

SDK has minimal retry support - implement manually:

```python
import time
from PowerPlatform.Dataverse.core.errors import DataverseError

def call_with_backoff(func, max_retries=3):
    """Call function with exponential backoff for rate limits."""
    
    for attempt in range(max_retries):
        try:
            return func()
        except DataverseError as e:
            if e.status_code == 429:  # Too Many Requests
                if attempt < max_retries - 1:
                    wait_time = 2 ** attempt  # 1s, 2s, 4s
                    print(f"Rate limited. Waiting {wait_time}s...")
                    time.sleep(wait_time)
                else:
                    raise
            else:
                raise

# Usage
ids = call_with_backoff(
    lambda: client.create("account", payload)
)
```

---

## 11. Transaction Consistency (Known Limitation)

SDK doesn't have transactional guarantees:

```python
# ⚠️ If bulk operation partially fails, some records may be created

def create_with_consistency_check(client, table_name, payloads):
    """Create records and verify all succeeded."""
    
    try:
        ids = client.create(table_name, payloads)
        
        # Verify all records created
        created = client.get(
            table_name,
            filter=f"isof(Microsoft.Dynamics.CRM.{table_name})"
        )
        
        if len(ids) != count_created:
            print(f"⚠️ Only {count_created}/{len(ids)} records created")
            # Handle partial failure
    except Exception as e:
        print(f"Creation failed: {e}")
        # Check what was created
```

---

## 12. Monitoring Performance

### Log Operation Duration

```python
import time
import logging

logger = logging.getLogger("dataverse")

def monitored_operation(operation_name):
    """Decorator to monitor operation performance."""
    def decorator(func):
        def wrapper(*args, **kwargs):
            start = time.time()
            try:
                result = func(*args, **kwargs)
                duration = time.time() - start
                logger.info(f"{operation_name}: {duration:.2f}s")
                return result
            except Exception as e:
                duration = time.time() - start
                logger.error(f"{operation_name} failed after {duration:.2f}s: {e}")
                raise
        return wrapper
    return decorator

@monitored_operation("Bulk Create Accounts")
def create_accounts(client, payloads):
    return client.create("account", payloads)
```

---

## 13. Performance Checklist

| Item | Status | Notes |
|------|--------|-------|
| Reuse client instance | ☐ | Create once, reuse |
| Use select to limit columns | ☐ | Only retrieve needed data |
| Filter server-side with OData | ☐ | Don't fetch all and filter |
| Use pagination with page_size | ☐ | Process incrementally |
| Batch operations | ☐ | Use create/update for multiple |
| Tune batch size by table type | ☐ | OOB=200-300, Simple=≤10 |
| Handle rate limiting (429) | ☐ | Implement exponential backoff |
| Use chunked upload for large files | ☐ | SDK handles for >128MB |
| Monitor operation duration | ☐ | Log timing for analysis |
| Test with production-like data | ☐ | Performance varies with data volume |

---

## 14. See Also

- [Dataverse Web API Performance](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/optimize-performance-create-update)
- [OData Query Options](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/query-data-web-api)
- [SDK Working with Data](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/work-data)

---


# Dataverse SDK for Python — Real-World Use Cases & Templates

Based on official Dataverse data migration and integration patterns.

## 1. Data Migration from Legacy Systems

### Migration Architecture

```
Legacy System → Staging Database → Dataverse
    (Extract)    (Transform)        (Load)
```

### Complete Migration Example

```python
import pandas as pd
import time
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.errors import DataverseError
from azure.identity import DefaultAzureCredential

class DataMigrationPipeline:
    """Migrate data from legacy system to Dataverse."""
    
    def __init__(self, org_url: str):
        self.client = DataverseClient(
            base_url=org_url,
            credential=DefaultAzureCredential()
        )
        self.success_records = []
        self.failed_records = []
    
    def extract_from_legacy(self, legacy_db_connection, query: str):
        """Extract data from source system."""
        return pd.read_sql(query, legacy_db_connection)
    
    def transform_accounts(self, df: pd.DataFrame) -> list:
        """Transform source data to Dataverse schema."""
        payloads = []
        
        for _, row in df.iterrows():
            # Map source fields to Dataverse
            payload = {
                "name": row["company_name"][:100],  # Limit to 100 chars
                "telephone1": row["phone"],
                "websiteurl": row["website"],
                "revenue": float(row["annual_revenue"]) if row["annual_revenue"] else None,
                "numberofemployees": int(row["employees"]) if row["employees"] else None,
                # Track source ID for reconciliation
                "new_sourcecompanyid": str(row["legacy_id"]),
                "new_importsequencenumber": row["legacy_id"]
            }
            payloads.append(payload)
        
        return payloads
    
    def load_to_dataverse(self, payloads: list, batch_size: int = 200):
        """Load data to Dataverse with error tracking."""
        total = len(payloads)
        
        for i in range(0, total, batch_size):
            batch = payloads[i:i + batch_size]
            
            try:
                ids = self.client.create("account", batch)
                self.success_records.extend(ids)
                print(f"✓ Created {len(ids)} records ({len(self.success_records)}/{total})")
                
                # Prevent rate limiting
                time.sleep(0.5)
                
            except DataverseError as e:
                self.failed_records.extend(batch)
                print(f"✗ Batch failed: {e.message}")
    
    def reconcile_migration(self, df: pd.DataFrame):
        """Verify migration and track results."""
        
        # Query created records
        created_accounts = self.client.get(
            "account",
            filter="new_importsequencenumber ne null",
            select=["accountid", "new_sourcecompanyid", "new_importsequencenumber"],
            top=10000
        )
        
        created_df = pd.DataFrame(list(created_accounts))
        
        # Update source table with Dataverse IDs
        merged = df.merge(
            created_df,
            left_on="legacy_id",
            right_on="new_importsequencenumber"
        )
        
        print(f"Successfully migrated {len(merged)} accounts")
        print(f"Failed: {len(self.failed_records)} records")
        
        return {
            "total_source": len(df),
            "migrated": len(merged),
            "failed": len(self.failed_records),
            "success_rate": len(merged) / len(df) * 100
        }

# Usage
pipeline = DataMigrationPipeline("https://myorg.crm.dynamics.com")

# Extract
source_data = pipeline.extract_from_legacy(
    legacy_connection,
    "SELECT id, company_name, phone, website, annual_revenue, employees FROM companies"
)

# Transform
payloads = pipeline.transform_accounts(source_data)

# Load
pipeline.load_to_dataverse(payloads, batch_size=300)

# Reconcile
results = pipeline.reconcile_migration(source_data)
print(results)
```

---

## 2. Data Quality & Deduplication Agent

### Detect and Merge Duplicates

```python
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import DefaultAzureCredential
import difflib

class DataQualityAgent:
    """Monitor and improve data quality."""
    
    def __init__(self, org_url: str):
        self.client = DataverseClient(
            base_url=org_url,
            credential=DefaultAzureCredential()
        )
    
    def find_potential_duplicates(self, table_name: str, match_fields: list):
        """Find potential duplicate records."""
        
        records = []
        for page in self.client.get(table_name, select=match_fields, top=10000):
            records.extend(page)
        
        duplicates = []
        seen = {}
        
        for record in records:
            # Create key from match fields
            key = tuple(
                record.get(field, "").lower().strip() 
                for field in match_fields
            )
            
            if key in seen and key != ("",) * len(match_fields):
                duplicates.append({
                    "original": seen[key],
                    "duplicate": record,
                    "fields_matched": match_fields
                })
            else:
                seen[key] = record
        
        return duplicates, len(records)
    
    def merge_records(self, table_name: str, primary_id: str, duplicate_id: str, 
                     mapping: dict):
        """Merge duplicate record into primary."""
        
        # Copy data from duplicate to primary
        updates = {}
        duplicate = self.client.get(table_name, duplicate_id)
        
        for source_field, target_field in mapping.items():
            if duplicate.get(source_field) and not primary.get(target_field):
                updates[target_field] = duplicate[source_field]
        
        # Update primary
        if updates:
            self.client.update(table_name, primary_id, updates)
        
        # Delete duplicate
        self.client.delete(table_name, duplicate_id)
        
        return f"Merged {duplicate_id} into {primary_id}"
    
    def generate_quality_report(self, table_name: str) -> dict:
        """Generate data quality metrics."""
        
        records = list(self.client.get(table_name, top=10000))
        
        report = {
            "table": table_name,
            "total_records": len(records),
            "null_values": {},
            "duplicates": 0,
            "completeness_score": 0
        }
        
        # Check null values
        all_fields = set()
        for record in records:
            all_fields.update(record.keys())
        
        for field in all_fields:
            null_count = sum(1 for r in records if not r.get(field))
            completeness = (len(records) - null_count) / len(records) * 100
            
            if completeness < 100:
                report["null_values"][field] = {
                    "null_count": null_count,
                    "completeness": completeness
                }
        
        # Check duplicates
        duplicates, _ = self.find_potential_duplicates(
            table_name, 
            ["name", "emailaddress1"]
        )
        report["duplicates"] = len(duplicates)
        
        # Overall completeness
        avg_completeness = sum(
            100 - ((d["null_count"] / len(records)) * 100)
            for d in report["null_values"].values()
        ) / len(report["null_values"]) if report["null_values"] else 100
        report["completeness_score"] = avg_completeness
        
        return report

# Usage
agent = DataQualityAgent("https://myorg.crm.dynamics.com")

# Find duplicates
duplicates, total = agent.find_potential_duplicates(
    "account",
    match_fields=["name", "emailaddress1"]
)

print(f"Found {len(duplicates)} potential duplicates out of {total} accounts")

# Merge if confident
for dup in duplicates[:5]:  # Process top 5
    result = agent.merge_records(
        "account",
        primary_id=dup["original"]["accountid"],
        duplicate_id=dup["duplicate"]["accountid"],
        mapping={"telephone1": "telephone1", "websiteurl": "websiteurl"}
    )
    print(result)

# Quality report
report = agent.generate_quality_report("account")
print(f"Data Quality: {report['completeness_score']:.1f}%")
```

---

## 3. Contact & Account Enrichment

### Enrich CRM Data from External Sources

```python
import requests
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import DefaultAzureCredential

class DataEnrichmentAgent:
    """Enrich CRM records with external data."""
    
    def __init__(self, org_url: str, external_api_key: str):
        self.client = DataverseClient(
            base_url=org_url,
            credential=DefaultAzureCredential()
        )
        self.api_key = external_api_key
    
    def enrich_accounts_with_industry_data(self):
        """Enrich accounts with industry classification."""
        
        accounts = self.client.get(
            "account",
            select=["accountid", "name", "websiteurl"],
            filter="new_industrydata eq null",
            top=500
        )
        
        enriched_count = 0
        for page in accounts:
            for account in page:
                try:
                    # Call external API
                    industry = self._lookup_industry(account["name"])
                    
                    if industry:
                        self.client.update(
                            "account",
                            account["accountid"],
                            {"new_industrydata": industry}
                        )
                        enriched_count += 1
                
                except Exception as e:
                    print(f"Failed to enrich {account['name']}: {e}")
        
        return enriched_count
    
    def enrich_contacts_with_social_profiles(self):
        """Find and link social media profiles."""
        
        contacts = self.client.get(
            "contact",
            select=["contactid", "fullname", "emailaddress1"],
            filter="new_linkedinurl eq null",
            top=500
        )
        
        for page in contacts:
            for contact in page:
                try:
                    # Find social profiles
                    profiles = self._find_social_profiles(
                        contact["fullname"],
                        contact["emailaddress1"]
                    )
                    
                    if profiles:
                        self.client.update(
                            "contact",
                            contact["contactid"],
                            {
                                "new_linkedinurl": profiles.get("linkedin"),
                                "new_twitterhandle": profiles.get("twitter")
                            }
                        )
                
                except Exception as e:
                    print(f"Failed to enrich {contact['fullname']}: {e}")
    
    def _lookup_industry(self, company_name: str) -> str:
        """Call external industry API."""
        response = requests.get(
            "https://api.example.com/industry",
            params={"company": company_name},
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        
        if response.status_code == 200:
            return response.json().get("industry")
        return None
    
    def _find_social_profiles(self, name: str, email: str) -> dict:
        """Find social media profiles for person."""
        response = requests.get(
            "https://api.example.com/social",
            params={"name": name, "email": email},
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        
        if response.status_code == 200:
            return response.json()
        return {}

# Usage
enricher = DataEnrichmentAgent(
    "https://myorg.crm.dynamics.com",
    api_key="your-api-key"
)

enriched = enricher.enrich_accounts_with_industry_data()
print(f"Enriched {enriched} accounts")
```

---

## 4. Automated Report Data Export

### Export CRM Data to Excel

```python
import pandas as pd
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import DefaultAzureCredential
from datetime import datetime

class ReportExporter:
    """Export Dataverse data to reports."""
    
    def __init__(self, org_url: str):
        self.client = DataverseClient(
            base_url=org_url,
            credential=DefaultAzureCredential()
        )
    
    def export_sales_summary(self, output_file: str):
        """Export sales data for reporting."""
        
        accounts = []
        for page in self.client.get(
            "account",
            select=["accountid", "name", "revenue", "numberofemployees", 
                   "createdon", "modifiedon"],
            filter="statecode eq 0",  # Active only
            orderby=["revenue desc"],
            top=10000
        ):
            accounts.extend(page)
        
        # Opportunities
        opportunities = []
        for page in self.client.get(
            "opportunity",
            select=["opportunityid", "name", "estimatedvalue", 
                   "statuscode", "parentaccountid", "createdon"],
            top=10000
        ):
            opportunities.extend(page)
        
        # Create DataFrames
        df_accounts = pd.DataFrame(accounts)
        df_opportunities = pd.DataFrame(opportunities)
        
        # Generate report
        with pd.ExcelWriter(output_file) as writer:
            df_accounts.to_excel(writer, sheet_name="Accounts", index=False)
            df_opportunities.to_excel(writer, sheet_name="Opportunities", index=False)
            
            # Summary sheet
            summary = pd.DataFrame({
                "Metric": [
                    "Total Accounts",
                    "Total Opportunities",
                    "Total Revenue",
                    "Export Date"
                ],
                "Value": [
                    len(df_accounts),
                    len(df_opportunities),
                    df_accounts["revenue"].sum() if "revenue" in df_accounts else 0,
                    datetime.now().isoformat()
                ]
            })
            summary.to_excel(writer, sheet_name="Summary", index=False)
        
        return output_file
    
    def export_activity_log(self, days_back: int = 30) -> str:
        """Export recent activity for audit."""
        
        from_date = pd.Timestamp.now(tz='UTC') - pd.Timedelta(days=days_back)
        
        activities = []
        for page in self.client.get(
            "activitypointer",
            select=["activityid", "subject", "activitytypecode", 
                   "createdon", "ownerid"],
            filter=f"createdon gt {from_date.isoformat()}",
            orderby=["createdon desc"],
            top=10000
        ):
            activities.extend(page)
        
        df = pd.DataFrame(activities)
        output = f"activity_log_{datetime.now():%Y%m%d}.csv"
        df.to_csv(output, index=False)
        
        return output

# Usage
exporter = ReportExporter("https://myorg.crm.dynamics.com")
report_file = exporter.export_sales_summary("sales_report.xlsx")
print(f"Report saved to {report_file}")
```

---

## 5. Workflow Integration - Bulk Operations

### Process Records Based on Conditions

```python
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import DefaultAzureCredential
from enum import IntEnum

class AccountStatus(IntEnum):
    PROSPECT = 1
    ACTIVE = 2
    CLOSED = 3

class BulkWorkflow:
    """Automate bulk operations."""
    
    def __init__(self, org_url: str):
        self.client = DataverseClient(
            base_url=org_url,
            credential=DefaultAzureCredential()
        )
    
    def mark_accounts_as_inactive_if_no_activity(self, days_no_activity: int = 90):
        """Deactivate accounts with no recent activity."""
        
        from_date = f"2025-{datetime.now().month:02d}-01T00:00:00Z"
        
        inactive_accounts = self.client.get(
            "account",
            select=["accountid", "name"],
            filter=f"modifiedon lt {from_date} and statecode eq 0",
            top=5000
        )
        
        accounts_to_deactivate = []
        for page in inactive_accounts:
            accounts_to_deactivate.extend([a["accountid"] for a in page])
        
        # Bulk update
        if accounts_to_deactivate:
            self.client.update(
                "account",
                accounts_to_deactivate,
                {"statecode": AccountStatus.CLOSED}
            )
            print(f"Deactivated {len(accounts_to_deactivate)} inactive accounts")
    
    def update_opportunity_status_based_on_amount(self):
        """Update opportunity stage based on estimated value."""
        
        opportunities = self.client.get(
            "opportunity",
            select=["opportunityid", "estimatedvalue"],
            filter="statuscode ne 7",  # Not closed
            top=5000
        )
        
        updates = []
        ids = []
        
        for page in opportunities:
            for opp in page:
                value = opp.get("estimatedvalue", 0)
                
                # Determine stage
                if value < 10000:
                    stage = 1  # Qualification
                elif value < 50000:
                    stage = 2  # Proposal
                else:
                    stage = 3  # Proposal Review
                
                updates.append({"stageid": stage})
                ids.append(opp["opportunityid"])
        
        # Bulk update
        if ids:
            self.client.update("opportunity", ids, updates)
            print(f"Updated {len(ids)} opportunities")

# Usage
workflow = BulkWorkflow("https://myorg.crm.dynamics.com")
workflow.mark_accounts_as_inactive_if_no_activity(days_no_activity=90)
workflow.update_opportunity_status_based_on_amount()
```

---

## 6. Scheduled Job Template

### Azure Function for Scheduled Operations

```python
# scheduled_migration_job.py
import azure.functions as func
from datetime import datetime
from DataMigrationPipeline import DataMigrationPipeline
import logging

def main(timer: func.TimerRequest) -> None:
    """Run migration job on schedule (e.g., daily)."""
    
    if timer.past_due:
        logging.info('The timer is past due!')
    
    try:
        logging.info(f'Migration job started at {datetime.utcnow()}')
        
        # Run migration
        pipeline = DataMigrationPipeline("https://myorg.crm.dynamics.com")
        
        # Extract, transform, load
        source_data = pipeline.extract_from_legacy(...)
        payloads = pipeline.transform_accounts(source_data)
        pipeline.load_to_dataverse(payloads)
        
        # Get results
        results = pipeline.reconcile_migration(source_data)
        
        logging.info(f'Migration completed: {results}')
        
    except Exception as e:
        logging.error(f'Migration failed: {e}')
        raise

# function_app.py - Azure Functions setup
app = func.FunctionApp()

@app.schedule_trigger(schedule="0 0 * * *")  # Daily at midnight
def migration_job(timer: func.TimerRequest) -> None:
    main(timer)
```

---

## 7. Complete Starter Template

```python
#!/usr/bin/env python3
"""
Dataverse SDK for Python - Complete Starter Template
"""

from azure.identity import DefaultAzureCredential
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.config import DataverseConfig
from PowerPlatform.Dataverse.core.errors import DataverseError
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataverseApp:
    """Base class for Dataverse applications."""
    
    def __init__(self, org_url: str):
        self.org_url = org_url
        self.client = self._create_client()
    
    def _create_client(self) -> DataverseClient:
        """Create authenticated client."""
        cfg = DataverseConfig()
        cfg.logging_enable = False
        
        return DataverseClient(
            base_url=self.org_url,
            credential=DefaultAzureCredential(),
            config=cfg
        )
    
    def create_account(self, name: str, phone: str = None) -> str:
        """Create account record."""
        try:
            payload = {"name": name}
            if phone:
                payload["telephone1"] = phone
            
            id = self.client.create("account", payload)[0]
            logger.info(f"Created account: {id}")
            return id
        
        except DataverseError as e:
            logger.error(f"Failed to create account: {e.message}")
            raise
    
    def get_accounts(self, filter_expr: str = None, top: int = 100) -> list:
        """Get account records."""
        try:
            accounts = self.client.get(
                "account",
                filter=filter_expr,
                select=["accountid", "name", "telephone1", "createdon"],
                orderby=["createdon desc"],
                top=top
            )
            
            all_accounts = []
            for page in accounts:
                all_accounts.extend(page)
            
            logger.info(f"Retrieved {len(all_accounts)} accounts")
            return all_accounts
        
        except DataverseError as e:
            logger.error(f"Failed to get accounts: {e.message}")
            raise
    
    def update_account(self, account_id: str, **kwargs) -> None:
        """Update account record."""
        try:
            self.client.update("account", account_id, kwargs)
            logger.info(f"Updated account: {account_id}")
        
        except DataverseError as e:
            logger.error(f"Failed to update account: {e.message}")
            raise

if __name__ == "__main__":
    # Usage
    app = DataverseApp("https://myorg.crm.dynamics.com")
    
    # Create
    account_id = app.create_account("Acme Inc", "555-0100")
    
    # Get
    accounts = app.get_accounts(filter_expr="statecode eq 0", top=50)
    print(f"Found {len(accounts)} active accounts")
    
    # Update
    app.update_account(account_id, telephone1="555-0199")
```

---

## 8. See Also

- [Dataverse Data Migration](https://learn.microsoft.com/en-us/power-platform/architecture/key-concepts/data-migration/workflow-complex-data-migration)
- [Working with Data (SDK)](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/work-data)
- [SDK Examples on GitHub](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/tree/main/examples)