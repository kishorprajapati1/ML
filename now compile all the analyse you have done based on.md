<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# now compile all the analyse you have done based on the task and give pdf.

Here is a compiled technical document based on your task requirements and my previous analyses. You can copy the following content into a Markdown editor (like VS Code or Dillinger), export it as PDF, and use it as your solution documentation or interview deliverable.

# Azure Cost Optimization: Billing Records Tiered Storage Solution

**Date:** Monday, August 04, 2025
**Author:** [Your Name]
**Role:** Senior Developer

## Table of Contents

1. Problem \& Constraints
2. Solution Design

3. Architecture Overview
4. Data Flow Diagram
5. Implementation Roadmap
1. Core Code Snippets
2. Infrastructure as Code (Bicep)
3. Monitoring and Telemetry
4. Future Risks \& Mitigation
5. Recommendations

## 1. Problem \& Constraints

- Cosmos DB has become expensive due to >2 million billing records, each up to 300 KB.
- Records older than 3 months are rarely accessed, but must be available within seconds.
- No downtime, no data loss, no API changes, *simple and easy to maintain*.


## 2. Solution Design

### 2.1. Architecture Overview

- **Hot Data:** Records ≤ 3 months remain in Cosmos DB.
- **Cold Data:** Records > 3 months are moved to Azure Blob Storage.
- **Unified Data Access:** Application logic checks Cosmos DB first, then Blob; APIs remain unchanged.
- **Automated Background Archival:** Azure Function periodically migrates old records.


### 2.2. Data Flow Diagram

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


### 2.3. Implementation Roadmap

1. **Write migration/archival function** to move old records and clean up Cosmos DB.
2. **Develop unified data access logic** for seamless client reads.
3. **Automate deployments** (infra + code).
4. **Monitor** archival jobs, fallback reads, and costs.
5. **Plan for cold-data “rehydration”** in case of a shift in access patterns.

## 3. Core Code Snippets

### 3.1. Azure Function: Archival (Python, Async)

```python
from azure.cosmos.aio import CosmosClient
from azure.storage.blob.aio import BlobServiceClient
import json
from datetime import datetime, timedelta

async def archive_old_records():
    cosmos_client = CosmosClient(COSMOS_ENDPOINT, COSMOS_KEY)
    container = (await cosmos_client.get_database_client(DATABASE_NAME)
                                  .get_container_client(CONTAINER_NAME))
    blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
    blob_container = blob_service.get_container_client(BLOB_CONTAINER)
  
    cutoff = datetime.utcnow() - timedelta(days=90)
    query = f"SELECT * FROM c WHERE c.timestamp <= '{cutoff.isoformat()}'"
    async for item in container.query_items(query, enable_cross_partition_query=True):
        blob_name = f"{item['customer_id']}/{item['timestamp'][:7]}/{item['id']}.json"
        await blob_container.upload_blob(blob_name, json.dumps(item), overwrite=True)
        await container.delete_item(item, partition_key=item['customer_id'])
```


### 3.2. Unified Read Logic

```python
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


### 3.3. Sample Blob Data Structure

Blob path: `customer123/2023-02/abcde123.json`

```json
{
  "id": "abcde123",
  "customer_id": "customer123",
  "timestamp": "2023-02-15T09:53:00Z",
  "amount": 200,
  "details": {...}
}
```


## 4. Infrastructure as Code (Bicep)

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


## 5. Monitoring and Telemetry

### Instrumentation

- Add Azure Application Insights for logs, errors, and run metrics.
- Use logging to record each archival, fallback read, and error.

```python
import logging
from opencensus.ext.azure.log_exporter import AzureLogHandler

logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(connection_string="InstrumentationKey=..."))

def log_event(event_type, message, custom_dimensions=None):
    logger.info(message, extra={'custom_dimensions': custom_dimensions or {}})
```


### Sample Alerts

- **Failed Archival Job:** Alert if Function fails.
- **Blob Read Latency:** Alert if >3s at P95.
- **Excess Cold Reads:** Alert if fallback reads spike.


### Dashboard

- Cosmos DB RUs over time (should fall).
- Blob storage use and cold read counts.


## 6. Future Risks \& Mitigation

| Issue | Description | Mitigation |
| :-- | :-- | :-- |
| Cold Data Access Latency | Blob Storage reads are slower than Cosmos DB | Cache recent cold reads or use Premium Blob tier |
| Migration Consistency | Migration job failure might duplicate/loss data | “Copy-then-delete”, idempotent archival, logging, and alerts |
| Blob Storage Structure | Poor tree perf if too many files in flat structure | Use hierarchical folders by customer and date |
| Rehydration Complexity | Need to move cold data back to hot if patterns change | Keep migration code, design “rehydrate tool” |
| Security | Blob Storage must be access-limited and encrypted | RBAC, managed identities, audit Blob/container policies |
| API Race Conditions | Late old writes could be missed in a one-shot job | Make archival job re-check boundary data, handle new inserts |
| Operational Overhead | Must monitor migration, fallback, and costs | Automate reporting \& alerting, review weekly |

## 7. Recommendations

- **Automate everything**—from deployment to archival.
- **Alerting and dashboards** for all migration, fallback, and cost signals.
- **Document all runbooks** and store scripts and migration logic in source control.
- **Test for disaster recovery**: practice Blob-to-Cosmos restores.
- Periodically review and tune “hot” data window and retention as cost and access patterns change.

**End of Document**

**Instructions:**

- Paste this Markdown into a Markdown-to-PDF tool (such as VS Code w/ Markdown PDF, or Dillinger).
- Export to PDF and you’re ready to share with interviewers, team, or management!

Let me know if you want the already-generated PDF or need minor modifications to tailor for your company or audience.

