---
layout: post
title: "Building a Scalable Real-Time Data Pipeline with Event Hub, Stream Analytics, Parquet, ADLS, and ADX"
date: 2024-06-19 21:00:00 +0300
categories: data azure
tags: ADLS Parquet Stream-Analytics-Job EventHub ADX
---

In today's data-driven world, the ability to ingest, process, and analyze data in real-time is crucial for making timely and informed decisions. Azure provides a suite of powerful tools that can help you build a scalable and efficient real-time data pipeline. Combining Azure Event Hub, Stream Analytics, Parquet, Azure Data Lake Storage (ADLS), and Azure Data Explorer (ADX) can create a robust solution for real-time data ingestion and analytics.

## Components Overview

![Components Overview](/assets/images/real-time-data-pipeline/Components.jpg)
_Participating Components_

### Azure Event Hub
[Azure Event Hub](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-about){:target="_blank"} is a big data streaming platform and event ingestion service capable of receiving and processing millions of events per second. It acts as the entry point for your real-time data pipeline, capturing data from various sources.

### Azure Stream Analytics
[Azure Stream Analytics](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction){:target="_blank"} is a fully managed, real-time data stream processing service provided by Microsoft Azure. It allows you to perform real-time analytics on multiple streams of data from sources such as IoT devices, social media, applications, and more. With its SQL-like query language, you can easily filter, aggregate, and transform streaming data to gain valuable insights. Key features of Stream Analytics: 

- **Real-Time Data Processing**: Azure Stream Analytics can process millions of events per second with low latency, enabling data analysis in real-time and quick decision making.
2. **SQL-Like Query Language**: The service uses a SQL-like query language, making it easy for developers and data analysts to write complex queries without needing to learn a new language.
3. **Integration with Azure Services**: Azure Stream Analytics seamlessly integrates with various Azure services, including Azure Event Hubs, Azure IoT Hub, Azure Blob Storage, Azure Data Lake Storage, and Azure SQL Database, making it easy to ingest, process, and store data.
4. **Scalability**: The service is designed to scale automatically based on the volume of incoming data, ensuring that your data processing pipeline can handle varying loads without manual intervention.
5. **Built-In Machine Learning**: Azure Stream Analytics supports built-in machine learning models, allowing you to apply predictive analytics and anomaly detection to your streaming data.
6. **Reliability and Security**: Azure Stream Analytics offers enterprise-grade reliability and security features, including data encryption, role-based access control, and compliance with industry standards.

### Parquet Files
[Parquet](https://parquet.apache.org/docs/){:target="_blank"} is an open-source, columnar storage file format designed to bring efficiency compared to row-based files like CSV or JSON. It was developed by Cloudera and Twitter and is now an open-source project under the Apache Software Foundation. Key features of Parquet files:

- **Columnar Storage**: Unlike traditional row-based storage formats, Parquet stores data in columns, which allows for efficient data compression and encoding schemes, reducing storage space. 
- **Efficient Compression**: Parquet supports various compression algorithms (e.g., Snappy, Gzip, LZO), which can significantly reduce the storage footprint of your data. Better compression ratios means less I/O and faster data scans, which can be significant in cloud environments.
- **Efficient Query Performance**: Columnar storage enables faster read times for analytical queries that only need a subset of columns.
- **Schema Evolution**: Parquet supports schema evolution, allowing you to add new columns without breaking existing data.

### Azure Data Lake Storage (ADLS) 
[Azure Data Lake Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction){:target="_blank"} is a scalable and secure data lake for high-performance analytics workloads. It is designed to handle large volumes of data, making it ideal for big data analytics. ADLS is part of the Azure Storage suite and comes in two generations: ADLS Gen1 and ADLS Gen2. The latter is the more recent version and offers enhanced features and better integration with other Azure services. Key features of ADLS:

- **Scalability**: ADLS can scale to exabytes of data, making it suitable for large-scale data analytics.
- **Security**: It offers enterprise-grade security features, including encryption at rest and in transit, as well as fine-grained access control.
- **Integration**: Seamlessly integrates with various Azure services like Azure Databricks, Azure Synapse Analytics, and Azure HDInsight.
- **Performance**: Optimized for high-performance analytics workloads, ensuring fast data retrieval and processing.
- **Cost-Effective**: Pay-as-you-go pricing model ensures you only pay for what you use.

### Azure Data Explorer (ADX)
[Azure Data Explorer](https://azure.microsoft.com/en-us/products/data-explorer){:target="_blank"} is a fast and highly scalable data exploration service for log and telemetry data. It allows you to run complex queries on large datasets in near real-time.

## Environment Setup

For the purpose of this demo, we will reuse some components from the [previous blog post](https://helnik.github.io/posts/Ingest-and-route-data-from-EventHub-to-ADX/){:target="_blank"}.

### Event Producer Code
Clone the console application from the [github](https://github.com/helnik/EventHubSamples/tree/master/GzipFeeder){:target="_blank"} repo. 

>After creating the Event Hub, fill in the `EvhConnectionString` and `EvhName` in the `appsettings.json` file.
{: .prompt-info }

### Event Hub Creation 
Follow the steps described in the [previous blog post](https://helnik.github.io/posts/Ingest-and-route-data-from-EventHub-to-ADX/#event-hub-creation){:target="_blank"}. 

### ADLS Creation
- To create an [ADLS Gen2](https://learn.microsoft.com/en-us/azure/storage/blobs/create-data-lake-storage-account) storage account, in the Azure Portal search for `Storage Accounts` select it and in the upper left corner select `+ Create`.

    ![New Storage Account Gen2](/assets/images/real-time-data-pipeline/1.SACreate.jpg)
_New Storage Account Gen2 creation_

- Once deployment is complete, select it, find `Containers` and in the upper left corner select `+ Container`. For this example we will create two (2) new containers, named "car" and "user".  

    ![New Container](/assets/images/real-time-data-pipeline/2.ContainerCreate.jpg)
_Container creation_

  > Based on the needs, one container can be created and later on use paths. For example, create container named data_raw and then specify paths /car and /user.
  {: .prompt-info }

### Azure Stream Analytics Job Creation
In the Azure Portal search for `Stream Analytics jobs` select it and in the upper left corner select `+ Create`

![New Stream Analytics job](/assets/images/real-time-data-pipeline/3.ASACreate.jpg)
    _Stream Analytics job creation_

![Assign Managed Identity](/assets/images/real-time-data-pipeline/4.ASAManagedIdentity.jpg)
    _Add System Assigned Identity_

#### Azure Stream Analytics Job Setup
Once deployment is complete go to resource and you will see your new job.

- **Configure Inputs**: <br>
[Inputs](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-add-inputs){:target="_blank"} in Azure Stream Analytics Jobs are the entry points for your streaming data. They define where the data is coming from and how it will be ingested into the Stream Analytics Job for processing. Azure Stream Analytics supports a variety of input sources, making it flexible and versatile for different real-time data scenarios.<br>
Select `Inputs` and in the upper left corner select `+Add input`. From the dropdown list select `Event Hub`. <br>
Fill in the necessary values. For our example pay attention to the following:
    - **Input Alias**: The alias of our input. Will be used in the query later.
    - Select the **Subscription** and the created **Event Hub Namespace** and **Event Hub**.
    - Create a new **Consumer group** or select one you have created in previous step.
    - **Authentication Mode**: Select `Managed Identity: System Assigned`
    - **Event Serialization Format**: JSON
    - **Event Compression Type**: GZIP

![Create Input](/assets/images/real-time-data-pipeline/5.ASAInput.jpg)
    _Input creation_

- **Configure Outputs**: <br>
[Outputs](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-define-outputs){:target="_blank"} in Azure Stream Analytics Jobs are the destinations where the processed and analyzed data is sent. They define how the results of your Stream Analytics queries are stored, visualized, or further processed. Azure Stream Analytics supports a variety of output destinations, making it flexible and versatile for different real-time data scenarios.<br>
Select `Outputs` and in the upper left corner select `+Add input`. From the dropdown list select `Event Hub`. <br>
Fill in the necessary values. For our example pay attention to the following:
    - **Output Alias**: The alias of our output. Will be used in the query later.
    - Select the **Subscription** and the created **Storage Account** and **Container**.
    - Create a new **Consumer group** or select one you have created in previous step.
    - **Authentication Mode**: Select `Managed Identity: System Assigned`
    - **Write Mode**: Select ["Once, when all results for the time partition are available. Ensures exactly once delivery (preview)"](https://learn.microsoft.com/en-us/azure/stream-analytics/blob-storage-azure-data-lake-gen2-output#exactly-once-delivery-public-preview){:target="_blank"}.
    - **Path Pattern**: Since `Once` was selected for write mode, Path Pattern becomes mandatory. For our example we will use `{date}/{time}`. <br>
For our example we will create two (2) outputs one for each container created above. 

> If one container was created in the storage then here we will change `Path Pattern` to "user\\{date}\\{time}" and "car\\{date}\\{time}"
  {: .prompt-info }

![Create Output](/assets/images/real-time-data-pipeline/6.ASAOutput.jpg)
_Output creation_

- **Query**: <br>
One of the core components of Azure Stream Analytics is the query language, which is based on a subset of SQL. This allows users to write powerful queries to process and analyze streaming data. <br>
In our example we want to route to different container based on our custom input [property](https://learn.microsoft.com/en-us/dotnet/api/azure.messaging.eventhubs.eventdata.properties?view=azure-dotnet){:target="_blank"}. To achieve this we will leverage [GetMetadataPropertyValue](https://learn.microsoft.com/en-us/stream-analytics-query/getmetadatapropertyvalue?toc=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Fstream-analytics%2Ftoc.json&bc=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json#limitations-and-restrictions){:target="_blank"} which is a function that allows you to retrieve metadata properties from the input data stream. This function is particularly useful when you need to access system-generated metadata, such as event timestamps, partition keys, or other properties that are not part of the actual data payload. <br> 
    - Common Metadata Properties:
EventEnqueuedUtcTime: The UTC time when the event was enqueued.<br>
EventProcessedUtcTime: The UTC time when the event was processed. <br>
PartitionId: The partition ID of the event. <br>
Offset: The offset of the event in the partition. 

Select Query and paste the following:

``` sql
WITH StreamData AS (
    SELECT *, 
    GetMetadataPropertyValue(InputEvh, '[User].[Table]') AS TargetTable --Table is our custom property. We use the path [User].[PropertyName] to access  it
    FROM  InputEvh  --InputEvh is the alias we set when creating the Input
    ) 
 
SELECT * INTO [car] FROM StreamData WHERE TargetTable = 'Car' --car is the alias set when creating the output
SELECT * INTO [user] FROM StreamData WHERE TargetTable = 'User' --user is the alias set when creating the output
```
Start the Console application to send some test data into the Event Hub. In the Query select `Refresh` to see a sample preview of the data.

![Sample Preview](/assets/images/real-time-data-pipeline/7.ASAQuery.jpg)
_Sample Preview_

- **Start the job**: <br>
Having everything in place, from the upper left, select `Start Job`. Soon, Parquet files will start popping in the containers.

### Use ADX to view and query Parquet Data
ADX has introduced the concept of [external tables](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/schema-entities/external-tables){:target="_blank"} which allow you to query data that resides outside of the ADX database. This can include data stored in various external storage systems such as Azure Blob Storage, Azure Data Lake Storage, and even other databases. By using external tables, you can seamlessly integrate and analyze data from multiple sources without the need to import it into ADX. <br>

Key Benefits of External Tables:
- **Data Integration**: Easily integrate and query data from different storage systems.
- **Cost Efficiency**: Avoid the costs associated with data ingestion and storage within ADX.
- **Flexibility**: Query data in its original location, which is useful for scenarios where data is frequently updated or where you need to maintain a single source of truth.
- **Performance**: Leverage the powerful query engine of ADX to perform complex analytics on external data.

#### Create External tables
To [create an external](https://learn.microsoft.com/en-us/azure/data-explorer/external-table){:target="_blank"} table in ADX from the Azure portal, navigate to your cluster, select your Database, right click and select `Create external table`.

![External table](/assets/images/real-time-data-pipeline/8.ADXCreateExternalTable.jpg)
_External table creation_

 > When creating the external table for first time you  will need to grant the `Storage Blob Data Reader` role assignment.
{: .prompt-tip }

Select the container that corresponds to the table we want to create. 

> File filter can be used to specify specific files. For our example if the approach with one container with paths was selected, then we can specify the paths (i.e. car/, /user).
  {: .prompt-info }

![External table creation](/assets/images/real-time-data-pipeline/9.ADXSelectContainer.jpg)
_External table container selection_

Table schema will be determined automatically. It can be altered according to our needs (e.g. change data types or delete specific columns). 

![External table creation schema](/assets/images/real-time-data-pipeline/10.ADXTransformSchema.jpg)
_External table schema definition_

Once done, tables will be visible under the `External Tables` folder. We can query them using the `external_table("TableName")` command.

![External table query](/assets/images/real-time-data-pipeline/11.ADXQuery.jpg)
_External table query_

## Conclusion 
 
By leveraging Azure Event Hub, Azure Stream Analytics, Parquet format, Azure Data Lake Storage, and Azure Data Explorer, you can easily create a robust and efficient pipeline (with zero code) that meets your real-time data processing needs.

This architecture not only ensures scalability and performance but also provides flexibility in querying and analyzing data. Whether you're monitoring IoT devices, analyzing financial transactions, or processing social media feeds, this pipeline can be adapted to suit various use cases.
