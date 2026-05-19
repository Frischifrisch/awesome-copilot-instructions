---
description: 'Dataverse Python: auth, error handling, file ops, pandas integration and testing'
applyTo: '**'
---


# Dataverse SDK for Python — Authentication & Security Patterns

Based on official Microsoft Azure SDK authentication documentation and Dataverse SDK best practices.

## 1. Authentication Overview

The Dataverse SDK for Python uses Azure Identity credentials for token-based authentication. This approach follows the principle of least privilege and works across local development, cloud deployment, and on-premises environments.

### Why Token-Based Authentication?

**Advantages over connection strings**:
- Establishes specific permissions needed by your app (principle of least privilege)
- Credentials are scoped only to intended apps
- With managed identity, no secrets to store or compromise
- Works seamlessly across environments without code changes

---

## 2. Credential Types & Selection

### Interactive Browser Credential (Local Development)

**Use for**: Developer workstations during local development.

```python
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient

# Opens browser for authentication
credential = InteractiveBrowserCredential()
client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential
)

# First use prompts for sign-in; subsequent calls use cached token
records = client.get("account")
```

**When to use**:
- ✅ Interactive development and testing
- ✅ Desktop applications with UI
- ❌ Background services or scheduled jobs

---

### Default Azure Credential (Recommended for All Environments)

**Use for**: Apps that run in multiple environments (dev → test → production).

```python
from azure.identity import DefaultAzureCredential
from PowerPlatform.Dataverse.client import DataverseClient

# Attempts credentials in this order:
# 1. Environment variables (app service principal)
# 2. Azure CLI credentials (local development)
# 3. Azure PowerShell credentials (local development)
# 4. Managed identity (when running in Azure)
credential = DefaultAzureCredential()

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential
)

records = client.get("account")
```

**Advantages**:
- Single code path works everywhere
- No environment-specific logic needed
- Automatically detects available credentials
- Preferred for production apps

**Credential chain**:
1. Environment variables (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`)
2. Visual Studio Code login
3. Azure CLI (`az login`)
4. Azure PowerShell (`Connect-AzAccount`)
5. Managed identity (on Azure VMs, App Service, AKS, etc.)

---

### Client Secret Credential (Service Principal)

**Use for**: Unattended authentication (scheduled jobs, scripts, on-premises services).

```python
from azure.identity import ClientSecretCredential
from PowerPlatform.Dataverse.client import DataverseClient
import os

credential = ClientSecretCredential(
    tenant_id=os.environ["AZURE_TENANT_ID"],
    client_id=os.environ["AZURE_CLIENT_ID"],
    client_secret=os.environ["AZURE_CLIENT_SECRET"]
)

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential
)

records = client.get("account")
```

**Setup steps**:
1. Create app registration in Azure AD
2. Create client secret (keep secure!)
3. Grant Dataverse permissions to the app
4. Store credentials in environment variables or secure vault

**Security concerns**:
- ⚠️ Never hardcode credentials in source code
- ⚠️ Store secrets in Azure Key Vault or environment variables
- ⚠️ Rotate credentials regularly
- ⚠️ Use minimal required permissions

---

### Managed Identity Credential (Azure Resources)

**Use for**: Apps hosted in Azure (App Service, Azure Functions, AKS, VMs).

```python
from azure.identity import ManagedIdentityCredential
from PowerPlatform.Dataverse.client import DataverseClient

# No secrets needed - Azure manages identity
credential = ManagedIdentityCredential()

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential
)

records = client.get("account")
```

**Benefits**:
- ✅ No secrets to manage
- ✅ Automatic token refresh
- ✅ Highly secure
- ✅ Built-in to Azure services

**Setup**:
1. Enable managed identity on Azure resource (App Service, VM, etc.)
2. Grant Dataverse permissions to the managed identity
3. Code automatically uses the identity

---

## 3. Environment-Specific Configuration

### Local Development

```python
# .env file (git-ignored)
DATAVERSE_URL=https://myorg-dev.crm.dynamics.com

# Python code
import os
from azure.identity import DefaultAzureCredential
from PowerPlatform.Dataverse.client import DataverseClient

# Uses your Azure CLI credentials
credential = DefaultAzureCredential()
client = DataverseClient(
    base_url=os.environ["DATAVERSE_URL"],
    credential=credential
)
```

**Setup**: `az login` with your developer account

---

### Azure App Service / Azure Functions

```python
from azure.identity import ManagedIdentityCredential
from PowerPlatform.Dataverse.client import DataverseClient

# Automatically uses managed identity
credential = ManagedIdentityCredential()
client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential
)
```

**Setup**: Enable managed identity in App Service, grant permissions in Dataverse

---

### On-Premises / Third-Party Hosting

```python
import os
from azure.identity import ClientSecretCredential
from PowerPlatform.Dataverse.client import DataverseClient

credential = ClientSecretCredential(
    tenant_id=os.environ["AZURE_TENANT_ID"],
    client_id=os.environ["AZURE_CLIENT_ID"],
    client_secret=os.environ["AZURE_CLIENT_SECRET"]
)

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential
)
```

**Setup**: Create service principal, store credentials securely, grant Dataverse permissions

---

## 4. Client Configuration & Connection Settings

### Basic Configuration

```python
from PowerPlatform.Dataverse.core.config import DataverseConfig
from azure.identity import DefaultAzureCredential
from PowerPlatform.Dataverse.client import DataverseClient

cfg = DataverseConfig()
cfg.logging_enable = True  # Enable detailed logging

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=DefaultAzureCredential(),
    config=cfg
)
```

### HTTP Tuning

```python
from PowerPlatform.Dataverse.core.config import DataverseConfig

cfg = DataverseConfig()

# Timeout settings
cfg.http_timeout = 30          # Request timeout in seconds

# Retry configuration
cfg.http_retries = 3           # Number of retry attempts
cfg.http_backoff = 1           # Initial backoff in seconds

# Connection reuse
cfg.connection_timeout = 5     # Connection timeout

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=credential,
    config=cfg
)
```

---

## 5. Security Best Practices

### 1. Never Hardcode Credentials

```python
# ❌ BAD - Don't do this!
credential = ClientSecretCredential(
    tenant_id="your-tenant-id",
    client_id="your-client-id",
    client_secret="your-secret-key"  # EXPOSED!
)

# ✅ GOOD - Use environment variables
import os
credential = ClientSecretCredential(
    tenant_id=os.environ["AZURE_TENANT_ID"],
    client_id=os.environ["AZURE_CLIENT_ID"],
    client_secret=os.environ["AZURE_CLIENT_SECRET"]
)
```

### 2. Store Secrets Securely

**Development**:
```bash
# .env file (git-ignored)
AZURE_TENANT_ID=your-tenant-id
AZURE_CLIENT_ID=your-client-id
AZURE_CLIENT_SECRET=your-secret-key
```

**Production**:
```python
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

# Retrieve secrets from Azure Key Vault
credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://mykeyvault.vault.azure.net",
    credential=credential
)

secret = client.get_secret("dataverse-client-secret")
```

### 3. Implement Principle of Least Privilege

```python
# Grant minimal permissions:
# - Only read if app only reads
# - Only specific tables if possible
# - Time-limit credentials (auto-rotation)
# - Use managed identity instead of shared secrets
```

### 4. Monitor Authentication Events

```python
import logging

logger = logging.getLogger("dataverse_auth")

try:
    client = DataverseClient(
        base_url="https://myorg.crm.dynamics.com",
        credential=credential
    )
    logger.info("Successfully authenticated to Dataverse")
except Exception as e:
    logger.error(f"Authentication failed: {e}")
    raise
```

### 5. Handle Token Expiration

```python
from azure.core.exceptions import ClientAuthenticationError
import time

def create_with_auth_retry(client, table_name, payload, max_retries=2):
    """Create record, retrying if token expired."""
    for attempt in range(max_retries):
        try:
            return client.create(table_name, payload)
        except ClientAuthenticationError:
            if attempt < max_retries - 1:
                logger.warning("Token expired, retrying...")
                time.sleep(1)
            else:
                raise
```

---

## 6. Multi-Tenant Applications

### Tenant-Aware Client

```python
from azure.identity import DefaultAzureCredential
from PowerPlatform.Dataverse.client import DataverseClient

def get_client_for_tenant(tenant_id: str) -> DataverseClient:
    """Get DataverseClient for specific tenant."""
    credential = DefaultAzureCredential()
    
    # Dataverse URL contains tenant-specific org
    base_url = f"https://{get_org_for_tenant(tenant_id)}.crm.dynamics.com"
    
    return DataverseClient(
        base_url=base_url,
        credential=credential
    )

def get_org_for_tenant(tenant_id: str) -> str:
    """Map tenant to Dataverse organization."""
    # Implementation depends on your multi-tenant strategy
    # Could be database lookup, configuration, etc.
    pass
```

---

## 7. Troubleshooting Authentication

### Error: "Access Denied" (403)

```python
try:
    client.get("account")
except DataverseError as e:
    if e.status_code == 403:
        print("User/app lacks Dataverse permissions")
        print("Ensure Dataverse security role is assigned")
```

### Error: "Invalid Credentials" (401)

```python
# Check credential source
from azure.identity import DefaultAzureCredential

try:
    cred = DefaultAzureCredential(exclude_cli_credential=False, 
                                  exclude_powershell_credential=False)
    # Force re-authentication
    import subprocess
    subprocess.run(["az", "login"])
except Exception as e:
    print(f"Authentication failed: {e}")
```

### Error: "Invalid Tenant" 

```python
# Verify tenant ID
import json
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
token = credential.get_token("https://dataverse.dynamics.com/.default")

# Decode token to verify tenant
import base64
payload = base64.b64decode(token.token.split('.')[1] + '==')
claims = json.loads(payload)
print(f"Token tenant: {claims.get('tid')}")
```

---

## 8. Credential Lifecycle

### Token Refresh

Azure Identity handles token refresh automatically:

```python
# Tokens are cached and refreshed automatically
credential = DefaultAzureCredential()

# First call acquires token
client.get("account")

# Subsequent calls reuse cached token
client.get("contact")

# If token expires, SDK automatically refreshes
```

### Session Management

```python
class DataverseSession:
    """Manages DataverseClient lifecycle."""
    
    def __init__(self, base_url: str):
        from azure.identity import DefaultAzureCredential
        
        self.client = DataverseClient(
            base_url=base_url,
            credential=DefaultAzureCredential()
        )
    
    def __enter__(self):
        return self.client
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        # Cleanup if needed
        pass

# Usage
with DataverseSession("https://myorg.crm.dynamics.com") as client:
    records = client.get("account")
```

---

## 9. Dataverse-Specific Security

### Row-Level Security (RLS)

User's Dataverse security role determines accessible records:

```python
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient

# Each user gets client with their credentials
def get_user_client(user_username: str) -> DataverseClient:
    # User must already be authenticated
    credential = InteractiveBrowserCredential()
    
    client = DataverseClient(
        base_url="https://myorg.crm.dynamics.com",
        credential=credential
    )
    
    # User only sees records they have access to
    return client
```

### Security Roles

Assign minimal required roles:
- **System Administrator**: Full access (avoid for apps)
- **Sales Manager**: Sales tables + reporting
- **Service Representative**: Service cases + knowledge
- **Custom**: Create role with specific table permissions

---

## 10. See Also

- [Azure Identity Client Library](https://learn.microsoft.com/en-us/python/api/azure-identity)
- [Authenticate to Azure Services](https://learn.microsoft.com/en-us/azure/developer/python/sdk/authentication/overview)
- [Azure Key Vault for Secrets](https://learn.microsoft.com/en-us/azure/key-vault/general/overview)
- [Dataverse Security Model](https://learn.microsoft.com/en-us/power-platform/admin/security/security-overview)

---


# Dataverse SDK for Python — Error Handling & Troubleshooting Guide

Based on official Microsoft documentation for Azure SDK error handling patterns and Dataverse SDK specifics.

## 1. DataverseError Class Overview

The Dataverse SDK for Python provides a structured exception hierarchy for robust error handling.

### DataverseError Constructor

```python
from PowerPlatform.Dataverse.core.errors import DataverseError

DataverseError(
    message: str,                          # Human-readable error message
    code: str,                             # Error category (e.g., "validation_error", "http_error")
    subcode: str | None = None,            # Optional specific error identifier
    status_code: int | None = None,        # HTTP status code (if applicable)
    details: Dict[str, Any] | None = None, # Additional diagnostic information
    source: str | None = None,             # Error source: "client" or "server"
    is_transient: bool = False             # Whether error may succeed on retry
)
```

### Key Properties

```python
try:
    client.get("account", record_id="invalid-id")
except DataverseError as e:
    print(f"Message: {e.message}")           # Human-readable message
    print(f"Code: {e.code}")                 # Error category
    print(f"Subcode: {e.subcode}")           # Specific error type
    print(f"Status Code: {e.status_code}")   # HTTP status (401, 403, 429, etc.)
    print(f"Source: {e.source}")             # "client" or "server"
    print(f"Is Transient: {e.is_transient}") # Can retry?
    print(f"Details: {e.details}")           # Additional context
    
    # Convert to dictionary for logging
    error_dict = e.to_dict()
```

---

## 2. Common Error Scenarios

### Authentication Errors (401)

**Cause**: Invalid credentials, expired tokens, or misconfigured settings.

```python
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.errors import DataverseError
from azure.identity import InteractiveBrowserCredential

try:
    # Bad credentials or expired token
    credential = InteractiveBrowserCredential()
    client = DataverseClient(
        base_url="https://invalid-org.crm.dynamics.com",
        credential=credential
    )
    records = client.get("account")
except DataverseError as e:
    if e.status_code == 401:
        print("Authentication failed. Check credentials and token expiration.")
        print(f"Details: {e.message}")
        # Don't retry - fix credentials first
    else:
        raise
```

### Authorization Errors (403)

**Cause**: User lacks permissions for the requested operation.

```python
try:
    # User doesn't have permission to read contacts
    records = client.get("contact")
except DataverseError as e:
    if e.status_code == 403:
        print("Access denied. User lacks required permissions.")
        print(f"Request ID for support: {e.details.get('request_id')}")
        # Escalate to administrator
    else:
        raise
```

### Resource Not Found (404)

**Cause**: Record, table, or resource doesn't exist.

```python
try:
    # Record doesn't exist
    record = client.get("account", record_id="00000000-0000-0000-0000-000000000000")
except DataverseError as e:
    if e.status_code == 404:
        print("Resource not found. Using default data.")
        record = {"name": "Unknown", "id": None}
    else:
        raise
```

### Rate Limiting (429)

**Cause**: Too many requests exceeding service protection limits.

**Note**: The SDK has minimal built-in retry support. Handle transient consistency issues manually.

```python
import time

def create_with_retry(client, table_name, payload, max_retries=3):
    """Create record with retry logic for rate limiting."""
    for attempt in range(max_retries):
        try:
            result = client.create(table_name, payload)
            return result
        except DataverseError as e:
            if e.status_code == 429 and e.is_transient:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Rate limited. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception(f"Failed after {max_retries} retries")
```

### Server Errors (500, 502, 503, 504)

**Cause**: Temporary service issues or infrastructure problems.

```python
try:
    result = client.create("account", {"name": "Acme"})
except DataverseError as e:
    if 500 <= e.status_code < 600:
        print(f"Server error ({e.status_code}). Service may be temporarily unavailable.")
        # Implement retry logic with exponential backoff
    else:
        raise
```

### Validation Errors (400)

**Cause**: Invalid request format, missing required fields, or business rule violations.

```python
try:
    # Missing required field or invalid data
    client.create("account", {"telephone1": "not-a-phone-number"})
except DataverseError as e:
    if e.status_code == 400:
        print(f"Validation error: {e.message}")
        if e.details:
            print(f"Details: {e.details}")
        # Log validation issues for debugging
    else:
        raise
```

---

## 3. Error Handling Best Practices

### Use Specific Exception Handling

Always catch specific exceptions before general ones:

```python
from PowerPlatform.Dataverse.core.errors import DataverseError
from azure.core.exceptions import AzureError

try:
    records = client.get("account", filter="statecode eq 0", top=100)
except DataverseError as e:
    # Handle Dataverse-specific errors
    if e.status_code == 401:
        print("Re-authenticate required")
    elif e.status_code == 404:
        print("Resource not found")
    elif e.is_transient:
        print("Transient error - may retry")
    else:
        print(f"Operation failed: {e.message}")
except AzureError as e:
    # Handle Azure SDK errors (network, auth, etc.)
    print(f"Azure error: {e}")
except Exception as e:
    # Catch-all for unexpected errors
    print(f"Unexpected error: {e}")
```

### Implement Smart Retry Logic

**Don't retry on**:
- 401 Unauthorized (authentication failures)
- 403 Forbidden (authorization failures)
- 400 Bad Request (client errors)
- 404 Not Found (unless resource should eventually appear)

**Consider retrying on**:
- 408 Request Timeout
- 429 Too Many Requests (with exponential backoff)
- 500 Internal Server Error
- 502 Bad Gateway
- 503 Service Unavailable
- 504 Gateway Timeout

```python
def should_retry(error: DataverseError) -> bool:
    """Determine if operation should be retried."""
    if not error.is_transient:
        return False
    
    retryable_codes = {408, 429, 500, 502, 503, 504}
    return error.status_code in retryable_codes

def call_with_exponential_backoff(func, *args, max_attempts=3, **kwargs):
    """Call function with exponential backoff retry."""
    for attempt in range(max_attempts):
        try:
            return func(*args, **kwargs)
        except DataverseError as e:
            if should_retry(e) and attempt < max_attempts - 1:
                wait_time = 2 ** attempt  # 1s, 2s, 4s...
                print(f"Attempt {attempt + 1} failed. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

### Extract Meaningful Error Information

```python
import json
from datetime import datetime

def log_error_for_support(error: DataverseError):
    """Log error with diagnostic information."""
    error_info = {
        "timestamp": datetime.utcnow().isoformat(),
        "error_type": type(error).__name__,
        "message": error.message,
        "code": error.code,
        "subcode": error.subcode,
        "status_code": error.status_code,
        "source": error.source,
        "is_transient": error.is_transient,
        "details": error.details
    }
    
    print(json.dumps(error_info, indent=2))
    
    # Save to log file or send to monitoring service
    return error_info
```

### Handle Bulk Operations Gracefully

```python
def bulk_create_with_error_tracking(client, table_name, payloads):
    """Create multiple records, tracking which succeed/fail."""
    results = {
        "succeeded": [],
        "failed": []
    }
    
    for idx, payload in enumerate(payloads):
        try:
            record_ids = client.create(table_name, payload)
            results["succeeded"].append({
                "payload": payload,
                "ids": record_ids
            })
        except DataverseError as e:
            results["failed"].append({
                "index": idx,
                "payload": payload,
                "error": {
                    "message": e.message,
                    "code": e.code,
                    "status": e.status_code
                }
            })
    
    return results
```

---

## 4. Enable Diagnostic Logging

### Configure Logging

```python
import logging
import sys

# Set up root logger
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('dataverse_sdk.log'),
        logging.StreamHandler(sys.stdout)
    ]
)

# Configure specific loggers
logging.getLogger('azure').setLevel(logging.DEBUG)
logging.getLogger('PowerPlatform').setLevel(logging.DEBUG)

# HTTP logging (careful with sensitive data)
logging.getLogger('azure.core.pipeline.policies.http_logging_policy').setLevel(logging.DEBUG)
```

### Enable SDK-Level Logging

```python
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.config import DataverseConfig
from azure.identity import InteractiveBrowserCredential

cfg = DataverseConfig()
cfg.logging_enable = True  # Enable detailed logging

client = DataverseClient(
    base_url="https://myorg.crm.dynamics.com",
    credential=InteractiveBrowserCredential(),
    config=cfg
)

# Now SDK will log detailed HTTP requests/responses
records = client.get("account", top=10)
```

### Parse Error Responses

```python
import json

try:
    client.create("account", invalid_payload)
except DataverseError as e:
    # Extract structured error details
    if e.details and isinstance(e.details, dict):
        error_code = e.details.get('error', {}).get('code')
        error_message = e.details.get('error', {}).get('message')
        
        print(f"Error Code: {error_code}")
        print(f"Error Message: {error_message}")
        
        # Some errors include nested details
        if 'error' in e.details and 'details' in e.details['error']:
            for detail in e.details['error']['details']:
                print(f"  - {detail.get('code')}: {detail.get('message')}")
```

---

## 5. Dataverse-Specific Error Handling

### Handle OData Query Errors

```python
try:
    # Invalid OData filter
    records = client.get(
        "account",
        filter="invalid_column eq 0"
    )
except DataverseError as e:
    if "invalid column" in e.message.lower():
        print("Check OData column names and syntax")
    else:
        print(f"Query error: {e.message}")
```

### Handle File Upload Errors

```python
try:
    client.upload_file(
        table_name="account",
        record_id=record_id,
        column_name="document_column",
        file_path="large_file.pdf"
    )
except DataverseError as e:
    if e.status_code == 413:
        print("File too large. Use chunked upload mode.")
    elif e.status_code == 400:
        print("Invalid column or file format.")
    else:
        raise
```

### Handle Table Metadata Operations

```python
try:
    # Create custom table
    table_def = {
        "SchemaName": "new_CustomTable",
        "DisplayName": "Custom Table"
    }
    client.create("EntityMetadata", table_def)
except DataverseError as e:
    if "already exists" in e.message:
        print("Table already exists")
    elif "permission" in e.message.lower():
        print("Insufficient permissions to create tables")
    else:
        raise
```

---

## 6. Monitoring and Alerting

### Wrap Client Calls with Monitoring

```python
from functools import wraps
import time

def monitor_operation(operation_name):
    """Decorator to monitor SDK operations."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start_time = time.time()
            try:
                result = func(*args, **kwargs)
                duration = time.time() - start_time
                print(f"✓ {operation_name} completed in {duration:.2f}s")
                return result
            except DataverseError as e:
                duration = time.time() - start_time
                print(f"✗ {operation_name} failed after {duration:.2f}s")
                print(f"  Error: {e.code} ({e.status_code}): {e.message}")
                raise
        return wrapper
    return decorator

@monitor_operation("Fetch Accounts")
def get_accounts(client):
    return client.get("account", top=100)

# Usage
try:
    accounts = get_accounts(client)
except DataverseError:
    print("Operation failed - check logs for details")
```

---

## 7. Common Troubleshooting Checklist

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| 401 Unauthorized | Expired token or bad credentials | Re-authenticate with valid credentials |
| 403 Forbidden | User lacks permissions | Request access from administrator |
| 404 Not Found | Record/table doesn't exist | Verify schema name and record ID |
| 429 Rate Limited | Too many requests | Implement exponential backoff retry |
| 500+ Server Error | Service issue | Retry with exponential backoff; check status page |
| 400 Bad Request | Invalid request format | Check OData syntax, field names, required fields |
| Network timeout | Connection issues | Check network, increase timeout in DataverseConfig |
| InvalidOperationException | Plugin/workflow error | Check plugin logs in Dataverse |

---

## 8. Logging Best Practices

```python
import logging
import json
from datetime import datetime

class DataverseErrorHandler:
    """Centralized error handling and logging."""
    
    def __init__(self, log_file="dataverse_errors.log"):
        self.logger = logging.getLogger("DataverseSDK")
        handler = logging.FileHandler(log_file)
        formatter = logging.Formatter(
            '%(asctime)s - %(levelname)s - %(message)s'
        )
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.ERROR)
    
    def log_error(self, error: DataverseError, context: str = ""):
        """Log error with context for debugging."""
        error_record = {
            "timestamp": datetime.utcnow().isoformat(),
            "context": context,
            "error": error.to_dict()
        }
        
        self.logger.error(json.dumps(error_record, indent=2))
    
    def is_retryable(self, error: DataverseError) -> bool:
        """Check if error should be retried."""
        return error.is_transient and error.status_code in {408, 429, 500, 502, 503, 504}

# Usage
error_handler = DataverseErrorHandler()

try:
    client.create("account", payload)
except DataverseError as e:
    error_handler.log_error(e, "create_account_batch_1")
    if error_handler.is_retryable(e):
        print("Will retry this operation")
    else:
        print("Operation failed permanently")
```

---

## 9. See Also

- [DataverseError API Reference](https://learn.microsoft.com/en-us/python/api/powerplatform-dataverse-client/powerplatform.dataverse.core.errors.dataverseerror)
- [Azure SDK Error Handling](https://learn.microsoft.com/en-us/azure/developer/python/sdk/fundamentals/errors)
- [Dataverse SDK Getting Started](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/get-started)
- [Service Protection API Limits](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/optimize-performance-create-update)

---

# Dataverse SDK for Python - File Operations & Practical Examples

## Overview
Complete guide to file upload operations, chunking strategies, and practical real-world examples using the PowerPlatform-DataverseClient-Python SDK.

---

## 1. File Upload Fundamentals

### Small File Upload (< 128 MB)
```python
from pathlib import Path
from PowerPlatform.Dataverse.client import DataverseClient

file_path = Path("document.pdf")
record_id = "account-guid"

# Single PATCH upload for small files
response = client.upload_file(
    table_name="account",
    record_id=record_id,
    file_column_name="new_documentfile",
    file_path=file_path
)

print(f"Upload successful: {response}")
```

**When to use:** Documents, images, PDFs under 128 MB

### Large File Upload with Chunking
```python
from pathlib import Path

file_path = Path("large_video.mp4")
record_id = "account-guid"

# SDK automatically handles chunking for large files
response = client.upload_file(
    table_name="account",
    record_id=record_id,
    file_column_name="new_videofile",
    file_path=file_path,
    chunk_size=4 * 1024 * 1024  # 4 MB chunks
)

print("Chunked upload complete")
```

**When to use:** Large videos, databases, archives > 128 MB

### Upload with Progress Tracking
```python
import hashlib
from pathlib import Path

def calculate_file_hash(file_path):
    """Calculate SHA-256 hash of file."""
    hash_obj = hashlib.sha256()
    with open(file_path, 'rb') as f:
        for chunk in iter(lambda: f.read(1024*1024), b''):
            hash_obj.update(chunk)
    return hash_obj.hexdigest()

def upload_with_tracking(client, table_name, record_id, column_name, file_path):
    """Upload file with validation tracking."""
    file_path = Path(file_path)
    file_size = file_path.stat().st_size
    
    print(f"Starting upload: {file_path.name} ({file_size / 1024 / 1024:.2f} MB)")
    
    # Calculate hash before upload
    original_hash = calculate_file_hash(file_path)
    print(f"File hash: {original_hash}")
    
    # Perform upload
    response = client.upload_file(
        table_name=table_name,
        record_id=record_id,
        file_column_name=column_name,
        file_path=file_path
    )
    
    print(f"✓ Upload complete")
    return response

# Usage
upload_with_tracking(client, "account", account_id, "new_documentfile", "report.pdf")
```

---

## 2. Upload Strategies & Configuration

### Automatic Chunking Decision
```python
def upload_file_smart(client, table_name, record_id, column_name, file_path):
    """Upload with automatic strategy selection."""
    file_path = Path(file_path)
    file_size = file_path.stat().st_size
    max_single_patch = 128 * 1024 * 1024  # 128 MB
    
    if file_size <= max_single_patch:
        print(f"Using single PATCH (file < 128 MB)")
        chunk_size = None  # SDK will use single request
    else:
        print(f"Using chunked upload (file > 128 MB)")
        chunk_size = 4 * 1024 * 1024  # 4 MB chunks
    
    response = client.upload_file(
        table_name=table_name,
        record_id=record_id,
        file_column_name=column_name,
        file_path=file_path,
        chunk_size=chunk_size
    )
    
    return response

# Usage
upload_file_smart(client, "account", account_id, "new_largemedifile", "video.mp4")
```

### Batch File Uploads
```python
from pathlib import Path
from PowerPlatform.Dataverse.core.errors import HttpError

def batch_upload_files(client, table_name, record_id, files_dict):
    """
    Upload multiple files to different columns of same record.
    
    Args:
        table_name: Table name
        record_id: Record ID
        files_dict: {"column_name": "file_path", ...}
    
    Returns:
        {"success": [...], "failed": [...]}
    """
    results = {"success": [], "failed": []}
    
    for column_name, file_path in files_dict.items():
        try:
            print(f"Uploading {Path(file_path).name} to {column_name}...")
            response = client.upload_file(
                table_name=table_name,
                record_id=record_id,
                file_column_name=column_name,
                file_path=file_path
            )
            results["success"].append({
                "column": column_name,
                "file": Path(file_path).name,
                "response": response
            })
            print(f"  ✓ Uploaded successfully")
        except HttpError as e:
            results["failed"].append({
                "column": column_name,
                "file": Path(file_path).name,
                "error": str(e)
            })
            print(f"  ❌ Upload failed: {e}")
    
    return results

# Usage
files = {
    "new_contractfile": "contract.pdf",
    "new_specfile": "specification.docx",
    "new_designfile": "design.png"
}
results = batch_upload_files(client, "account", account_id, files)
print(f"Success: {len(results['success'])}, Failed: {len(results['failed'])}")
```

### Resume Failed Uploads
```python
from pathlib import Path
import time
from PowerPlatform.Dataverse.core.errors import HttpError

def upload_with_retry(client, table_name, record_id, column_name, file_path, max_retries=3):
    """Upload with exponential backoff retry logic."""
    file_path = Path(file_path)
    
    for attempt in range(max_retries):
        try:
            print(f"Upload attempt {attempt + 1}/{max_retries}: {file_path.name}")
            response = client.upload_file(
                table_name=table_name,
                record_id=record_id,
                file_column_name=column_name,
                file_path=file_path,
                chunk_size=4 * 1024 * 1024
            )
            print(f"✓ Upload successful")
            return response
        except HttpError as e:
            if attempt == max_retries - 1:
                print(f"❌ Upload failed after {max_retries} attempts")
                raise
            
            # Exponential backoff: 1s, 2s, 4s
            backoff_seconds = 2 ** attempt
            print(f"⚠ Upload failed. Retrying in {backoff_seconds}s...")
            time.sleep(backoff_seconds)

# Usage
upload_with_retry(client, "account", account_id, "new_documentfile", "contract.pdf")
```

---

## 3. Real-World Examples

### Example 1: Customer Document Management System

```python
from pathlib import Path
from datetime import datetime
from enum import IntEnum
from PowerPlatform.Dataverse.client import DataverseClient
from azure.identity import ClientSecretCredential

class DocumentType(IntEnum):
    CONTRACT = 1
    INVOICE = 2
    SPECIFICATION = 3
    OTHER = 4

# Setup
credential = ClientSecretCredential(
    tenant_id="tenant-id",
    client_id="client-id",
    client_secret="client-secret"
)
client = DataverseClient("https://yourorg.crm.dynamics.com", credential)

def upload_customer_document(customer_id, doc_path, doc_type):
    """Upload document for customer."""
    doc_path = Path(doc_path)
    
    # Create document record
    doc_record = {
        "new_documentname": doc_path.stem,
        "new_documenttype": doc_type,
        "new_customerid": customer_id,
        "new_uploadeddate": datetime.now().isoformat(),
        "new_filesize": doc_path.stat().st_size
    }
    
    doc_ids = client.create("new_customerdocument", doc_record)
    doc_id = doc_ids[0]
    
    # Upload file
    print(f"Uploading {doc_path.name}...")
    client.upload_file(
        table_name="new_customerdocument",
        record_id=doc_id,
        file_column_name="new_documentfile",
        file_path=doc_path
    )
    
    print(f"✓ Document uploaded and linked to customer")
    return doc_id

# Usage
customer_id = "customer-guid-here"
doc_id = upload_customer_document(
    customer_id,
    "contract.pdf",
    DocumentType.CONTRACT
)

# Query uploaded documents
docs = client.get(
    "new_customerdocument",
    filter=f"new_customerid eq '{customer_id}'",
    select=["new_documentname", "new_documenttype", "new_uploadeddate"]
)

for page in docs:
    for doc in page:
        print(f"- {doc['new_documentname']} ({doc['new_uploadeddate']})")
```

### Example 2: Media Gallery with Thumbnails

```python
from pathlib import Path
from enum import IntEnum
from PowerPlatform.Dataverse.client import DataverseClient

class MediaType(IntEnum):
    PHOTO = 1
    VIDEO = 2
    DOCUMENT = 3

def create_media_gallery(client, gallery_name, media_files):
    """
    Create media gallery with multiple files.
    
    Args:
        gallery_name: Gallery name
        media_files: [{"file": path, "type": MediaType, "description": text}, ...]
    """
    # Create gallery record
    gallery_ids = client.create("new_mediagallery", {
        "new_galleryname": gallery_name,
        "new_createddate": datetime.now().isoformat()
    })
    gallery_id = gallery_ids[0]
    
    # Create and upload media items
    for media_info in media_files:
        file_path = Path(media_info["file"])
        
        # Create media item record
        item_ids = client.create("new_mediaitem", {
            "new_itemname": file_path.stem,
            "new_mediatype": media_info["type"],
            "new_description": media_info.get("description", ""),
            "new_galleryid": gallery_id,
            "new_filesize": file_path.stat().st_size
        })
        item_id = item_ids[0]
        
        # Upload media file
        print(f"Uploading {file_path.name}...")
        client.upload_file(
            table_name="new_mediaitem",
            record_id=item_id,
            file_column_name="new_mediafile",
            file_path=file_path
        )
        print(f"  ✓ {file_path.name}")
    
    return gallery_id

# Usage
media_files = [
    {"file": "photo1.jpg", "type": MediaType.PHOTO, "description": "Product shot 1"},
    {"file": "photo2.jpg", "type": MediaType.PHOTO, "description": "Product shot 2"},
    {"file": "demo.mp4", "type": MediaType.VIDEO, "description": "Product demo video"},
    {"file": "manual.pdf", "type": MediaType.DOCUMENT, "description": "User manual"}
]

gallery_id = create_media_gallery(client, "Q4 Product Launch", media_files)
print(f"Created gallery: {gallery_id}")
```

### Example 3: Backup & Archival System

```python
from pathlib import Path
from datetime import datetime, timedelta
from PowerPlatform.Dataverse.client import DataverseClient
from PowerPlatform.Dataverse.core.errors import DataverseError
import json

def backup_table_data(client, table_name, output_dir):
    """
    Backup table data to JSON files and create archive record.
    """
    output_dir = Path(output_dir)
    output_dir.mkdir(exist_ok=True)
    
    backup_time = datetime.now()
    backup_file = output_dir / f"{table_name}_{backup_time.strftime('%Y%m%d_%H%M%S')}.json"
    
    print(f"Backing up {table_name}...")
    
    # Retrieve all records
    all_records = []
    for page in client.get(table_name, top=5000):
        all_records.extend(page)
    
    # Write to JSON
    with open(backup_file, 'w') as f:
        json.dump(all_records, f, indent=2, default=str)
    
    print(f"  ✓ Exported {len(all_records)} records")
    
    # Create backup record in Dataverse
    backup_ids = client.create("new_backuprecord", {
        "new_tablename": table_name,
        "new_recordcount": len(all_records),
        "new_backupdate": backup_time.isoformat(),
        "new_status": 1  # Completed
    })
    backup_id = backup_ids[0]
    
    # Upload backup file
    print(f"Uploading backup file...")
    client.upload_file(
        table_name="new_backuprecord",
        record_id=backup_id,
        file_column_name="new_backupfile",
        file_path=backup_file
    )
    
    return backup_id

# Usage
backup_id = backup_table_data(client, "account", "backups")
print(f"Backup created: {backup_id}")
```

### Example 4: Automated Report Generation & Storage

```python
from pathlib import Path
from datetime import datetime
from enum import IntEnum
from PowerPlatform.Dataverse.client import DataverseClient
import json

class ReportStatus(IntEnum):
    PENDING = 1
    PROCESSING = 2
    COMPLETED = 3
    FAILED = 4

def generate_and_store_report(client, report_type, data):
    """
    Generate report from data and store in Dataverse.
    """
    report_time = datetime.now()
    
    # Generate report file (simulated)
    report_file = Path(f"report_{report_type}_{report_time.strftime('%Y%m%d_%H%M%S')}.json")
    with open(report_file, 'w') as f:
        json.dump(data, f, indent=2)
    
    # Create report record
    report_ids = client.create("new_report", {
        "new_reportname": f"{report_type} Report",
        "new_reporttype": report_type,
        "new_generateddate": report_time.isoformat(),
        "new_status": ReportStatus.PROCESSING,
        "new_recordcount": len(data.get("records", []))
    })
    report_id = report_ids[0]
    
    try:
        # Upload report file
        print(f"Uploading report: {report_file.name}")
        client.upload_file(
            table_name="new_report",
            record_id=report_id,
            file_column_name="new_reportfile",
            file_path=report_file
        )
        
        # Update status to completed
        client.update("new_report", report_id, {
            "new_status": ReportStatus.COMPLETED
        })
        
        print(f"✓ Report stored successfully")
        return report_id
        
    except Exception as e:
        print(f"❌ Report generation failed: {e}")
        client.update("new_report", report_id, {
            "new_status": ReportStatus.FAILED,
            "new_errormessage": str(e)
        })
        raise
    finally:
        # Clean up temp file
        report_file.unlink(missing_ok=True)

# Usage
sales_data = {
    "month": "January",
    "records": [
        {"product": "A", "sales": 10000},
        {"product": "B", "sales": 15000},
        {"product": "C", "sales": 8000}
    ]
}

report_id = generate_and_store_report(client, "SALES_SUMMARY", sales_data)
```

---

## 4. File Management Best Practices

### File Size Validation
```python
from pathlib import Path

def validate_file_for_upload(file_path, max_size_mb=500):
    """Validate file before upload."""
    file_path = Path(file_path)
    
    if not file_path.exists():
        raise FileNotFoundError(f"File not found: {file_path}")
    
    file_size = file_path.stat().st_size
    max_size_bytes = max_size_mb * 1024 * 1024
    
    if file_size > max_size_bytes:
        raise ValueError(f"File too large: {file_size / 1024 / 1024:.2f} MB > {max_size_mb} MB")
    
    return file_size

# Usage
try:
    size = validate_file_for_upload("document.pdf", max_size_mb=128)
    print(f"File valid: {size / 1024 / 1024:.2f} MB")
except (FileNotFoundError, ValueError) as e:
    print(f"Validation failed: {e}")
```

### Supported File Types Validation
```python
from pathlib import Path

ALLOWED_EXTENSIONS = {'.pdf', '.docx', '.xlsx', '.jpg', '.png', '.mp4', '.zip'}

def validate_file_type(file_path):
    """Validate file extension."""
    file_path = Path(file_path)
    
    if file_path.suffix.lower() not in ALLOWED_EXTENSIONS:
        raise ValueError(f"Unsupported file type: {file_path.suffix}")
    
    return True

# Usage
try:
    validate_file_type("document.pdf")
    print("File type valid")
except ValueError as e:
    print(f"Invalid: {e}")
```

### Upload Logging & Audit Trail
```python
from pathlib import Path
from datetime import datetime
import json

def log_file_upload(table_name, record_id, file_path, status, error=None):
    """Log file upload for audit trail."""
    file_path = Path(file_path)
    
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "table": table_name,
        "record_id": record_id,
        "file_name": file_path.name,
        "file_size": file_path.stat().st_size if file_path.exists() else 0,
        "status": status,
        "error": error
    }
    
    # Append to log file
    log_file = Path("upload_audit.log")
    with open(log_file, 'a') as f:
        f.write(json.dumps(log_entry) + "\n")
    
    return log_entry

# Usage in upload wrapper
def upload_with_logging(client, table_name, record_id, column_name, file_path):
    """Upload with audit logging."""
    try:
        client.upload_file(
            table_name=table_name,
            record_id=record_id,
            file_column_name=column_name,
            file_path=file_path
        )
        log_file_upload(table_name, record_id, file_path, "SUCCESS")
    except Exception as e:
        log_file_upload(table_name, record_id, file_path, "FAILED", str(e))
        raise
```

---

## 5. Troubleshooting File Operations

### Common Issues & Solutions

#### Issue: File Upload Timeout
```python
# For very large files, increase chunk size strategically
response = client.upload_file(
    table_name="account",
    record_id=record_id,
    file_column_name="new_file",
    file_path="large_file.zip",
    chunk_size=8 * 1024 * 1024  # 8 MB chunks
)
```

#### Issue: Insufficient Disk Space
```python
import shutil
from pathlib import Path

def check_upload_space(file_path):
    """Check if system has space for file + temp buffer."""
    file_path = Path(file_path)
    file_size = file_path.stat().st_size
    
    # Get disk space
    total, used, free = shutil.disk_usage(file_path.parent)
    
    # Need file_size + 10% buffer
    required_space = file_size * 1.1
    
    if free < required_space:
        raise OSError(f"Insufficient disk space: {free / 1024 / 1024:.0f} MB free, {required_space / 1024 / 1024:.0f} MB needed")
    
    return True
```

#### Issue: File Corruption During Upload
```python
import hashlib

def verify_uploaded_file(local_path, remote_data):
    """Verify uploaded file integrity."""
    # Calculate local hash
    with open(local_path, 'rb') as f:
        local_hash = hashlib.sha256(f.read()).hexdigest()
    
    # Compare with metadata
    remote_hash = remote_data.get("new_filehash")
    
    if local_hash != remote_hash:
        raise ValueError("File corruption detected: hash mismatch")
    
    return True
```

---

## Reference
- [Official File Upload Example](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/blob/main/examples/advanced/file_upload.py)
- [File Upload Best Practices](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/file-column-data)

---

# Dataverse SDK for Python - Pandas Integration Guide

## Overview
Guide to integrating the Dataverse SDK for Python with pandas DataFrames for data science and analysis workflows. The SDK's JSON response format maps seamlessly to pandas DataFrames, enabling data scientists to work with Dataverse data using familiar data manipulation tools.

---

## 1. Introduction to PandasODataClient

### What is PandasODataClient?
`PandasODataClient` is a thin wrapper around the standard `DataverseClient` that returns data in pandas DataFrame format instead of raw JSON dictionaries. This makes it ideal for:
- Data scientists working with tabular data
- Analytics and reporting workflows
- Data exploration and cleaning
- Integration with machine learning pipelines

### Installation Requirements
```bash
# Install core dependencies
pip install PowerPlatform-Dataverse-Client
pip install azure-identity

# Install pandas for data manipulation
pip install pandas
```

### When to Use PandasODataClient
✅ **Use when you need:**
- Data exploration and analysis
- Working with tabular data
- Integration with statistical/ML libraries
- Efficient data manipulation

❌ **Use DataverseClient instead when you need:**
- Real-time CRUD operations only
- File upload operations
- Metadata operations
- Single record operations

---

## 2. Basic DataFrame Workflow

### Converting Query Results to DataFrame
```python
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient
import pandas as pd

# Setup authentication
base_url = "https://<myorg>.crm.dynamics.com"
credential = InteractiveBrowserCredential()
client = DataverseClient(base_url=base_url, credential=credential)

# Query data
pages = client.get(
    "account",
    select=["accountid", "name", "creditlimit", "telephone1"],
    filter="statecode eq 0",
    orderby=["name"]
)

# Collect all pages into one DataFrame
all_records = []
for page in pages:
    all_records.extend(page)

# Convert to DataFrame
df = pd.DataFrame(all_records)

# Display first few rows
print(df.head())
print(f"Total records: {len(df)}")
```

### Query Parameters Map to DataFrame
```python
# All query parameters return as columns in DataFrame
df = pd.DataFrame(
    client.get(
        "account",
        select=["accountid", "name", "creditlimit", "telephone1", "createdon"],
        filter="creditlimit > 50000",
        orderby=["creditlimit desc"]
    )
)

# Result is a DataFrame with columns:
# accountid | name | creditlimit | telephone1 | createdon
```

---

## 3. Data Exploration with Pandas

### Basic Exploration
```python
import pandas as pd
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient

client = DataverseClient("https://<myorg>.crm.dynamics.com", InteractiveBrowserCredential())

# Load account data
records = []
for page in client.get("account", select=["accountid", "name", "creditlimit", "industrycode"]):
    records.extend(page)

df = pd.DataFrame(records)

# Explore the data
print(df.shape)           # (1000, 4)
print(df.dtypes)          # Data types
print(df.describe())      # Statistical summary
print(df.info())          # Column info and null counts
print(df.head(10))        # First 10 rows
```

### Filtering and Selecting
```python
# Filter rows by condition
high_value = df[df['creditlimit'] > 100000]

# Select specific columns
names_limits = df[['name', 'creditlimit']]

# Multiple conditions
filtered = df[(df['creditlimit'] > 50000) & (df['industrycode'] == 1)]

# Value counts
print(df['industrycode'].value_counts())
```

### Sorting and Grouping
```python
# Sort by column
sorted_df = df.sort_values('creditlimit', ascending=False)

# Group by and aggregate
by_industry = df.groupby('industrycode').agg({
    'creditlimit': ['mean', 'sum', 'count'],
    'name': 'count'
})

# Group statistics
print(df.groupby('industrycode')['creditlimit'].describe())
```

### Data Cleaning
```python
# Handle missing values
df_clean = df.dropna()                    # Remove rows with NaN
df_filled = df.fillna(0)                  # Fill NaN with 0
df_ffill = df.fillna(method='ffill')      # Forward fill

# Check for duplicates
duplicates = df[df.duplicated(['name'])]
df_unique = df.drop_duplicates()

# Data type conversion
df['creditlimit'] = pd.to_numeric(df['creditlimit'])
df['createdon'] = pd.to_datetime(df['createdon'])
```

---

## 4. Data Analysis Patterns

### Aggregation and Summarization
```python
# Create summary report
summary = df.groupby('industrycode').agg({
    'accountid': 'count',
    'creditlimit': ['mean', 'min', 'max', 'sum'],
    'name': lambda x: ', '.join(x.head(3))  # Sample names
}).round(2)

print(summary)
```

### Time-Series Analysis
```python
# Convert to datetime
df['createdon'] = pd.to_datetime(df['createdon'])

# Resample to monthly
monthly = df.set_index('createdon').resample('M').size()

# Extract date components
df['year'] = df['createdon'].dt.year
df['month'] = df['createdon'].dt.month
df['day_of_week'] = df['createdon'].dt.day_name()
```

### Join and Merge Operations
```python
# Load two related tables
accounts = pd.DataFrame(client.get("account", select=["accountid", "name"]))
contacts = pd.DataFrame(client.get("contact", select=["contactid", "parentcustomerid", "fullname"]))

# Merge on relationship
merged = accounts.merge(
    contacts,
    left_on='accountid',
    right_on='parentcustomerid',
    how='left'
)

print(merged.head())
```

### Statistical Analysis
```python
# Correlation matrix
correlation = df[['creditlimit', 'industrycode']].corr()

# Distribution analysis
print(df['creditlimit'].describe())
print(df['creditlimit'].skew())
print(df['creditlimit'].kurtosis())

# Percentiles
print(df['creditlimit'].quantile([0.25, 0.5, 0.75]))
```

---

## 5. Pivot Tables and Reports

### Creating Pivot Tables
```python
# Pivot table by industry and status
pivot = pd.pivot_table(
    df,
    values='creditlimit',
    index='industrycode',
    columns='statecode',
    aggfunc=['sum', 'mean', 'count']
)

print(pivot)
```

### Generating Reports
```python
# Sales report by industry
industry_report = df.groupby('industrycode').agg({
    'accountid': 'count',
    'creditlimit': 'sum',
    'name': 'first'
}).rename(columns={
    'accountid': 'Account Count',
    'creditlimit': 'Total Credit Limit',
    'name': 'Sample Account'
})

# Export to CSV
industry_report.to_csv('industry_report.csv')

# Export to Excel
industry_report.to_excel('industry_report.xlsx')
```

---

## 6. Data Visualization

### Matplotlib Integration
```python
import matplotlib.pyplot as plt

# Create visualizations
fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# Histogram
df['creditlimit'].hist(bins=30, ax=axes[0, 0])
axes[0, 0].set_title('Credit Limit Distribution')

# Bar chart
df['industrycode'].value_counts().plot(kind='bar', ax=axes[0, 1])
axes[0, 1].set_title('Accounts by Industry')

# Box plot
df.boxplot(column='creditlimit', by='industrycode', ax=axes[1, 0])
axes[1, 0].set_title('Credit Limit by Industry')

# Scatter plot
df.plot.scatter(x='creditlimit', y='industrycode', ax=axes[1, 1])
axes[1, 1].set_title('Credit Limit vs Industry')

plt.tight_layout()
plt.show()
```

### Seaborn Integration
```python
import seaborn as sns

# Correlation heatmap
plt.figure(figsize=(8, 6))
sns.heatmap(df[['creditlimit', 'industrycode']].corr(), annot=True)
plt.title('Correlation Matrix')
plt.show()

# Distribution plot
sns.distplot(df['creditlimit'], kde=True)
plt.title('Credit Limit Distribution')
plt.show()
```

---

## 7. Machine Learning Integration

### Preparing Data for ML
```python
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split

# Load and prepare data
records = []
for page in client.get("account", select=["accountid", "creditlimit", "industrycode", "statecode"]):
    records.extend(page)

df = pd.DataFrame(records)

# Feature engineering
df['log_creditlimit'] = np.log1p(df['creditlimit'])
df['industry_cat'] = pd.Categorical(df['industrycode']).codes

# Split features and target
X = df[['industrycode', 'log_creditlimit']]
y = df['statecode']

# Train-test split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

print(f"Training set: {len(X_train)}, Test set: {len(X_test)}")
```

### Building a Classification Model
```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report

# Train model
model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)

# Evaluate
y_pred = model.predict(X_test)
print(classification_report(y_test, y_pred))

# Feature importance
importances = pd.Series(
    model.feature_importances_,
    index=X.columns
).sort_values(ascending=False)

print(importances)
```

---

## 8. Advanced DataFrame Operations

### Custom Functions
```python
# Apply function to columns
df['name_length'] = df['name'].apply(len)

# Apply function to rows
df['category'] = df.apply(
    lambda row: 'High' if row['creditlimit'] > 100000 else 'Low',
    axis=1
)

# Conditional operations
df['adjusted_limit'] = df['creditlimit'].where(
    df['statecode'] == 0,
    df['creditlimit'] * 0.5
)
```

### String Operations
```python
# String methods
df['name_upper'] = df['name'].str.upper()
df['name_starts'] = df['name'].str.startswith('A')
df['name_contains'] = df['name'].str.contains('Inc')
df['name_split'] = df['name'].str.split(',').str[0]

# Replace and substitute
df['industry'] = df['industrycode'].map({
    1: 'Retail',
    2: 'Manufacturing',
    3: 'Technology'
})
```

### Reshaping Data
```python
# Transpose
transposed = df.set_index('name').T

# Stack/Unstack
stacked = df.set_index(['name', 'industrycode'])['creditlimit'].unstack()

# Melt long format
melted = pd.melt(df, id_vars=['name'], var_name='metric', value_name='value')
```

---

## 9. Performance Optimization

### Efficient Data Loading
```python
# Load large datasets in chunks
all_records = []
chunk_size = 1000

for page in client.get(
    "account",
    select=["accountid", "name", "creditlimit"],
    top=10000,        # Limit total records
    page_size=chunk_size
):
    all_records.extend(page)
    if len(all_records) % 5000 == 0:
        print(f"Loaded {len(all_records)} records")

df = pd.DataFrame(all_records)
print(f"Total: {len(df)} records")
```

### Memory Optimization
```python
# Reduce memory usage
# Use categorical for repeated values
df['industrycode'] = df['industrycode'].astype('category')

# Use appropriate numeric types
df['creditlimit'] = pd.to_numeric(df['creditlimit'], downcast='float')

# Delete columns no longer needed
df = df.drop(columns=['unused_col1', 'unused_col2'])

# Check memory usage
print(df.memory_usage(deep=True).sum() / 1024**2, "MB")
```

### Query Optimization
```python
# Apply filters on server, not client
# ✅ GOOD: Filter on server
accounts = client.get(
    "account",
    filter="creditlimit > 50000",  # Server-side filter
    select=["accountid", "name", "creditlimit"]
)

# ❌ BAD: Load all, filter locally
all_accounts = client.get("account")  # Loads everything
filtered = [a for a in all_accounts if a['creditlimit'] > 50000]  # Client-side
```

---

## 10. Complete Example: Sales Analytics

```python
import pandas as pd
import numpy as np
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient

# Setup
client = DataverseClient(
    "https://<myorg>.crm.dynamics.com",
    InteractiveBrowserCredential()
)

# Load data
print("Loading account data...")
records = []
for page in client.get(
    "account",
    select=["accountid", "name", "creditlimit", "industrycode", "statecode", "createdon"],
    orderby=["createdon"]
):
    records.extend(page)

df = pd.DataFrame(records)
df['createdon'] = pd.to_datetime(df['createdon'])

# Data cleaning
df = df.dropna()

# Feature engineering
df['year'] = df['createdon'].dt.year
df['month'] = df['createdon'].dt.month
df['year_month'] = df['createdon'].dt.to_period('M')

# Analysis
print("\n=== ACCOUNT OVERVIEW ===")
print(f"Total accounts: {len(df)}")
print(f"Total credit limit: ${df['creditlimit'].sum():,.2f}")
print(f"Average credit limit: ${df['creditlimit'].mean():,.2f}")

print("\n=== BY INDUSTRY ===")
industry_summary = df.groupby('industrycode').agg({
    'accountid': 'count',
    'creditlimit': ['sum', 'mean']
}).round(2)
print(industry_summary)

print("\n=== BY STATUS ===")
status_summary = df.groupby('statecode').agg({
    'accountid': 'count',
    'creditlimit': 'sum'
})
print(status_summary)

# Export report
print("\n=== EXPORTING REPORT ===")
industry_summary.to_csv('industry_analysis.csv')
print("Report saved to industry_analysis.csv")
```

---

## 11. Known Limitations

- `PandasODataClient` currently requires manual DataFrame creation from query results
- Very large DataFrames (millions of rows) may experience memory constraints
- Pandas operations are client-side; server-side aggregation is more efficient for large datasets
- File operations require standard `DataverseClient`, not pandas wrapper

---

## 12. Related Resources

- [Pandas Documentation](https://pandas.pydata.org/docs/)
- [Official Example: quickstart_pandas.py](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/blob/main/examples/quickstart_pandas.py)
- [SDK for Python README](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/blob/main/README.md)
- [Microsoft Learn: Working with data](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/sdk-python/work-data)

---


# Dataverse SDK for Python — Testing & Debugging Strategies

Based on official Azure Functions and pytest testing patterns.

## 1. Testing Overview

### Testing Pyramid for Dataverse SDK

```
         Integration Tests  <- Test with real Dataverse
              /\
             /  \
            /Unit Tests (Mocked)\
           /____________________\
          < Framework Tests
```

---

## 2. Unit Testing with Mocking

### Setup Test Environment

```bash
# Install test dependencies
pip install pytest pytest-cov unittest-mock
```

### Mock DataverseClient

```python
# tests/test_operations.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from PowerPlatform.Dataverse.client import DataverseClient

@pytest.fixture
def mock_client():
    """Provide mocked DataverseClient."""
    client = Mock(spec=DataverseClient)
    return client

def test_create_account(mock_client):
    """Test account creation with mocked client."""
    
    # Setup mock response
    mock_client.create.return_value = ["id-123"]
    
    # Call function
    from my_app import create_account
    result = create_account(mock_client, {"name": "Acme"})
    
    # Verify
    assert result == "id-123"
    mock_client.create.assert_called_once_with("account", {"name": "Acme"})

def test_create_account_error(mock_client):
    """Test error handling in account creation."""
    from PowerPlatform.Dataverse.core.errors import DataverseError
    
    # Setup mock to raise error
    mock_client.create.side_effect = DataverseError(
        message="Account exists",
        code="validation_error",
        status_code=400
    )
    
    # Verify error is raised
    from my_app import create_account
    with pytest.raises(DataverseError):
        create_account(mock_client, {"name": "Acme"})
```

### Test Data Structures

```python
# tests/fixtures.py
import pytest

@pytest.fixture
def sample_account():
    """Sample account record for testing."""
    return {
        "accountid": "id-123",
        "name": "Acme Inc",
        "telephone1": "555-0100",
        "statecode": 0,
        "createdon": "2025-01-01T00:00:00Z"
    }

@pytest.fixture
def sample_accounts(sample_account):
    """Multiple sample accounts."""
    return [
        sample_account,
        {**sample_account, "accountid": "id-124", "name": "Fabrikam"},
        {**sample_account, "accountid": "id-125", "name": "Contoso"},
    ]

# Usage in tests
def test_process_accounts(mock_client, sample_accounts):
    mock_client.get.return_value = iter([sample_accounts])
    # Test processing
```

---

## 3. Mocking Common Patterns

### Mock Get with Pagination

```python
def test_pagination(mock_client, sample_accounts):
    """Test handling paginated results."""
    
    # Mock returns generator with pages
    mock_client.get.return_value = iter([
        sample_accounts[:2],  # Page 1
        sample_accounts[2:]   # Page 2
    ])
    
    from my_app import process_all_accounts
    result = process_all_accounts(mock_client)
    
    assert len(result) == 3  # All pages processed
```

### Mock Bulk Operations

```python
def test_bulk_create(mock_client):
    """Test bulk account creation."""
    
    payloads = [
        {"name": "Account 1"},
        {"name": "Account 2"},
    ]
    
    # Mock returns list of IDs
    mock_client.create.return_value = ["id-1", "id-2"]
    
    from my_app import create_accounts
    ids = create_accounts(mock_client, payloads)
    
    assert len(ids) == 2
    mock_client.create.assert_called_once_with("account", payloads)
```

### Mock Errors

```python
def test_rate_limiting_retry(mock_client):
    """Test retry logic on rate limiting."""
    from PowerPlatform.Dataverse.core.errors import DataverseError
    
    # Mock fails then succeeds
    error = DataverseError(
        message="Too many requests",
        code="http_error",
        status_code=429,
        is_transient=True
    )
    mock_client.create.side_effect = [error, ["id-123"]]
    
    from my_app import create_with_retry
    result = create_with_retry(mock_client, "account", {})
    
    assert result == "id-123"
    assert mock_client.create.call_count == 2  # Retried
```

---

## 4. Integration Testing

### Local Development Testing

```python
# tests/test_integration.py
import pytest
from azure.identity import InteractiveBrowserCredential
from PowerPlatform.Dataverse.client import DataverseClient

@pytest.fixture
def dataverse_client():
    """Real client for integration testing."""
    client = DataverseClient(
        base_url="https://myorg-dev.crm.dynamics.com",
        credential=InteractiveBrowserCredential()
    )
    return client

@pytest.mark.integration
def test_create_and_retrieve_account(dataverse_client):
    """Test creating and retrieving account (against real Dataverse)."""
    
    # Create
    account_id = dataverse_client.create("account", {
        "name": "Test Account"
    })[0]
    
    # Retrieve
    account = dataverse_client.get("account", account_id)
    
    # Verify
    assert account["name"] == "Test Account"
    
    # Cleanup
    dataverse_client.delete("account", account_id)
```

### Test Isolation

```python
# tests/conftest.py
import pytest

@pytest.fixture(scope="function")
def test_account(dataverse_client):
    """Create test account, cleanup after test."""
    
    account_id = dataverse_client.create("account", {
        "name": "Test Account"
    })[0]
    
    yield account_id
    
    # Cleanup
    try:
        dataverse_client.delete("account", account_id)
    except:
        pass  # Already deleted

# Usage
def test_update_account(dataverse_client, test_account):
    """Test updating account."""
    dataverse_client.update("account", test_account, {"telephone1": "555-0100"})
    
    account = dataverse_client.get("account", test_account)
    assert account["telephone1"] == "555-0100"
```

---

## 5. Pytest Configuration

### pytest.ini

```ini
[pytest]
# Skip integration tests by default
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

markers =
    integration: marks tests as integration (run with -m integration)
    slow: marks tests as slow
    unit: marks tests as unit tests
```

### Run Tests

```bash
# Unit tests only
pytest

# Unit + integration
pytest -m "unit or integration"

# Integration only
pytest -m integration

# With coverage
pytest --cov=my_app tests/

# Specific test
pytest tests/test_operations.py::test_create_account
```

---

## 6. Coverage Analysis

### Generate Coverage Report

```bash
# Run tests with coverage
pytest --cov=my_app --cov-report=html tests/

# View coverage
open htmlcov/index.html  # macOS
start htmlcov/index.html  # Windows
```

### Coverage Configuration (.coveragerc)

```ini
[run]
branch = True
source = my_app

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:

[html]
directory = htmlcov
```

---

## 7. Debugging with print/logging

### Enable Debug Logging

```python
import logging
import sys

# Configure logging
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler('debug.log')
    ]
)

# Enable SDK logging
logging.getLogger('PowerPlatform').setLevel(logging.DEBUG)
logging.getLogger('azure').setLevel(logging.DEBUG)

# In test
def test_with_logging(mock_client):
    logger = logging.getLogger(__name__)
    logger.debug("Starting test")
    
    result = my_function(mock_client)
    
    logger.debug(f"Result: {result}")
```

### Pytest Capturing Output

```bash
# Show print/logging output in tests
pytest -s tests/

# Capture and show on failure only
pytest --tb=short tests/
```

---

## 8. Performance Testing

### Measure Operation Duration

```python
import pytest
import time

def test_bulk_create_performance(dataverse_client):
    """Test bulk create performance."""
    
    payloads = [{"name": f"Account {i}"} for i in range(1000)]
    
    start = time.time()
    ids = dataverse_client.create("account", payloads)
    duration = time.time() - start
    
    assert len(ids) == 1000
    assert duration < 10  # Should complete in under 10 seconds
    
    print(f"Created 1000 records in {duration:.2f}s ({1000/duration:.0f} records/s)")
```

### Pytest Benchmark Plugin

```bash
pip install pytest-benchmark
```

```python
def test_query_performance(benchmark, dataverse_client):
    """Benchmark query performance."""
    
    def get_accounts():
        return list(dataverse_client.get("account", top=100))
    
    result = benchmark(get_accounts)
    assert len(result) <= 100
```

---

## 9. Common Testing Patterns

### Testing Retry Logic

```python
def test_retry_on_transient_error(mock_client):
    """Test retry on transient error."""
    from PowerPlatform.Dataverse.core.errors import DataverseError
    
    error = DataverseError(
        message="Timeout",
        code="http_error",
        status_code=408,
        is_transient=True
    )
    
    # Fail then succeed
    mock_client.create.side_effect = [error, ["id-123"]]
    
    from my_app import create_with_retry
    result = create_with_retry(mock_client, "account", {})
    
    assert result == "id-123"
```

### Testing Filter Building

```python
def test_filter_builder():
    """Test OData filter generation."""
    from my_app import build_account_filter
    
    # Test cases
    assert build_account_filter(status="active") == "statecode eq 0"
    assert build_account_filter(name="Acme") == "contains(name, 'Acme')"
    assert build_account_filter(status="active", name="Acme") \
        == "statecode eq 0 and contains(name, 'Acme')"
```

### Testing Error Handling

```python
def test_handles_missing_record(mock_client):
    """Test handling 404 errors."""
    from PowerPlatform.Dataverse.core.errors import DataverseError
    
    mock_client.get.side_effect = DataverseError(
        message="Not found",
        code="http_error",
        status_code=404
    )
    
    from my_app import get_account_safe
    result = get_account_safe(mock_client, "invalid-id")
    
    assert result is None  # Returns None instead of raising
```

---

## 10. Debugging Checklist

| Issue | Debug Steps |
|-------|-------------|
| Test fails unexpectedly | Add `-s` flag to see print output |
| Mock not called | Check method name/parameters match exactly |
| Real API failing | Check credentials, URL, permissions |
| Rate limiting in tests | Add delays or use smaller batches |
| Data not found | Verify record created and not cleaned up |
| Assertion errors | Print actual vs expected values |

---

## 11. See Also

- [Pytest Documentation](https://docs.pytest.org/)
- [unittest.mock Reference](https://docs.python.org/3/library/unittest.mock.html)
- [Azure Functions Testing](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python#unit-testing)
- [Dataverse SDK Examples](https://github.com/microsoft/PowerPlatform-DataverseClient-Python/tree/main/examples)