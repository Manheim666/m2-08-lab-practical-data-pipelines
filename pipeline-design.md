## Page 1

mermaid
graph TD;
    subgraph Data Ingestion Process;
        BatchFile[Batch File Online] --> Batch[Batch];
        LiveTransaction[Live Transaction] --> Stream[Stream];
        Batch --> IngestionLayer[Ingestion Layer format];
        Stream --> IngestionLayer;
        IngestionLayer --> RawDataLayer[RAW Data Layer];
        RawDataLayer --> ValidationLayer[Validation schema +];
        ValidationLayer --> CleanData[Clean Data];
        CleanData --> TransformationLayer[Transformation +];
        TransformationLayer --> FeatureEngineering[Feature];
        FeatureEngineering --> BusinessIntelligence[BI];
        FeatureEngineering --> MachineLearning[ML];
    end;

---


## Page 2

The pipeline processes transactional data coming from two different sources. The first source is a historical dataset (Online Retail.xlsx) which contains past transactions. This file will be loaded once at the beginning to initialize the system. The second source is a live stream of transactions, which simulates new orders arriving one by one in real time.

Both sources go through an ingestion layer where the data gets normalized into a consistent format. After that, the data is stored in a raw data layer (also called a landing zone). This layer keeps the original records before any processing happens, which is useful in case we need to reprocess the data later.

Next, a validation stage checks each record to make sure it follows the schema and business rules. Records that pass validation move into the clean data layer, while records that fail validation are sent to a quarantine or dead letter area for investigation.

Once the data is validated and clean, a transformation stage computes derived fields and aggregations. The processed results are stored in a feature layer which contains datasets used for analytics and machine learning.

Finally, the processed data is consumed by two systems: a BI dashboard that shows sales analytics, and a machine learning model that predicts whether a customer is likely to become high value.

Monitoring runs across the entire pipeline to track data quality, schema consistency, and system health.

---


## Page 3

# Historical Batch File

The historical dataset is the Online Retail.xlsx file which contains all past transactions. This file is used to bootstrap the pipeline and load historical data into the system. A batch ingestion job reads the file and converts it into a more efficient format before sending it to the ingestion layer.

# Live Transaction Stream

The live stream simulates new orders coming from an online store in real time. Each record represents a single transaction event. These records are received by a stream listener that forwards them into the pipeline.

# Ingestion Layer

The ingestion layer receives data from both the batch loader and the live stream. Its job is to standardize the records so they follow the same structure. It may also add metadata such as ingestion timestamps or source identifiers. The output of this layer is consistent records written into the raw data storage.

# Raw Data Layer

The raw layer acts as a landing zone where incoming data is stored exactly as it arrives. This is basically a backup of the original data. Storing the raw data makes the system more reliable since we can reprocess the pipeline if something breaks later. The data here is stored in Parquet format and partitioned by date.

# Validation Stage

The validation stage checks each record for correctness. This includes verifying that fields have the correct type, required columns exist, and values fall within reasonable ranges. If a record passes validation it moves forward in the pipeline, otherwise it is redirected to a quarantine area.

# Quarantine / Dead Letter Zone

Invalid records are stored in the quarantine zone instead of being deleted. This allows engineers to inspect them and understand what went wrong. Each record is stored together with an error message explaining the validation failure.

# Clean Data Layer

---


## Page 4

The clean data layer stores all validated transaction records. At this point the data should be consistent and reliable. Analysts and other systems can safely query this dataset without worrying about broken records.

**Transformation Stage**

The transformation stage enriches the clean data. Derived columns such as line totals and cancellation flags are calculated here. The pipeline also performs aggregations to compute customer-level statistics.

**Feature Data Layer**

The feature layer contains aggregated datasets used by machine learning models. These features include metrics like total spending, number of orders, and recency of purchases.

**BI Dashboard Consumer**

The BI dashboard reads processed data and generates visual reports. These reports include metrics like daily revenue, best selling products, and country-level sales breakdowns.

**ML Model Consumer**

The machine learning model uses customer-level features to predict whether a customer is likely to become high value. These predictions can be used for marketing or recommendation systems.

**Monitoring System**

Monitoring tools track pipeline health and performance. Metrics such as data freshness, validation error rate, and data volume are collected continuously. If something unusual happens (for example a sudden drop in transactions), alerts are triggered so engineers can investigate.

---


## Page 5

# Schema Validation

Each record must follow the expected structure:

*   InvoiceNo must be a non empty string
*   StockCode must be a string
*   Quantity must be an integer
*   UnitPrice must be numeric
*   InvoiceDate must be a valid timestamp
*   Country must be a string
*   CustomerID can be missing but if present it must be numeric

# Value Range Validation

These checks make sure values are realistic:

*   Quantity cannot be zero
*   UnitPrice must be greater than zero
*   InvoiceDate cannot be in the future
*   CustomerID must be positive if present

# Business Rule Validation

These rules depend on the business logic of the retailer:

*   If the invoice number starts with "C" it indicates a cancellation
*   Cancellation transactions should have negative quantity
*   Normal purchases should have positive quantity
*   A single invoice must contain at least one product
*   Total order value should not exceed a certain limit

If a record fails schema validation it is rejected and written to the dead letter queue. These records usually indicate structural problems like missing fields or incorrect types.

If a record fails value range checks it is sent to the quarantine zone. These might be unusual values that require manual inspection.

If a business rule fails the record is also quarantined but flagged differently since it may indicate a data entry mistake or suspicious transaction.

Operators are notified if validation failure rates exceed a certain threshold.

Once the root cause of the problem is fixed, quarantined records can be reprocessed by feeding them back into the validation stage.

---


## Page 6

# Cleaning

Input: validated transaction data  
Output: cleaned transaction dataset  

Cleaning includes removing duplicates, fixing inconsistent formatting, and normalizing country names.

# Derived Columns

Some new fields are calculated from the existing ones:

* LineTotal = Quantity × UnitPrice
* IsCancellation
* TransactionYear
* TransactionMonth
* TransactionDay

# Customer Aggregations

The pipeline calculates customer level statistics such as:

* total revenue per customer
* total number of orders
* number of unique products purchased
* days since last purchase

# ML Feature Engineering

The aggregated data is converted into features used by the ML model:

* purchase frequency
* average order value
* product diversity
* customer recency

---


## Page 7

<table>
  <thead>
    <tr>
      <th>Layer</th>
      <th>Contents</th>
      <th>Format</th>
      <th>Update Frequency</th>
      <th>Retention</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Raw</td>
      <td>Original records</td>
      <td>Parquet</td>
      <td>Continuous</td>
      <td>Permanent</td>
    </tr>
    <tr>
      <td>Clean</td>
      <td>Valid records</td>
      <td>Parquet</td>
      <td>Daily</td>
      <td>Long Term</td>
    </tr>
    <tr>
      <td>Feature</td>
      <td>Aggregated features</td>
      <td>Parquet/Table</td>
      <td>Daily</td>
      <td>1-2 years</td>
    </tr>
  </tbody>
</table>

The raw layer stores the original data for traceability and recovery.
The clean layer contains reliable transaction data that can be queried by analysts.
The feature layer contains aggregated datasets optimized for machine learning.

The pipeline processes new data incrementally using a timestamp checkpoint. Each run processes only records that arrived after the last successful run.

Late arriving records are allowed within a small time window. If late data appears, the affected partitions are recomputed.

Customer features are refreshed daily, while the machine learning model is retrained once per week using the latest feature dataset.
