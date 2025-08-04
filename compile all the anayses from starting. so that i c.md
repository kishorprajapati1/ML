# Azure Cost Optimization Challenge: Billing Records Tiered Storage Solution

**Date:** Monday, August 04, 2025
**By:** Prashant Poonia

## 1. Problem Statement \& Context

- The system stores over 2 million billing records in Azure Cosmos DB.
- Each record can be up to 300 KB in size.
- The system is read-heavy, but records older than 3 months are rarely accessed.
- The database size and associated cost have grown significantly over time.
- Goal: Optimize costs while ensuring availability and minimal latency (seconds) for old records.
- Constraints:
    - No downtime or data loss during migration.
    - No changes to existing API contracts.
    - Solution must be simple and maintainable.


## 2. Initial Solution Outline

### Tiered Storage Approach:

- **Hot Data:** Keep billing data for the last 3 months in Cosmos DB (fast access, costly).
- **Cold Data:** Archive older data (records older than 3 months) into Azure Blob Storage (low cost, slower).
- **Unified Access Layer:** Introduce a logic layer that checks Cosmos DB first; if not found, fetches data from Blob Storage — keeps APIs unchanged.


### Benefits:

- Reduced Cosmos DB storage and RU consumption costs.
- Keeps data available with acceptable latency.
- Minimal changes to client-facing APIs.


## 3. Process

- **Separation of concerns:** hot data in Cosmos DB, cold data in Blob Storage.
- **Zero downtime:** background migration using Azure Functions running on scheduled intervals.
- **Consistency:** copy records to Blob, verify success, then delete from Cosmos to avoid data loss.
- **Unified read logic:** API layer queries Cosmos first and Blob if needed, abstracted internally.
- **Folder structure in Blob:** organized by customer and month (e.g., customerId/year-month/recordId.json).
- **Potential risks:** migration failure, latency from cold accesses, rehydration needs, security, and operational overhead.
- **Mitigation:** robust retry, caching, secure storage, monitoring, and alerting.


## 4. Architecture Diagram

```
+-------------------+      (1) Write (Recent Data)      +---------------------+
|  CLIENT / API     +------------------------------->   |   Cosmos DB (Hot)   |
|    Consumer       |                                  |  (<=3 months)       |
+-------------------+                                  +----------+----------+
        ^                                                          |
        |                                              (4) Read    |
        |       (3) Read (Old Data)                    (recent)    |
        |                                              |           |
+-------+-----+       (2) Move Old Data                |           |
|  Unified   +<----------------------------------------+           |
|  Data      |                                  +------------------v----------+
|  Access    +--------------------------------->|  Blob Storage (Cold, JSON)  |
|  Layer     |                              (3) |   (>3 months old data)      |
+------------+----------------------------------+----------------------------+
```


## 5. Core Implementation Snippets

### A. Azure Function for Archival (Python, Async)

```python
import asyncio
from azure.cosmos.aio import CosmosClient
from azure.storage.blob.aio import BlobServiceClient
import json
from datetime import datetime, timedelta

COSMOS_ENDPOINT = "<COSMOS_ENDPOINT>"
COSMOS_KEY = "<COSMOS_KEY>"
DATABASE_NAME = "<DB_NAME>"
CONTAINER_NAME = "<CONTAINER>"
BLOB_CONN_STR = "<BLOB_CONN_STRING>"
BLOB_CONTAINER = "<BLOB_CONTAINER>"
DAYS_OLD = 90  # 3 months

async def archive_old_records():
    cosmos_client = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
    container = (await cosmos_client.get_database_client(DATABASE_NAME)
                                  .get_container_client(CONTAINER_NAME))
    blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
    blob_container = blob_service.get_container_client(BLOB_CONTAINER)
    
    cutoff = datetime.utcnow() - timedelta(days=DAYS_OLD)
    query = f"SELECT * FROM c WHERE c.timestamp <= '{cutoff.isoformat()}'"
    async for item in container.query_items(query, enable_cross_partition_query=True):
        blob_name = f"{item['customer_id']}/{item['timestamp'][:7]}/{item['id']}.json"
        await blob_container.upload_blob(blob_name, json.dumps(item), overwrite=True)
        await container.delete_item(item, partition_key=item['customer_id'])
    await cosmos_client.close()

asyncio.run(archive_old_records())
```


### B. Unified Data Access Function (Pseudo Python)

```python
from azure.cosmos import CosmosClient, exceptions
from azure.storage.blob import BlobServiceClient
import json

def get_billing_record(record_id, customer_id):
    try:
        doc = container.read_item(item=record_id, partition_key=customer_id)
        return doc
    except exceptions.CosmosResourceNotFoundError:
        blob_name = f"{customer_id}/{record_id[:7]}/{record_id}.json"
        blob_client = blob_service.get_blob_client(container=BLOB_CONTAINER, blob=blob_name)
        if blob_client.exists():
            return json.loads(blob_client.download_blob().readall())
        else:
            raise Exception("Record not found")
```


### C. Blob Storage Data Organization Example

```
customer123/2023-02/abcde123.json
```

JSON Example:

```json
{
  "id": "abcde123",
  "customer_id": "customer123",
  "timestamp": "2023-02-15T09:53:00Z",
  "amount": 200,
  "details": { ... }
}
```


## 6. Infrastructure as Code Sample (Azure Bicep)

```bicep
resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts@2021-04-15' = {
  name: 'billing-cosmosdb'
  location: resourceGroup().location
  kind: 'GlobalDocumentDB'
  properties: {
    consistencyPolicy: { defaultConsistencyLevel: 'Session' }
    locations: [{ locationName: resourceGroup().location }]
    databaseAccountOfferType: 'Standard'
  }
}

resource blobStorage 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: 'billingarchive'
  location: resourceGroup().location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource functionApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'archive-billing-func'
  location: resourceGroup().location
  kind: 'functionapp'
  properties: { serverFarmId: appServicePlan.id }
}
```


## 7. Monitoring and Telemetry Recommendations

- Enable Azure Application Insights on Azure Functions and API service.
- Log successes, failures, fallback reads, and latency metrics.
- Example logging snippet:

```python
import logging
from opencensus.ext.azure.log_exporter import AzureLogHandler

logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(connection_string="InstrumentationKey=..."))

def log_event(event_type, message, custom_dimensions=None):
    logger.info(message, extra={'custom_dimensions': custom_dimensions or {}})
```

- Configure alerts on:
    - Failed archival jobs.
    - Slow blob reads.
    - Increased fallback cold reads.
- Create dashboard showing Cosmos RU consumption, Blob storage growth, and fallback event counts.


## 8. Future Risks \& Mitigations

| Risk | Description | Mitigation |
| :-- | :-- | :-- |
| Cold Data Access Latency | Blob Storage access slower than Cosmos | Cache recent cold data, use premium Blob tier |
| Migration Failure | Partial migration failures causing duplicates/loss | Use copy-then-delete, logging, and retries |
| Blob Layout Inefficiency | Hotspots or performance degradation in Blob | Use folder partitioning by customer and month |
| Rehydration Complexity | Difficulty moving cold data back to Cosmos | Maintain rehydration scripts and procedures |
| Security Concerns | Unauthorized access or unencrypted data | Apply RBAC, encryption, and private endpoints |
| Race Conditions in Writes | Late data writes could be missed in migration job | Handle late data in archival job, add guardrails |
| Operational Overhead | Continuous monitoring and management effort | Automate alerts and monitoring |

## 9. Summary Recommendations

- Use hybrid hot-cold storage pattern with Cosmos DB and Azure Blob Storage.
- Ensure archival runs incrementally, safely, and with retries.
- Implement unified read logic to preserve existing API contracts.
- Monitor using Azure native tools; alert proactively.
- Document code, architecture, and operations clearly.
- Test restore and rehydration procedures regularly.
- Tune retention window and blob tier for cost/performance balance.
