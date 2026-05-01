# **GLOBAL MANUFACTURING DATA FABRIC (GMDF-DP) <br/> DESIGN SPECIFICATIONS**

|**REVIEWERS** |
| :--- |
|The following individuals have reviewed this document for their area of expertise.|
|- **DIA SME –** |
|- **DIA Solution Architect –** |
|- **DIA Validation Lead** – |

|**AGREEMENTS** |
| :--- |
|The following individuals have agreed with the contents of this document.|
|- **System Custodian** – |
|- **System Owner** – Silvia Carloni |
|- **Computer System Quality Assurance (CSQA)** – |

<br/>

**PURPOSE**

This Design Specification provides the necessary information for support teams to build, maintain, and understand the **GMDF-DP EWM Framework**. The framework is designed for schema-governed, continuous ingestion of SAP EWM data into the Databricks platform. It will be kept up to date and will present current design information throughout the system lifecycle.

<br/>

**IN SCOPE**

The following components and processes are in scope for this document:

*   GMDF-DP EWM data ingestion design, from landing zone files to a schema-governed Raw Layer Delta table.
*   The three-job architecture: Schema Activation (Job A), Continuous Ingestion (Job B), and Continuous Validation (Job C).
*   Schema governance, evolution, and backfill strategy.
*   Configuration management via GitHub.
*   Logging, validation, and operational monitoring strategy.

**OUT OF SCOPE**

Anything not specifically mentioned in the "in scope" section of this document, including downstream data consumption (e.g., a "Refined" layer), analytics, and reporting solutions built on top of the Raw Layer data.

<br/>

**ACRONYMS AND DEFINITIONS**

|**Acronym**|**Description**|
| :--- | :--- |
|**EWM**|Extended Warehouse Management (SAP Module)|
|**DDL**|Data Definition Language. In this context, a JSON file defining the schema.|
|**CI/CD**|Continuous Integration / Continuous Deployment|
|**GMDF-DP**|Global Manufacturing Data Fabric - Data Product|
|**ADLS**|Azure Data Lake Storage|

<br/>

**TABLE OF CONTENTS**

[1. OVERVIEW](#1-overview)

[2. DESIGN SPECIFICATION](#2-design-specification)

[2.1 System diagram and architectural components](#21-system-diagram-and-architectural-components)

[2.2 Source Systems](#22-source-systems)

[2.3 Data Layer Design](#23-data-layer-design)

[2.4 Technical Fields](#24-technical-fields)

[2.5 Configuration Management](#25-configuration-management)

[2.6 Ingestion Process & Job Orchestration](#26-ingestion-process--job-orchestration)

[2.6.1 Job A: Schema Activator & Backfill](#261-job-a-ewm_schema_activator_and_backfill_job)

[2.6.2 Job B: Continuous Ingestion](#262-job-b-ewm_continuous_ingestion_job)

[2.6.3 Job C: Continuous Ingestion Validation](#263-job-c-ewm_continuous_ingestion_validation_job)

[2.7 Error Management and Incident Notification](#27-error-management-and-incident-notification)

[2.8 Logging and Control Tables](#28-logging-and-control-tables)

[2.9 Post-Load and Continuous Validation Strategy](#29-post-load-and-continuous-validation-strategy)

# **1. OVERVIEW**

The GMDF-DP EWM Framework provides a robust, repeatable, and schema-governed process for ingesting continuous data streams from SAP EWM into the Databricks Lakehouse Raw Layer. The primary goal is to centralize and integrate manufacturing data, making it available for further processing and analytics while ensuring data integrity, traceability, and operational stability.

The architecture is built on a "config-as-code" principle, with all orchestration driven by workflows and configurations managed in a central GitHub repository. This enables safe, automated promotion of changes across environments and ensures that data structures are explicit, audited, and version-controlled.

The framework is composed of three distinct but cooperative Databricks jobs that handle schema management, data ingestion, and validation independently to ensure a clear separation of concerns and a resilient pipeline.

# **2. DESIGN SPECIFICATION**

## **2.1. System diagram and architectural components**

The architecture is structured as a three-job system orchestrated by GitHub Actions, operating on a shared set of control tables.

**Conceptual Flow Diagram**

```mermaid
graph TD
    subgraph GitHub
        A[Commit to Schema DDL File];
    end

    subgraph Databricks
        subgraph "Control Plane (Delta Tables)"
            CP1[schema_transition];
            CP2[schema_store_columns];
            CP3[job_a_execution_log];
            CP4[stream_job_execution_log];
            CP5[microbatch_execution_log];
        end

        B(Job A: ewm_schema_activator_and_backfill_job);
        C(Job B: ewm_continuous_ingestion_job);
        D(Job C: ewm_continuous_ingestion_validation_job);
    end
    
    subgraph ADLS
        E[Landing Zone (Parquet)];
        F[Raw Layer Table (Delta)];
    end

    A -->|Triggers| B;
    B -->|Writes| CP1;
    B -->|Writes| CP2;
    B -->|Writes| CP3;
    B -->|Alters| F;
    B -->|Backfills from| E;
    
    C -->|Reads Config from| CP1;
    C -->|Reads Blueprint from| CP2;
    C -->|Reads from| E;
    C -->|Writes to| F;
    C -->|Writes Log to| CP4;
    C -->|Writes Log to| CP5;

    D -->|Reads from| F;
    D -->|Reads Config from| CP1;
    D -->|Reports on| D;
```

**Figure 1: Architectural Flow**

The following table summarizes the core architectural components:

|**Name**|**Service/Type**|**Note**|
| :--- | :--- | :--- |
|**ewm\_schema\_activator\_and\_backfill\_job**|Databricks Job|**Job A:** On-demand job triggered by GitHub. Manages schema activation, Raw Table alteration, and historical data backfill.|
|**ewm\_continuous\_ingestion\_job**|Databricks Job|**Job B:** Long-running Structured Streaming job. Ingests data from the Landing Zone, enforcing the active schema. One job runs per table.|
|**ewm\_continuous\_ingestion\_validation\_job**|Databricks Job|**Job C:** Scheduled batch job. Independently monitors pipeline health, data freshness, and quality of the Raw Layer table.|
|**Schema DDL (*.json)**|GitHub-Managed File|The master schema definition file (e.g., `ewm_ddl_ext.json`). Changes to this file initiate the entire schema evolution process.|
|**Control Plane Tables**|Databricks Delta Tables|A set of tables in the `mq_gmdf_dp_poc.metadata_ddl` schema that store state, schema history, and execution logs.|

## **2.2. Source Systems**

The primary source system for this framework is **SAP EWM**. Data is delivered from the source to the landing zone as files.

*   **Source Type**: SAP
*   **Data Format**: Parquet files
*   **Instances**: Initially a single source, but the framework is designed to be configurable for multiple instances (e.g., different plants).

## **2.3. Data Layer Design**

The framework utilizes a two-layer approach within Azure Data Lake Storage (ADLS Gen2).

*   **Landing Layer**: This layer contains the raw, unaltered source data as delivered by the source system. It is the source for the continuous ingestion job (Job B).
    *   **Path**: The path is configuration-driven and passed to the job at runtime. An example path is `abfss://topics@mqosidevadls01.dfs.core.windows.net/ZMW_I_INTFLOGS_v2/`.
    *   **Note**: The path and access credentials are parameterized and passed from configuration files.

*   **Raw Layer**: This layer contains the validated, schema-governed, and enriched data, stored as a Delta table. It is the final destination for this ingestion pipeline and the source for all subsequent processing.
    *   **Table Name**: The target table name is determined by the `table_name` property within the source DDL JSON file (e.g., `mq_gmdf_dp_poc.kafka_ewm_poc.ewm_ddl_ext`).
    *   **Format**: Delta Lake

## **2.4. Technical Fields**

In addition to the business fields defined in the DDL, the ingestion pipeline (Job B) enriches the data with the following pipeline-generated columns to provide auditability and operational traceability.

|**Column Name**|**Data Type**|**Description**|
| :--- | :--- | :--- |
|**ingest\_batch\_id**|long|The microbatch ID from the Structured Streaming job (Job B) that processed the record. Useful for grouping records from a single batch.|
|**loaded\_timestamp**|timestamp|The timestamp when the record was processed and loaded into the Raw Layer table by Job B.|
|**schema\_hash\_at\_ingest**|string|The unique MD5 hash of the schema that was active when this record was ingested. This is critical for backfill operations and schema lineage.|
|**\_rescued\_data**|string|A column automatically added by Spark's `cloudFiles` or `from_json`. It captures any data that does not conform to the expected schema during parsing, preventing data loss.|

## **2.5. Configuration Management**

The framework is fundamentally config-driven, with the central source of truth residing in a **GitHub repository**.

*   **Schema Definition**: The schema for each table is explicitly defined in a JSON file (e.g., `schemas/ewm_ddl_ext.json`). This file contains the target table name, column definitions, data types, and source-to-target mappings.
*   **Orchestration**: GitHub Actions workflows are used to trigger jobs. A commit to a schema file in the `main` branch will automatically trigger **Job A** to apply the schema changes to the production environment.
*   **Job Parameters**: Table-specific parameters such as source paths, access credentials, and checkpoint locations are maintained in configuration files within the repository, allowing the same generic notebooks to be reused for multiple tables.

## **2.6. Ingestion Process & Job Orchestration**

The core of the framework is the orchestrated execution of three distinct jobs.

### **2.6.1. Job A: `ewm_schema_activator_and_backfill_job`**

This on-demand job serves as the control gate for all schema changes. It is triggered by a GitHub Action upon a commit to a schema DDL file.

**Tasks:**
1.  **Fetch & Hash Schema**: Reads the updated DDL file from the GitHub repository and computes a canonical MD5 hash of its structure.
2.  **Detect Change**: Compares the new hash against the currently 'ACTIVE' hash stored in the `schema_transition` control table. If they match, the job exits successfully.
3.  **Archive Old Schema**: If a change is detected, the job updates the `schema_transition` table, changing the status of the old schema from 'ACTIVE' to 'ARCHIVED'.
4.  **Activate New Schema**: A new record for the new schema hash is inserted into `schema_transition` with a status of 'ACTIVE' and a `backfill_status` of 'BACKFILL_IN_PROGRESS'.
5.  **Store Blueprint**: The detailed column mappings for the new schema are written to the `schema_store_columns` table, providing a "blueprint" for Job B.
6.  **Apply Schema to Table**: The job issues an `ALTER TABLE ... ADD COLUMNS` command to the target Raw Layer Delta table to add any new columns defined in the DDL.
7.  **Perform Backfill**: This is the most critical step. The job identifies all existing records in the Raw Layer table that were written with the old schema hash. It then reads the original source data from the Landing Layer, re-processes it to extract the data for the *newly added columns only*, and merges this new information back into the existing records, finally updating their `schema_hash_at_ingest` to the new hash.
8.  **Complete Backfill**: Upon successful completion of the merge, the `backfill_status` in the `schema_transition` table is updated to 'COMPLETE'.

### **2.6.2. Job B: `ewm_continuous_ingestion_job`**

This is a long-running, "always-on" Structured Streaming job responsible for the continuous flow of data from the Landing Layer to the Raw Layer.

**Tasks:**
1.  **Lock Active Schema**: At startup, the job queries the `schema_transition` table to get the single 'ACTIVE' schema hash for the table it is processing. It will use this hash for its entire run.
2.  **Read Source Data**: Uses Spark's `cloudFiles` to efficiently and incrementally read new Parquet files from the Landing Layer path specified in its configuration.
3.  **Dynamic Transformation**: For each microbatch of data:
    *   It reads the schema "blueprint" from the `schema_store_columns` table corresponding to its locked schema hash.
    *   It applies all specified transformations, including handling case-sensitivity issues in JSON payloads (`"ClientId"` -> `"clientId"`) and parsing nested JSON fields (`ReqPayload`, `RespPayload`).
    *   It dynamically constructs the final DataFrame, aligning columns to the DDL contract.
4.  **Enrich with Audit Columns**: Adds the `ingest_batch_id`, `loaded_timestamp`, and `schema_hash_at_ingest` technical fields.
5.  **Write to Raw Table**: Appends the processed microbatch to the Raw Layer Delta table. Schema enforcement ensures that the data strictly conforms to the target table's structure.

### **2.6.3. Job C: `ewm_continuous_ingestion_validation_job`**

This scheduled batch job runs independently to monitor and validate the output of Job B, ensuring the pipeline remains healthy and data quality is maintained.

**Tasks:**
1.  **Check Data Freshness**: Verifies that the `loaded_timestamp` of the newest record in the Raw Layer table is within an acceptable threshold (e.g., last 90 minutes), ensuring the pipeline is not stalled.
2.  **Verify Schema Consistency**: Confirms that all recently ingested records have the correct 'ACTIVE' schema hash, detecting any potential state inconsistencies.
3.  **Monitor Data Quality**:
    *   Calculates the null percentage for critical columns (e.g., `run_id`, `req_clientId`) and flags anomalies.
    *   Counts the number of records where data was captured in the `_rescued_data` column, indicating parsing or schema mismatch issues at the source.
4.  **Reporting**: Logs its findings. If any check fails, it raises an alert for investigation.

## **2.7. Error Management and Incident Notification**

Each job in the framework has built-in error handling.

*   **Job A & B**: If a fatal error occurs, the job logs the detailed exception to its corresponding log table (`job_a_execution_log` or `stream_job_execution_log`) and fails.
*   **Job C**: If a validation check fails, it will raise an alert to notify the support team.

**Incident Notification (Future Implementation)**
*   **Status**: Implementation in progress.
*   **Design**: The framework will be enhanced to automatically create an incident in **Service Now** upon job failure or a critical validation alert from Job C. The incident will contain the job name, run ID, error message, and a link to the detailed logs to expedite troubleshooting.

## **2.8. Logging and Control Tables**

The framework's state, history, and execution logs are stored in a centralized database (`mq_gmdf_dp_poc`) within the `metadata_ddl` schema. These tables are critical for the coordination and auditability of the entire system.

|**Table Name**|**Schema Name**|**Description**|
| :--- | :--- | :--- |
|**schema\_transition**|metadata_ddl|Acts as the master state machine. It tracks the lifecycle of each schema version (ACTIVE, ARCHIVED) and the status of backfill operations.|
|**schema\_store\_columns**|metadata_ddl|Contains the detailed "blueprint" for every schema hash. It stores column names, types, and source-to-target mappings, which Job B uses for dynamic processing.|
|**job\_a\_execution\_log**|metadata_ddl|Logs every execution of the schema activator job (Job A), capturing its status (RUNNING, SUCCESS, FAIL), timing, and any error messages.|
|**stream\_job\_execution\_log**|metadata_ddl|Contains high-level execution information for the streaming job (Job B), with one row per job run.|
|**microbatch\_execution\_log**|metadata_ddl|Provides detailed logs for each microbatch processed by Job B, including batch ID, row counts, schema hash used, and processing times.|

## **2.9. Post-Load and Continuous Validation Strategy**

Validation is a key principle of the framework, divided into two phases: post-schema activation validation and continuous operational validation.

### **2.9.1. Post-Activation Validation (After Job A)**

Immediately after Job A successfully completes a schema activation, a validation script is run to ensure the control plane was left in a consistent and correct state before Job B begins using the new schema.

**Checks Performed:**
1.  **State Coherency Check**:
    *   Verifies that there is **exactly one** schema marked as 'ACTIVE' for the target table in the `schema_transition` table.
    *   Confirms that all other historical schemas for the table are marked as 'ARCHIVED'.
    *   Ensures the hash of the 'ACTIVE' schema matches the newly activated hash.
2.  **Backfill Completeness Check**:
    *   Queries the `schema_transition` table to confirm the `backfill_status` for the new schema is either 'COMPLETE' or 'NOT_APPLICABLE'. Any other status (e.g., 'IN_PROGRESS') indicates a failure.
3.  **Blueprint Existence Check**:
    *   Confirms that a valid column blueprint for the newly activated schema hash exists in the `schema_store_columns` table. A missing blueprint would cause Job B to fail.

### **2.9.2. Continuous Validation (Job C)**

This scheduled job runs periodically to monitor the live pipeline and ensures ongoing data quality and health.

**Checks Performed:**
1.  **Data Freshness Check**:
    *   Measures the time elapsed since the most recent record was loaded (based on `loaded_timestamp`).
    *   Fails if this duration exceeds a configured threshold (e.g., 90 minutes), indicating a stalled pipeline.
2.  **Schema Hash Consistency Check**:
    *   Scans recent data (e.g., last 60 minutes) to ensure all records were written with the currently 'ACTIVE' schema hash from the `schema_transition` table.
    *   Detects if any part of the system is using an old or incorrect schema.
3.  **Null Rate Anomaly Check**:
    *   For a pre-defined list of critical columns it calculates the percentage of null values in the recent data.
    *   Fails if the null rate exceeds a configured threshold (e.g., 1.0%), flagging potential upstream data quality issues.
4.  **Rescued Data Volume Check**:
    *   Counts the number of recent records that have a non-null value in the `_rescued_data` column.
    *   Fails if this count exceeds a threshold (e.g., 100), indicating a persistent problem with source file structure or schema misalignment.