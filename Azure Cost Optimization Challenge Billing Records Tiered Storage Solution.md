**Azure Cost Optimization Challenge: Billing Records Tiered Storage Solution**

**Date:** Monday, August 04, 2025\
By-Prashant Poonia

**1. Problem Statement & Context**

- The system stores over 2 million billing records in Azure Cosmos DB.
- Each record can be up to 300 KB in size.
- The system is read-heavy, but records older than 3 months are rarely accessed.
- The database size and associated cost have grown significantly over time.
- Goal: Optimize costs while ensuring availability and minimal latency (seconds) for old records.
- Constraints:
  - No downtime or data loss during migration.
  - No changes to existing API contracts.
  - Solution must be simple and maintainable.

**2. Initial Solution Outline**

**Tiered Storage Approach:**

- **Hot Data:** Keep billing data for the last 3 months in Cosmos DB (fast access, costly).
- **Cold Data:** Archive older data (records older than 3 months) into Azure Blob Storage (low cost, slower).
- **Unified Access Layer:** Introduce a logic layer that checks Cosmos DB first; if not found, fetches data from Blob Storage — keeps APIs unchanged.

**Benefits:**

- Reduced Cosmos DB storage and RU consumption costs.
- Keeps data available with acceptable latency.
- Minimal changes to client-facing APIs.

**3. Process**

- **Separation of concerns:** hot data in Cosmos DB, cold data in Blob Storage.
- **Zero downtime:** background migration using Azure Functions running on scheduled intervals.
- **Consistency:** copy records to Blob, verify success, then delete from Cosmos to avoid data loss.
- **Unified read logic:** API layer queries Cosmos first and Blob if needed, abstracted internally.
- **Folder structure in Blob:** organized by customer and month (e.g., customerId/year-month/recordId.json).
- **Potential risks:** migration failure, latency from cold accesses, rehydration needs, security, and operational overhead.
- **Mitigation:** robust retry, caching, secure storage, monitoring, and alerting.

**4. Architecture Diagram**

+-------------------+      (1) Write (Recent Data)      +---------------------+\
|  CLIENT / API     +------------------------------->   |   Cosmos DB (Hot)   |\
|    Consumer       |                                  |  (<=3 months)       |\
+-------------------+                                  +----------+----------+\
`        `^                                                          |\
`        `|                                              (4) Read    |\
`        `|       (3) Read (Old Data)                    (recent)    |\
`        `|                                              |           |\
+-------+-----+       (2) Move Old Data                |           |\
|  Unified   +<----------------------------------------+           |\
|  Data      |                                  +------------------v----------+\
|  Access    +--------------------------------->|  Blob Storage (Cold, JSON)  |\
|  Layer     |                              (3) |   (>3 months old data)      |\
+------------+----------------------------------+----------------------------+



**5. Core Implementation Snippets**

**A. Azure Function for Archival (Python, Async)**

import asyncio\
from azure.cosmos.aio import CosmosClient\
from azure.storage.blob.aio import BlobServiceClient\
import json\
from datetime import datetime, timedelta\
\
COSMOS\_ENDPOINT = "<COSMOS\_ENDPOINT>"\
COSMOS\_KEY = "<COSMOS\_KEY>"\
DATABASE\_NAME = "<DB\_NAME>"\
CONTAINER\_NAME = "<CONTAINER>"\
BLOB\_CONN\_STR = "<BLOB\_CONN\_STRING>"\
BLOB\_CONTAINER = "<BLOB\_CONTAINER>"\
DAYS\_OLD = 90  # 3 months\
\
async def archive\_old\_records():\
`    `cosmos\_client = CosmosClient(COSMOS\_ENDPOINT, COSMOS\_KEY)\
`    `container = (await cosmos\_client.get\_database\_client(DATABASE\_NAME)\
.get\_container\_client(CONTAINER\_NAME))\
`    `blob\_service = BlobServiceClient.from\_connection\_string(BLOB\_CONN\_STR)\
`    `blob\_container = blob\_service.get\_container\_client(BLOB\_CONTAINER)\
\
`    `cutoff = datetime.utcnow() - timedelta(days=DAYS\_OLD)\
`    `query = f"SELECT \* FROM c WHERE c.timestamp <= '{cutoff.isoformat()}'"\
`    `async for item in container.query\_items(query, enable\_cross\_partition\_query=True):\
`        `blob\_name = f"{item['customer\_id']}/{item['timestamp'][:7]}/{item['id']}.json"\
`        `await blob\_container.upload\_blob(blob\_name, json.dumps(item), overwrite=True)\
`        `await container.delete\_item(item, partition\_key=item['customer\_id'])\
`    `await cosmos\_client.close()\
\
asyncio.run(archive\_old\_records())


**B. Unified Data Access Function (Pseudo Python)**

from azure.cosmos import CosmosClient, exceptions\
from azure.storage.blob import BlobServiceClient\
import json\
\
def get\_billing\_record(record\_id, customer\_id):\
`    `try:\
`        `doc = container.read\_item(item=record\_id, partition\_key=customer\_id)\
`        `return doc\
`    `except exceptions.CosmosResourceNotFoundError:\
`        `blob\_name = f"{customer\_id}/{record\_id[:7]}/{record\_id}.json"\
`        `blob\_client = blob\_service.get\_blob\_client(container=BLOB\_CONTAINER, blob=blob\_name)\
`        `if blob\_client.exists():\
`            `return json.loads(blob\_client.download\_blob().readall())\
`        `else:\
`            `raise Exception("Record not found")

**C. Blob Storage Data Organization Example**

customer123/2023-02/abcde123.json

JSON Example:

{\
`  `"id": "abcde123",\
`  `"customer\_id": "customer123",\
`  `"timestamp": "2023-02-15T09:53:00Z",\
`  `"amount": 200,\
`  `"details": { ... }\
}



**6. Infrastructure as Code Sample (Azure Bicep)**

resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts@2021-04-15' = {\
`  `name: 'billing-cosmosdb'\
`  `location: resourceGroup().location\
`  `kind: 'GlobalDocumentDB'\
`  `properties: {\
`    `consistencyPolicy: { defaultConsistencyLevel: 'Session' }\
`    `locations: [{ locationName: resourceGroup().location }]\
`    `databaseAccountOfferType: 'Standard'\
`  `}\
}\
\
resource blobStorage 'Microsoft.Storage/storageAccounts@2022-09-01' = {\
`  `name: 'billingarchive'\
`  `location: resourceGroup().location\
`  `sku: { name: 'Standard\_LRS' }\
`  `kind: 'StorageV2'\
}\
\
resource functionApp 'Microsoft.Web/sites@2023-01-01' = {\
`  `name: 'archive-billing-func'\
`  `location: resourceGroup().location\
`  `kind: 'functionapp'\
`  `properties: { serverFarmId: appServicePlan.id }\
}






**7. Monitoring and Telemetry Recommendations**

- Enable Azure Application Insights on Azure Functions and API service.
- Log successes, failures, fallback reads, and latency metrics.
- Example logging snippet:

import logging\
from opencensus.ext.azure.log\_exporter import AzureLogHandler\
\
logger = logging.getLogger(\_\_name\_\_)\
logger.addHandler(AzureLogHandler(connection\_string="InstrumentationKey=..."))\
\
def log\_event(event\_type, message, custom\_dimensions=None):\
`    `logger.info(message, extra={'custom\_dimensions': custom\_dimensions or {}})

- Configure alerts on:
  - Failed archival jobs.
  - Slow blob reads.
  - Increased fallback cold reads.
- Create dashboard showing Cosmos RU consumption, Blob storage growth, and fallback event counts.







**8. Future Risks & Mitigations**

|Risk|Description|Mitigation|
| :- | :- | :- |
|Cold Data Access Latency|Blob Storage access slower than Cosmos|Cache recent cold data, use premium Blob tier|
|Migration Failure|Partial migration failures causing duplicates/loss|Use copy-then-delete, logging, and retries|
|Blob Layout Inefficiency|Hotspots or performance degradation in Blob|Use folder partitioning by customer and month|
|Rehydration Complexity|Difficulty moving cold data back to Cosmos|Maintain rehydration scripts and procedures|
|Security Concerns|Unauthorized access or unencrypted data|Apply RBAC, encryption, and private endpoints|
|Race Conditions in Writes|Late data writes could be missed in migration job|Handle late data in archival job, add guardrails|
|Operational Overhead|Continuous monitoring and management effort|Automate alerts and monitoring|

**9. Summary Recommendations**

- Use hybrid hot-cold storage pattern with Cosmos DB and Azure Blob Storage.
- Ensure archival runs incrementally, safely, and with retries.
- Implement unified read logic to preserve existing API contracts.
- Monitor using Azure native tools; alert proactively.
- Document code, architecture, and operations clearly.
- Test restore and rehydration procedures regularly.
- Tune retention window and blob tier for cost/performance balance.

