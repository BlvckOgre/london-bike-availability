# 🚲 Real-Time London Bike Availability Monitoring using Microsoft Fabric

## 📌 Project Overview

This project aims to build a **real-time data streaming pipeline** to monitor the **availability of bikes across London**. Leveraging the **sample streaming data available in Microsoft Fabric**, we ingest, process, and analyze live bike station data using the **Eventstream (Eventhouse)** platform. The pipeline is structured using the **Medallion Architecture** to ensure data quality, scalability, and easy downstream consumption.

Key technologies used include:

- **Microsoft Fabric Eventstream** for real-time ingestion
- **KQL (Kusto Query Language)** for stream transformation and querying
- **Notebooks** for batch and stream processing
- **Power BI** for building insightful real-time dashboards and analytics

---

## 🚨 Problem Statement

Public bike-sharing systems are essential for sustainable urban transportation. However, the availability of bikes at different stations can fluctuate rapidly throughout the day, influenced by demand, traffic, and weather conditions. 

**Problem:**  
*How can we track and analyze the real-time availability of bikes across London to improve resource distribution and enhance user experience?*

---

## 🧱 Architecture Overview

We adopt the **Medallion Architecture** to organize our data processing pipeline into three layers:

### Eventstream Pipeline Steps

1. **Create Eventhouse** – First, an Eventhouse is created in Microsoft Fabric.
2. **Add Event Stream** – A streaming source of bike availability is added to the pipeline.
3. **Branch the Pipeline** into 3 separate branches:
   - **Branch 1: `Manage Fields` Transformation**
     - Adds fields: `timestamp`, `bikepoint`, and `street coordinates`.
     - Sends transformed data to **KQL scripts** for further processing using the **Medallion Architecture**:
       - `bronzelayer.kql`
       - `silverlayer.kql`
       - `goldlayer.kql`
   - **Branch 2: Filter Transformation**
     - Applies a filter by `Neighbourhood = "Chelsea"`.
     - Then creates a **Derived Stream** from the filtered data.
   - **Branch 3: Activator**
     - Adds an activator that triggers alerts for monitoring critical events (e.g., no bikes available).

---

## 🧪 Medallion Architecture: KQL Processing

### 🪙 Bronze Layer

```kql
// Moving BikeRawData Table to Bronze Folder
.alter table BikesRawData (
  TimeStamp: datetime,
  BikepointID: string,
  Street: string,
  Neighbourhood: string,
  Latitue: dynamic,
  Longitude: dynamic,
  No_Bikes: long,
  No_Empty_Docks: long
) with (folder = "Bronze")
```

---

### 🥈 Silver Layer

```kql
// Adhoc Query for Analysis
.set-or-replace BikesTransformedData <|
BikesRawData
| parse BikepointID with * "BikePoints_" BikepointID:int
| extend BikesToBeFilled = No_Empty_Docks - No_Bikes
| extend Action = iff(BikesToBeFilled > 0, tostring(BikesToBeFilled), "NA")

// Create Function to Transform Raw Data
.create-or-alter function with (docstring = "Transforms Raw Bike Data", folder = "SilverLayer") TransformBikeData() {
  BikesRawData
  | parse BikepointID with * "Bikepoints_" BikepointID:int
  | extend BikesToBeFilled = No_Empty_Docks - No_Bikes
  | extend Action = iff(BikesToBeFilled > 0, tostring(BikesToBeFilled), "NA")
}

// Update Policy for Silver Table
.alter table BikeTranformedData policy update
'''[{
  "IsEnabled": true,
  "Source": "BikesRawData",
  "Query": "TransformBikeData()",
  "IsTransactional": false,
  "PropagateIngestionProperties": false
}]'''

// Move Silver Table to Silver Folder
.alter table BikesTransformedData (
  TimeStamp: datetime,
  BikepointID: int,
  Street: string,
  Neighbourhood: string,
  Latitue: dynamic,
  Longitude: dynamic,
  No_Bikes: long,
  No_Empty_Docks: long,
  BikesToBeFilled: long,
  Action: string
) with (folder = "Silver")
```

---

### 🥇 Gold Layer

```kql
// Create Materialized View for Gold Layer
.create-or-alter materialized-view with (folder="Gold") AggregatedData on table
{
  BikeTranformedData
  | summarize arg_max(TimeStamp, No_Bikes) by BikepointID
}
```

---

## 📊 Sample Visualization Query

```kql
AggregatedData
    | sort by BikepointID
    | render columnchart with (ycolum=No_Bikes,xcolum=BikrpointID)

BikesRawData
    | where Neighbourhood == "Chelsea"
    | summarize max(No_Bikes) by Street

BikesRawData
    | where TimeStamp > ago(5m)
```

---

## 🔧 Tools & Technologies

| Tool | Description |
|------|-------------|
| **Microsoft Fabric** | End-to-end platform to build and deploy data pipelines |
| **Eventstream (Eventhouse)** | Real-time data ingestion from streaming sources |
| **KQL (Kusto Query Language)** | Powerful language for streaming queries and transformations |
| **Microsoft Fabric Notebooks** | Used to process and clean data at various stages |
| **Power BI** | Real-time dashboards for insights and visualization |
| **Medallion Architecture** | Bronze → Silver → Gold layering strategy for structured data processing |

---

## 📁 Project Structure

```
/London-Bike-Streaming/
│
├── notebooks/
│   ├── bronze_ingest.ipynb
│   ├── silver_transform.ipynb
│   └── gold_aggregation.ipynb
│
├── dashboards/
│   └── powerbi_bike_availability.pbix
│
├── kql/
│   ├── bronzelayer.kql
│   ├── silverlayer.kql
│   ├── goldlayer.kql
│   └── queries.kql
│
├── README.md
└── assets/
    └── architecture_diagram.png
```

---

## 📬 Contact

For questions, collaboration, or suggestions:  
**📧 Email:** yourname@example.com  
**🔗 LinkedIn:** [linkedin.com/in/your-profile](https://linkedin.com/in/your-profile)
