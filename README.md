# Metadata-Driven Incremental ETL Pipeline (Azure Data Factory)

This project implements a scalable, **metadata-driven ETL framework** using Azure Data Factory (ADF). It is designed to incrementally load data from an Azure SQL Database into Azure Blob Storage (Parquet format) without hardcoding individual tables.

## 🚀 Key Features

- **Metadata-Driven:** New tables can be added to the pipeline simply by inserting a row into a SQL Control Table. No ADF changes required.
- **Incremental Loading:** Uses High-Watermark (CDC) logic to fetch only new or updated records.
- **Auto-Healing:** Automatically handles "First Run" scenarios for new tables.
- **Backfill Capability:** Supports manual backfilling via pipeline parameters.
- **Alerting:** Integrated with Azure Logic Apps to send email notifications for failures (instant) and success summaries (batch).

## 🏗 Architecture

1. **Source:** Azure SQL Database (Transactional Tables).
2. **Control Layer:** SQL Tables (`Metadata_Control_List` and `run_history`) manage the scope and state of the ETL.
3. **Orchestrator:** ADF Master Pipeline (`pl_dataset_orchestrator`).
4. **Worker:** ADF Child Pipeline (`pl_metadata_driven_sql_control`).
5. **Sink:** Azure Blob Storage (Bronze Layer / Parquet).
6. **Monitoring:** Azure Logic Apps (Email Notifications).

---

## 🛠 Prerequisites

- **Azure Data Factory**
- **Azure SQL Database**
- **Azure Storage Account** (Blob/Data Lake Gen2)
- **Azure Logic App** (Consumption plan) for sending emails.

---

## ⚙️ Database Setup (Control Tables)

Before running the pipeline, you must create the control tables in your Azure SQL Database.

### 1\. Metadata Control List (The "Menu")

This table defines which tables the pipeline should process.

SQL

```
CREATE TABLE dbo.Metadata_Control_List (
    ID INT IDENTITY(1,1) PRIMARY KEY,
    SourceSchema VARCHAR(50) DEFAULT 'dbo',
    SourceTableName VARCHAR(100) NOT NULL,
    CDC_Column VARCHAR(50) NOT NULL,       -- Column used for watermarking (e.g., updated_at)
    TargetContainer VARCHAR(50) NOT NULL,  -- e.g., 'bronze'
    IsActive BIT DEFAULT 1,                -- 1 = Enable, 0 = Disable
    Description VARCHAR(255)
);

-- Example Insert
INSERT INTO dbo.Metadata_Control_List (SourceTableName, CDC_Column, TargetContainer)
VALUES ('DimUser', 'updated_at', 'bronze');

```

### 2\. Run History (The "Memory")

This table tracks the last successful load time (Watermark) for each table.

SQL

```
CREATE TABLE dbo.run_history (
    table_name VARCHAR(255) PRIMARY KEY,
    watermark_value DATETIME,
    last_run_time DATETIME DEFAULT GETDATE()
);

```

---

## 🔄 Pipeline Logic

### 1\. Master Pipeline: `pl_dataset_orchestrator`

This is the entry point.

1. **Lookup:** Queries `Metadata_Control_List` to get all active tables (`IsActive = 1`).
2. **ForEach Loop:** Iterates through the list of tables.
3. **Execute Pipeline:** Triggers the worker pipeline (`pl_metadata_driven_sql_control`) for each table.
4. **Failure Handling:** If a table fails, it immediately sends a failure email via Logic App.
5. **Success Reporting:** After the loop finishes, it queries `run_history` to see which tables were updated in the last 30 minutes and sends a summary email.

### 2\. Worker Pipeline: `pl_metadata_driven_sql_control`

This performs the actual data movement.

1. **GetWatermark:** Checks `run_history`. If the table is new, it defaults to `1900-01-01`.
2. **Copy Data:**
   - Source Query: `SELECT * FROM Source WHERE updated_at > @Watermark`  
   - Sink: Writes to `Container/{TableName}/{TableName}_{Timestamp}.parquet`
3. **UpsertWatermark:** Upon success, updates (or inserts) the new maximum timestamp into `run_history`.

---

## 🔔 Alerting Configuration

The pipeline uses **Web Activities** to call an Azure Logic App HTTP trigger.

**Payload for Failure:**

JSON

```
{
    "to": "user@example.com",
    "subject": "ADF FAILURE: Load failed for table @{item().SourceTableName}",
    "message": "Error Details: @{activity('ExecuteWorker').error.message}"
}

```

**Payload for Success:**

JSON

```
{
    "to": "user@example.com",
    "subject": "Pipeline Success",
    "message": "Tables Updated: @{string(activity('getSuccesRuns').output.value)}"
}

```

---

## ▶️ How to Run

### Standard Run (Incremental)

1. Trigger `pl_dataset_orchestrator`.
2. Leave the `Backfill_Date` parameter **empty**.
3. The pipeline will process only new records created since the last run.

### Backfill Run (Historical Load)

1. Trigger `pl_dataset_orchestrator`.
2. Enter a date in `Backfill_Date` (e.g., `2023-01-01`).
3. The pipeline will ignore the stored watermark and reload all data from that date onwards.
