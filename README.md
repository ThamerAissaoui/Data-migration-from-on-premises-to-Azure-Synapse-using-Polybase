# Data Migration from On-Premises to Azure Synapse using PolyBase

This project demonstrates how to migrate data from an on-premises SQL Server database to Azure Synapse Analytics using PolyBase. The AdventureWorks dataset is used for this data warehousing project.

## Table of Contents
- [Overview](#overview)
- [Dataset](#dataset)
- [Data Preparation](#data-preparation)
- [PolyBase Setup](#polybase-setup)
- [Data Migration Steps](#data-migration-steps)
- [Distribution Method](#distribution-method)
- [Performance Optimization](#performance-optimization)
- [Verification](#verification)
- [Conclusion](#conclusion)
- [References](#references)

## Overview
![image](https://github.com/user-attachments/assets/ba272125-e1e3-4a6a-9a98-008eb29f3f79)

The goal of this project is to illustrate the steps involved in migrating data from an on-premises environment to Azure Synapse Analytics using PolyBase. We use an expanded version of the AdventureWorks dataset, creating large fact and dimension tables by introducing dummy records.

## Dataset
We use the AdventureWorks dataset provided by Microsoft. You can download and configure the dataset from the following link:
[AdventureWorks Dataset](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms)

## Data Preparation
To prepare the dataset for migration, we first create two new tables in the `AdventureWorksDW2017` database: `DimBigProduct` and `FactTransactionHistory`. The size of these tables is increased by introducing dummy records, ensuring the dataset is large enough to simulate a real-world scenario.

```sql
-- Example script to create the dimension table
SELECT
    p.ProductKey + (a.number * 1000) AS ProductKey,
    p.EnglishProductName + CONVERT(VARCHAR, (a.number * 1000)) AS EnglishProductName,
    ...
INTO DimBigProduct
FROM DimProduct AS p
CROSS JOIN master..spt_values AS a
WHERE a.type = 'p'
AND a.number BETWEEN 1 AND 50;

-- Example script to create the fact table
SELECT 
    ROW_NUMBER() OVER (ORDER BY x.TransactionDate, (SELECT NEWID())) AS TransactionID,
    p1.ProductKey,
    ...
INTO FactTransactionHistory
FROM ...
```

These tables are then exported to flat files and stored in Azure Blob Storage.

## PolyBase Setup
PolyBase is used to load data into Azure Synapse. The following steps are required:

1. **Create Master Key**: To encrypt the credential secret.
2. **Create Database Scoped Credential**: To store the storage account key.
3. **Create External Data Source**: To specify the Azure Blob Storage as the data source.
4. **Create External File Format**: To define the format of the files.
5. **Create External Table**: To map the external data to the Synapse table.
6. **Create Table As (CTAS)**: To load the data into the final table.

## Data Migration Steps
Below is the SQL script used to set up PolyBase and migrate the data:

```sql
-- Step 1: Create a Master Key
CREATE MASTER KEY;

-- Step 2: Create a Database Scoped Credential
CREATE DATABASE SCOPED CREDENTIAL BlobStorageCredential
WITH IDENTITY = 'blobuser',  
SECRET = '<Your-Storage-Account-Key>';

-- Step 3: Create an External Data Source
CREATE EXTERNAL DATA SOURCE AzureBlobStorage
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://dp203polybase@dp203ploybase.blob.core.windows.net',
    CREDENTIAL = BlobStorageCredential
);

-- Step 4: Create an External File Format
CREATE EXTERNAL FILE FORMAT CSVFileFormatPoly 
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS  (FIELD_TERMINATOR = ',', STRING_DELIMITER = '', DATE_FORMAT = 'yyyy-MM-dd HH:mm:ss', USE_TYPE_DEFAULT = FALSE)
);

-- Step 5: Create an External Table
CREATE EXTERNAL TABLE [stage].FactTransactionHistory 
(
    [TransactionID] [int] NOT NULL,
    [ProductKey] [int] NOT NULL,
    [OrderDate] [datetime] NULL,
    [Quantity] [int] NULL,
    [ActualCost] [money] NULL
)
WITH (
    LOCATION = '/FileTransactionHistory.txt/', 
    DATA_SOURCE = AzureBlobStorage, 
    FILE_FORMAT = CSVFileFormatPoly, 
    REJECT_TYPE = VALUE, 
    REJECT_VALUE = 0
);

-- Step 6: Create Table As (CTAS)
CREATE TABLE [prod].[FactTransactionHistory]       
WITH (DISTRIBUTION = HASH([ProductKey])) 
AS 
SELECT * FROM [stage].[FactTransactionHistory]
OPTION (LABEL = 'Load [prod].[FactTransactionHistory]');
```

## Distribution Method
In this project, we use the **hash distribution method** based on the `ProductKey`. This ensures that data is evenly distributed across the nodes in Azure Synapse, optimizing query performance.

## Performance Optimization
After the data load completes, we recommend rebuilding the table to optimize query performance and compress all the rows into the columnstore.

```sql
ALTER INDEX ALL ON [prod].[FactTransactionHistory] REBUILD;
```

## Verification
To verify that the data was successfully loaded and distributed:

```sql
-- Verify number of rows
SELECT count(1) FROM [prod].[FactTransactionHistory];

-- Check the distribution and data skew
DBCC PDW_SHOWSPACEUSED('prod.FactTransactionHistory');
```

## Conclusion
This project demonstrates a complete process for migrating data from on-premises SQL Server to Azure Synapse Analytics using PolyBase. By following these steps, you can efficiently move and optimize large datasets in the cloud.

## References
- [AdventureWorks Dataset](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms)
- [PolyBase in Azure Synapse](https://docs.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-overview-polybase)
