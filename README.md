# Real-Time CDC Pipeline: SQL Server to ClickHouse with Debezium and Kafka

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

This repository provides a complete, containerized demonstration of a real-time Change Data Capture (CDC) pipeline. It captures every row-level change (`INSERT`, `UPDATE`, `DELETE`) from a **Microsoft SQL Server** database, streams these events through **Apache Kafka** using **Debezium**, and loads them into a **ClickHouse** data warehouse for high-performance analytics. The pipeline also includes **Trino** as a federated query engine to analyze the data.

This project is a blueprint for building modern, event-driven data architectures that require low-latency replication from transactional systems to analytical platforms.

## Table of Contents

1.  [Key Features](#key-features)
2.  [System Architecture](#system-architecture)
3.  [Technology Stack](#technology-stack)
4.  [In-Depth Component Breakdown](#in-depth-component-breakdown)
    -   [Source Database (SQL Server)](#1-source-database-sql-server--cdc)
    -   [CDC Platform (Debezium)](#2-cdc-platform-debezium)
    -   [Message Broker (Apache Kafka)](#3-message-broker-apache-kafka)
    -   [Analytical Warehouse (ClickHouse)](#4-analytical-warehouse-clickhouse)
    -   [Federated Query Engine (Trino)](#5-federated-query-engine-trino)

5.  [Getting Started: Step-by-Step Guide](#getting-started-step-by-step-guide)
    -   [Prerequisites](#prerequisites)
    -   [Step 1: Clone the Repository](#step-1-clone-the-repository)
    -   [Step 2: Launch the Docker Environment](#step-2-launch-the-docker-environment)
    -   [Step 3: Configure the Debezium SQL Server Connector](#step-3-configure-the-debezium-sql-server-connector)
    -   [Step 4: Set Up the SQL Server Database and Enable CDC](#step-4-set-up-the-sql-server-database-and-enable-cdc)
    -   [Step 5: Set Up the ClickHouse Tables](#step-5-set-up-the-clickhouse-tables)
6.  [Testing the End-to-End Pipeline](#testing-the-end-to-end-pipeline)
    -   [Step 1: Perform Database Operations in SQL Server](#step-1-perform-database-operations-in-sql-server)
    -   [Step 2: Observe Change Events in Kafka](#step-2-observe-change-events-in-kafka)
    -   [Step 3: Query the Replicated Data in ClickHouse](#step-3-query-the-replicated-data-in-clickhouse)
7.  [Shutting Down the Environment](#shutting-down-the-environment)

## Key Features

-   **Non-Invasive CDC:** Captures data changes directly from the SQL Server transaction log without altering the source application.
-   **Real-Time Replication:** Propagates data changes with millisecond latency from the transactional database to the analytical warehouse.
-   **Decoupled & Resilient Architecture:** Kafka acts as a durable buffer, ensuring data is not lost even if downstream consumers (like ClickHouse) are temporarily unavailable.
-   **High-Performance Analytics:** Leverages ClickHouse's columnar storage and fast query engine for real-time analytics on the replicated data.
-   **Fully Containerized:** The entire stack is managed by Docker Compose, making setup and teardown simple and reproducible.
-   **Unified Querying:** Trino provides a single SQL interface to query data in ClickHouse, Kafka, and other connected systems.

## System Architecture

The data flows through the pipeline in the following sequence:

1.  **Data Change:** An `INSERT`, `UPDATE`, or `DELETE` operation occurs on the `dbo.Customers` table in the SQL Server database.
2.  **CDC Capture:** SQL Server's native CDC feature records this change in its system tables.
3.  **Event Streaming:** The **Debezium SQL Server Connector**, running on Kafka Connect, monitors the CDC tables. It captures the change, formats it into a detailed JSON event (including before/after state, operation type, and metadata), and publishes it to a dedicated Kafka topic (`sqlserver.dbo.Customers`).
4.  **Data Consumption:** A **ClickHouse** table (`kafka_customers`), powered by the Kafka engine, subscribes to this topic and reads the change events as they arrive.
5.  **Transformation & Loading:** A **ClickHouse Materialized View** (`customers_mv`) is triggered automatically for every new event in `kafka_customers`. It transforms the JSON data, extracts the relevant fields, and inserts the final, structured record into a permanent `MergeTree` table (`customers`).
6.  **Federated Querying:** **Trino** is configured with connectors to ClickHouse and Kafka. Users can connect to Trino using a standard SQL client to run complex analytical queries against the replicated data in ClickHouse.

## Technology Stack

| Component               | Technology                                             | Purpose                                                      |
| ----------------------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| **Source Database**     | Microsoft SQL Server 2019                              | The transactional database (OLTP)                            |
| **CDC Platform**        | Debezium SQL Server Connector                          | Captures and streams database changes                        |
| **Message Broker**      | Apache Kafka                                           | High-throughput, distributed event streaming platform        |
| **Data Warehouse**      | ClickHouse                                             | High-performance columnar database for analytics (OLAP)      |
| **Federated Querying**  | Trino (formerly PrestoSQL)                             | Query engine for running SQL queries on multiple data sources|
| **Containerization**    | Docker & Docker Compose                                | Building and orchestrating the multi-container application   |
| **Cluster Coordination**| Zookeeper                                              | Required for managing the Kafka cluster                      |

## Getting Started: Step-by-Step Guide

### Prerequisites

-   Git
-   Docker and Docker Compose
-   A command-line tool capable of making HTTP requests, like `cURL` or `Postman`.

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-username/your-repository-name.git
cd your-repository-name
```

### Step 2: Launch the Docker Environment

This command will start all the required services in detached mode.

```bash
docker-compose up -d
```
Wait for a couple of minutes for all services, especially SQL Server and Kafka Connect, to initialize fully.

### Step 3: Configure the Debezium SQL Server Connector

Post the connector configuration to the Kafka Connect REST API. This tells Debezium to start monitoring our SQL Server instance.

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{
  "name": "sqlserver-connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "database.hostname": "sqlserver_cdc",
    "database.port": "1433",
    "database.user": "sa",
    "database.password": "StrongP@ssw0rd",
    "database.dbname": "TestDB",
    "database.server.name": "sqlserver",
    "table.include.list": "dbo.Customers",
    "database.history.kafka.bootstrap.servers": "kafka_cdc:29092",
    "database.history.kafka.topic": "schema-changes.testdb",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false"
  }
}'
```
To verify the connector is running, use: `curl http://localhost:8083/connectors/sqlserver-connector/status`

### Step 4: Set Up the SQL Server Database and Enable CDC

Connect to the SQL Server CLI and run the necessary SQL commands.

```bash
# Connect to SQL Server CLI
docker exec -it sqlserver_cdc /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P StrongP@ssw0rd

# Inside the sqlcmd shell, run the following:
USE TestDB;
GO

-- Create the table to be tracked
CREATE TABLE Customers (
  id INT IDENTITY(1,1) PRIMARY KEY,
  name NVARCHAR(100),
  email NVARCHAR(100),
  created_at DATETIME2 DEFAULT GETUTCDATE()
);
GO

-- Enable CDC on the database
EXEC sys.sp_cdc_enable_db;
GO

-- Enable CDC on the Customers table
EXEC sys.sp_cdc_enable_table
  @source_schema = 'dbo',
  @source_name = 'Customers',
  @role_name = NULL;
GO

-- Exit the shell
EXIT
```

### Step 5: Set Up the ClickHouse Tables

Connect to the ClickHouse client to create the Kafka engine table, the materialized view, and the final destination table.

```bash
# Connect to ClickHouse CLI
docker exec -it clickhouse_cdc clickhouse-client

# Inside the client, run these SQL statements:

-- 1. Create a table that reads from the Kafka topic
CREATE TABLE testdb.kafka_customers (
    id UInt32,
    name String,
    email String,
    created_at UInt64
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka_cdc:29092',
    kafka_topic_list = 'sqlserver.dbo.Customers',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow';

-- 2. Create the final destination table with MergeTree engine
CREATE TABLE testdb.customers (
    id UInt32,
    name String,
    email String,
    created_at DateTime64(3),
    op String,
    source_ts_ms DateTime64(3),
    ingestion_time DateTime64(3) DEFAULT now()
) ENGINE = MergeTree()
ORDER BY id;

-- 3. Create a materialized view to automatically move and transform data
CREATE MATERIALIZED VIEW testdb.customers_mv TO testdb.customers AS
SELECT
    id,
    name,
    email,
    toDateTime64(CAST(created_at, 'UInt64') / 1000, 3) AS created_at,
    -- Add other metadata fields from Debezium payload if needed
    '' AS op,
    now() AS source_ts_ms
FROM testdb.kafka_customers;
```

## Testing the End-to-End Pipeline

Now that the setup is complete, let's test the flow with some DML operations.

### Step 1: Perform Database Operations in SQL Server

```bash
# Connect back to the SQL Server CLI
docker exec -it sqlserver_cdc /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P StrongP@ssw0rd

# Run DML statements
USE TestDB;
GO
INSERT INTO Customers (name, email) VALUES ('John Doe', 'john.doe@example.com');
INSERT INTO Customers (name, email) VALUES ('Jane Smith', 'jane.smith@example.com');
GO
UPDATE Customers SET email = 'john.d.updated@example.com' WHERE id = 1;
GO
DELETE FROM Customers WHERE id = 2;
GO
EXIT
```
<img width="1253" height="72" alt="image" src="https://github.com/user-attachments/assets/a07bb1b6-430f-48ba-b843-5b5a81f2ed6c" />
<img width="1253" height="453" alt="image" src="https://github.com/user-attachments/assets/9f5fdb37-b27f-45e4-b6cd-d5c5168728ee" />
<img width="1237" height="133" alt="image" src="https://github.com/user-attachments/assets/5495cb56-0707-4e1b-ada8-6fc67c3ee6af" />
<img width="1238" height="691" alt="image" src="https://github.com/user-attachments/assets/2d05c123-49cb-43c2-910c-d8d11c477823" />
<img width="1010" height="128" alt="image" src="https://github.com/user-attachments/assets/e0958c49-104f-4b4b-a63f-f0f44b414b92" />
<img width="1255" height="346" alt="image" src="https://github.com/user-attachments/assets/2ca47d62-83f0-4f86-8d5a-370a03bc3e57" />


### Step 2: Observe Change Events in Kafka

You can see the raw Debezium JSON events in the Kafka topic.

```bash
docker exec -it kafka_cdc kafka-console-consumer --bootstrap-server localhost:9092 --topic sqlserver.dbo.Customers --from-beginning --timeout-ms 5000
```
You will see structured JSON messages detailing each `INSERT`, `UPDATE`, and `DELETE`.
![Yeni Microsoft Word Belgesi Image 44](https://github.com/user-attachments/assets/23b073b2-64c9-4585-a500-bf4cbc4938ff)



### Step 3: Query the Replicated Data in ClickHouse

Connect to Trino to query the final, structured data in the ClickHouse `customers` table.

```bash
# Connect to Trino CLI
docker exec -it trino_cdc trino

# Inside the Trino shell
USE clickhouse.testdb;
SELECT * FROM customers;
```

You should see the final state of your data:
-   The inserted record for 'John Doe' with its updated email.
-   The record for 'Jane Smith' will be absent, as it was deleted.
<img width="1248" height="627" alt="image" src="https://github.com/user-attachments/assets/b74acb7c-b62f-4589-95f3-0bf13eb58c89" />
-----------------------------------------------------------------------------------------------------------------
<img width="1245" height="479" alt="image" src="https://github.com/user-attachments/assets/0b630627-9faf-4258-bc72-e6e82c3001af" />



## Shutting Down the Environment

To stop and remove all running containers, networks, and volumes, execute:

```bash
docker-compose down -v
```
The `-v` flag is important as it removes the Docker volumes, ensuring a clean state for the next run. Omit it if you wish to preserve the data in SQL Server and ClickHouse.
