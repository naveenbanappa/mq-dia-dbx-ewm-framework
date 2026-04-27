# **GLOBAL MANUFACTURING DATA FABRIC (GMDF-DP) <br/> DESIGN SPECIFICATIONS**

|**REVIEWERS**  |
| :-- |

The following individuals have reviewed this document for their area ofexpertise.

-  **DIA SME –** Mukul, Marco.
-  **DIA Solution Architect –** Mukul, Marco
-  **DIA Validation Lead** – 

|**AGREEMENTS** |
| :-- |
The following individuals have agreed with the contents of this document.
- **System Custodian** – TBD
- **System Owner** – Silvia Carloni
- **Computer System Quality Assurance (CSQA)** – TBD

<br/>

**PURPOSE**

This Design Specification provides information necessary for support teams to build and maintain the GMDF-DP DDL-Driven Streaming Framework in Azure Databricks. It will be kept up to date and will present current design information throughout the system lifecycle.

<br/>

**IN SCOPE**

The following are in scope:

-   A generic, DDL-driven, and self-healing streaming framework for Databricks.
-   Control Plane for schema management and activation.
-   Data Plane for streaming ingestion and historical data backfilling.
-   Code versioning and deployment via GitHub Actions.

**OUT OF SCOPE**

Anything not specifically mentioned in the "in scope" section of this document.

<br/>

**ACRONYMS AND DEFINITIONS**

The following acronyms are used:

|**Acronym**|**Description**|
| :-------- | :------------ |
|**DDL**|Data Definition Language|
|**UC**|Unity Catalog|
|**ADLS**|Azure Data Lake Storage|
|**GMDF-DP**|Global Manufacturing Data Fabric - Digital Plant|

<br/>

**TABLE OF CONTENTS**

[1. OVERVIEW	](#1-OVERVIEW)

[2. DESIGN SPECIFICATION	](#2-design-specification)

[2.1 System diagram and architectural components	](#21-system-diagram-and-architectural-components)

[2.2 Core Concepts: DDL-Driven Design](#22-core-concepts-ddl-driven-design)

[2.3 Control Plane Design](#23-control-plane-design)

[2.3.1 Control Tables](#231-control-tables)

[2.3.2 Schema Definition Files (DDL)](#232-schema-definition-files-ddl)

[2.3.3 Job A: The Schema Activator (`activate_schema.ipynb`)](#233-job-a-the-schema-activator-activate_schemaipynb)

[2.4 Data Plane Design](#24-data-plane-design)

[2.4.1 Job B: The Streaming Ingestion Job (`ewm_streaming_job.ipynb`)](#241-job-b-the-streaming-ingestion-job-ewm_streaming_jobipynb)

[2.4.2 The Backfill Job (`backfill_processor.ipynb`)](#242-the-backfill-job-backfill_processoripynb)

[2.5 Error Management and Incident Notification](#25-error-management-and-incident-notification)

[DOCUMENT REVISION HISTORY	](#document-revision-history)

# **1. OVERVIEW**

The end goal of the Global Manufacturing Data Fabric - Digital Plant (GMDF-DP) is to centralize and integrate manufacturing data from various sources and provide robust, scalable access to that data for reporting and analytics solutions.

This document describes a generic, DDL-driven, and self-healing streaming framework designed for Databricks. The core principle is the **separation of the control plane (schema management) from the data plane (streaming ingestion)**. The framework is designed to handle schema evolution automatically, including a **"Commit-Then-Correct"** backfill strategy, ensuring data integrity and minimizing downtime.

MQ Data Integration Analytics (DIA) is responsible for all aspects of the extraction, transform, and load of source system data to GMDF-DP. The architecture is built with reusable, parameterized components orchestrated through Databricks Jobs and GitHub Actions.

# **2. DESIGN SPECIFICATION**

## **2.1. System diagram and architectural components**

The Figure 1 below shows how the DDL-driven framework is structured in terms of architectural layers and how the components interact.

![](images/figure1.jpg)
**Figure 1**

The architecture involves the usage of multiple components to schedule and manage the data loading and schema evolution process. A key feature of the framework is that all components are designed to be generic and reusable, driven by configuration and DDL files.

The following table summarizes all the architectural components used:

|**Name**|**Cloud Service**|**Note**|
| :-- | :-- | :-- |
|**DDL Schema Files**|GitHub Repository|JSON files defining source-to-target mapping stored in `/schemas`.|
|***activate\_schema***|GitHub Action|Triggers Job A when a DDL file is modified.|
|***ewm\_ddl\_schema\_activator***|Databricks Batch Job|Job A: Reads DDL, syncs schema, and activates new version.|
|***ewm\_streaming\_job***|Databricks Streaming Job|Job B: Continuously ingests data based on the active schema.|
|***ewm\_backfill\_processor***|Databricks Batch Job|Backfill Job: Corrects historical data after a schema change.|
|***activate\_schema.ipynb***|Databricks Notebook|GitHub: `job_a_activate_schema.txt`|
|***ewm\_streaming\_job.ipynb***|Databricks Notebook|GitHub: `job_b_ewm_streaming.txt`|
|***backfill\_processor.ipynb***|Databricks Notebook|GitHub: `job_backfill.txt`|

<br/>

Jobs and functions read necessary information from Control Tables stored in Azure Databricks Unity Catalog. See [section 2.3.1](#231-control-tables).

## **2.2. Core Concepts: DDL-Driven Design**

The framework is built on the principle that the data processing logic should be defined declaratively and be separated from the engine that executes it.

-   **Declarative DDL**: The complete source-to-target mapping, including transformations like JSON unpacking, is defined in a simple JSON file. This file serves as the single source of truth for a table's schema.
-   **Schema Hash**: A canonical hash of the DDL's mapping logic is calculated to uniquely identify a schema version. This ensures that only meaningful changes (not cosmetic ones) trigger a new version.
-   **"Commit-Then-Correct" Strategy**: When a schema changes, the streaming job (Job B) immediately switches to writing data with the new schema for all incoming records (Commit). It then triggers a separate, asynchronous backfill job to find and update historical records to the new schema (Correct). This prevents pipeline stalls during schema evolution.

## **2.3 Control Plane Design**

The Control Plane is responsible for managing schema definitions, detecting changes, and safely applying them to the environment.

### **2.3.1. Control Tables**

All control and audit tables are managed within Databricks Unity Catalog to ensure centralized governance and access control.

|**Table name**|**Database name**|**Schema name**|**Description**|
| :-- | :-- | :-- | :-- |
|**schema\_transition**|`mq_gmdf_dp_poc`|`metadata_ddl`|The master signal table. Records every new schema hash activation for a target table. Includes `backfill_status` ('PENDING', 'IN_PROGRESS', 'COMPLETE', 'FAIL') to manage the backfill lifecycle.|
|**schema\_store\_columns**|`mq_gmdf_dp_poc`|`metadata_ddl`|The detailed blueprint table. Stores the column-level extraction logic (source JSON, field name, type) for each schema hash. Used by Job B and the Backfill Job.|
|**job\_a\_execution\_log**|`mq_gmdf_dp_poc`|`metadata_ddl`|Audit table to log the execution status and outcome of each run of the Schema Activator job (Job A).|
|**job\_b\_run\_log**|`mq_gmdf_dp_poc`|`metadata_ddl`|High-level log for each run of the streaming job (Job B), tracking start/stop events and the schema hash it's locked to.|
|**job\_b\_microbatch\_log**|`mq_gmdf_dp_poc`|`metadata_ddl`|Detailed log for each micro-batch processed by Job B, tracking record counts and processing times.|

### **2.3.2. Schema Definition Files (DDL)**

The schema for a target table is defined in a JSON file stored in a version-controlled GitHub repository (e.g., in a `/schemas` directory). This DDL file specifies the target table name, metadata, and a list of all columns.

For each column, the DDL defines its name, data type, and, crucially, its `source_mapping`. The mapping dictates how the value is derived:

-   `"ATOMIC.<source_column_name>"`: Direct mapping from a root-level field in the source data.
-   `"JSON.<source_json_column>.<path.to.field>"`: Unpacks a field from a nested JSON string stored in another column.
-   `"STRUCT.<source_struct_column>.<path.to.field>"`: Extracts a field from a structured type, like Auto Loader's `__metadata` column.
-   `"PIPELINE.<column_name>"`: Indicates the column is generated by the pipeline itself (e.g., `ingest_ts`).

***JSON DDL Example***
```json
{
  "table_name": "mq_gmdf_dp_poc.kafka_ewm_poc.ewm_ddl_ext",
  "domain": "EWM",
  "description": "Governed EWM interface logs with flattened payload attributes",
  "columns": [
    { "name": "lgnum", "type": "string", "source_mapping": "ATOMIC.Lgnum"},
    { "name": "ReqPayload", "type": "string", "source_mapping": "ATOMIC.ReqPayload"},
    { "name": "req_clientId", "type": "string", "source_mapping": "JSON.ReqPayload.clientId"},
    { "name": "meta_filePath", "type": "string", "source_mapping": "STRUCT.__metadata.file_path"},
    { "name": "ingest_ts", "type": "timestamp", "source_mapping": "PIPELINE.ingest_ts"}
  ]
}
### **2.3.3. Job A: The Schema Activator (`activate_schema.ipynb`)**

This component is the engine of the Control Plane. It is a generic, parameterized batch job responsible for activating new schema versions.

**Orchestration:**
-   **Trigger**: A GitHub Action that runs when a DDL JSON file is pushed or merged to a specific branch in the `/schemas` directory.
-   **Parameters**: The job takes the raw GitHub URL of the modified DDL file as a parameter.

**Execution Logic:**
1.  **Log Start**: Logs its execution start in the `job_a_execution_log` table for auditability.
2.  **Fetch DDL**: Reads the DDL file from the GitHub URL passed as a parameter.
3.  **Calculate Hash**: Calculates a canonical MD5 hash of the source-to-target mapping logic. This hash ignores cosmetic changes like descriptions.
4.  **Compare Hashes**: Compares the new hash to the latest hash for the same table in the `schema_transition` table.
5.  **No Change**: If the hashes match, the job logs success and exits.
6.  **Change Detected**: If a change is detected, it proceeds with the activation workflow:
    a.  **Sync Physical Table**: It performs a non-destructive `ALTER TABLE ADD COLUMNS` on the target Delta table for any new columns. This is a safe operation that does not rewrite data. If the table does not exist, it creates it.
    b.  **Update Blueprint**: It overwrites the partition for the target table in the `schema_store_columns` table with the new, detailed column mappings.
    c.  **Activate New Schema**: It inserts a new record into `schema_transition` with the new schema hash and a `backfill_status` of **'PENDING'**. This atomic insert acts as the master signal for the rest of the system.
7.  **Log Final Status**: Updates its execution log with the final status (SUCCESS/FAIL).

## **2.4. Data Plane Design**

The Data Plane is responsible for the continuous ingestion of new data and the correction of historical data according to the schema versions defined by the Control Plane.

### **2.4.1. Job B: The Streaming Ingestion Job (`ewm_streaming_job.ipynb`)**

This is a generic, parameterized continuous streaming job that forms the core of the Data Plane.

**Orchestration:**
-   **Trigger**: Runs continuously as a Databricks Job, processing data in micro-batches using a `processingTime` trigger (e.g., every 30 seconds).
-   **Security**: Access to source data in ADLS is handled securely by passing the `storage_account_name`, `secret_scope_name`, and `secret_key_name` as job parameters. The secret value itself is fetched at runtime using `dbutils.secrets.get()`.

**`foreachBatch` Logic:**
1.  **Locking**: On startup, the job queries `schema_transition` to find the latest schema hash for its target table and "locks" itself to that version.
2.  **Graceful Shutdown Check**: In every micro-batch, it re-checks if a newer hash has been activated in `schema_transition`. If the latest hash is different from its locked hash, it throws an exception to stop itself. The orchestration system (e.g., Databricks Jobs with retries) is expected to restart it, at which point it will lock to the new version.
3.  **Dynamic Transformation**: It fetches the detailed column blueprint from `schema_store_columns` for its locked hash. It dynamically builds a schema and uses `from_json` to unpack data from the specified source columns (e.g., `ReqPayload`, `RespPayload`).
4.  **Stamping**: It stamps each processed row with pipeline-generated columns, including `ingest_ts` and `schema_hash_applied`, which contains its locked schema hash. This stamping is critical for the backfill process.
5.  **Write to Delta**: It appends the transformed micro-batch to the target Delta table.
6.  **Automatic Backfill Trigger**: After its *very first* successful micro-batch, it checks the `backfill_status` for its locked hash in `schema_transition`.
    -   If the status is **'PENDING'**, it makes an API call to the Databricks Jobs API to trigger the `ewm_backfill_processor` job in 'WET_RUN' mode. It passes the old and new schema hashes.
    -   It then immediately updates the status in `schema_transition` to **'IN_PROGRESS'** to prevent duplicate triggers. *(Note: This API call is designed but commented out pending Service Principal setup for authentication).*

### **2.4.2. The Backfill Job (`backfill_processor.ipynb`)**

This is a generic, parameterized batch job whose sole purpose is to correct historical data after a schema change. It is a critical part of the "Commit-Then-Correct" strategy.

**Orchestration:**
-   **Trigger**: Triggered automatically by Job B via the Jobs API, or can be run manually by an operator for retries or dry runs.
-   **Parameters**: `target_table_name`, `old_schema_hash`, `new_schema_hash`, and `run_mode` ('DRY_RUN' or 'WET_RUN').

**Execution Logic:**
1.  **Identify Schema Difference**: Queries `schema_store_columns` for both the `old_schema_hash` and `new_schema_hash` to determine the exact set of columns that were added in the new version.
2.  **Find Target Rows**: Finds all rows in the target table that need to be fixed by filtering on `WHERE schema_hash_applied = 'old_schema_hash'`.
3.  **Generate `UPDATE` Statement**: Dynamically generates a single `UPDATE` statement.
    -   The `SET` clauses use `get_json_object()` to re-parse the raw JSON from columns (e.g., `ReqPayload`) that are already stored in each row, populating the newly added columns.
    -   It also updates `schema_hash_applied` to the `new_schema_hash` to mark the rows as corrected.
4.  **Execute Run Mode**:
    -   **`DRY_RUN`**: Prints a report of the columns to be populated and the number of rows that would be affected. No changes are made.
    -   **`WET_RUN`**: Executes the generated `UPDATE` statement. After a successful update, its final action is to set the `backfill_status` in `schema_transition` to **'COMPLETE'**. If the update fails, it sets the status to **'FAIL'**.

## **2.5. Error Management and Incident Notification**

-   **Job A (Schema Activator)**: Contains comprehensive `try...except` blocks. If any step fails, it updates its record in `job_a_execution_log` to 'FAIL' with a detailed error message and then re-raises the exception to fail the Databricks job run, triggering any configured alerts.
-   **Job B (Streaming)**: Relies on the graceful shutdown mechanism for schema changes. For other processing errors, the exception will cause the stream to fail. Standard Databricks alerting on job failure is used for notification.
-   **Backfill Job**: In a `WET_RUN`, if the `UPDATE` statement fails, it sets the `backfill_status` to 'FAIL' in the `schema_transition` table. This provides a clear signal for manual investigation and prevents the system from assuming the backfill is complete.

## **DOCUMENT REVISION HISTORY**

|**Version** |**Revision Date**|**Summary of Changes**|**Revised By**|**Reference Number**|
| :-- | :-- | :-- | :-- | :-- |
|1.0|2026-04-27|Initial document creation based on the DDL-Driven Streaming Framework.|naveenbanappa|-|