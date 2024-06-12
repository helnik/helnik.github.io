---
layout: post
title: "Ingest and route compressed data from EventHub to ADX"
date: 2024-06-12 17:00:00 +0300
categories: data azure
tags: Azure EventHub ADX C# GZIP
---

[Azure Data Explorer](https://azure.microsoft.com/en-us/products/data-explorer){:target="_blank"} a.k.a ADX, a not so well known, data analytics service designed to help users unlock valuable insights from vast amounts of raw data. Built by Microsoft, Azure Data Explorer (ADX) is a powerful, fast, and highly scalable tool that excels in real-time and complex event processing, enabling users to analyze large volumes of structured, semi-structured, and unstructured data with exceptional speed and efficiency. Its integration with a wide array of Azure services makes it a versatile and invaluable asset for any data-centric enterprise.

## The Goal (Simplified)

Continuously extracted data from multiple tables in an on-premise SQL Server database needs to be ingested into Azure Data Explorer (ADX). The initial step involves ensuring that ADX mirrors the same schema as the on-premise SQL Server. Data should be sent compressed via Event Hub.

## Environment Setup 

### Event Hub Creation 

[Azure Event Hub](https://azure.microsoft.com/en-gb/products/event-hubs/){:target="_blank"} is a highly scalable data streaming platform and event ingestion service designed to handle millions of events per second, making it an ideal solution for big data and real-time analytics. By acting as a buffer between event producers and event consumers, EventHub effectively decouples the process of information sending. This decoupling allows producers to send data without worrying about the readiness or availability of consumers, and vice versa. Producers can continuously push data into EventHub, where it is temporarily stored and made available for consumers to process at their own pace. This architecture not only enhances system reliability and scalability but also simplifies the development of complex data processing pipelines, enabling seamless integration across various applications and services.

Creation is straight forward from Azure Portal

- Create an [Event Hub Namespace](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-create){:target="_blank"}. <br>
Search for [Event Hubs](https://portal.azure.com/#browse/Microsoft.EventHub%2Fnamespaces){:target="_blank"} and in the upper left corner select `+ Create` to create a new Namespace.
![New Event Hub Namespace](/assets/images/evh-to-adx/1.EvhCreateNamespace.jpg)
_New Event Hub Namespace creation_

  > Select Standard pricing tier or above, since a dedicated consumer group is needed
  {: .prompt-tip }

- When deployment is complete, go to the resource and the upper left corner select `+ Event Hub`.
![New Event Hub](/assets/images/evh-to-adx/2.EvhCreate.jpg)
_New Event Hub creation_

- Create a new Consumer group. <br>
Select the newly created Event Hub (having your Namespace go to `Entities -> Event Hubs` and select your Event Hub in the new window). 
Select the `+ Consumer group` to create a new Consumer group. 
![New Event Hub Namespace](/assets/images/evh-to-adx/3.EvhCreateCg.jpg)
_New Consumer group creation_

  From the [docs](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-about#key-architecture-components){:target="_blank"}: 
  >**Consumer group**: This logical group of consumer instances reads data from an event hub or Kafka topic. It enables multiple consumers to read the same streaming data in an event hub independently at their own pace and with their own offsets.

### ADX Creation 

- [Create an ADX Cluster](https://learn.microsoft.com/en-us/azure/data-explorer/create-cluster-and-database?tabs=free){:target="_blank"}. <br>
Search for [Data Explorer](https://portal.azure.com/#browse/Microsoft.Kusto%2Fclusters){:target="_blank"} and in the upper left corner select `+ Create` to create a new Cluster.
![New Azure Data Explorer Cluster](/assets/images/evh-to-adx/1.AdxCreateCluster.jpg)
_New Cluster creation_

- When deployment is complete, go to the resource and the upper left corner select `+ Add Database`. <br>
![New Database](/assets/images/evh-to-adx/2.AdxCreateDB.jpg)
_New Database creation_
When deployment is complete you will see your newly created Database by navigating to `Data -> Databases`. <br>
You can execute your queries by selecting and your Database and moving to the query section. You can also visit the [Azure Data Explorer portal](https://dataexplorer.azure.com/){:target="_blank"} and work with your cluster from there.

>Before creating the sample Tables there is another important concept called [Ingestion mappings](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/management/mappings){:target="_blank"}. 
>In a nutshell ingestion mappings provide a flexible way to handle diverse data formats, such as JSON, CSV, or Avro, ensuring that data is accurately parsed and aligned with the schema of the target table. By leveraging ingestion mappings, users can streamline the data ingestion process, reduce errors, and enhance the efficiency of their data analytics workflows, ultimately enabling more insightful and actionable business intelligence. For the purpose of this demo [JSON ingestion mappings](https://learn.microsoft.com/en-us/azure/data-explorer/ingest-json-formats?tabs=kusto-query-language){:target="_blank"} will be used.
{: .prompt-info }

- Create two new Tables on your Database. Right click your Database and select `Create table`. <br>
For convenience you can use the following commands that create the Tables and their corresponding mappings (run one after another): 

``` 
//1. Create User Table
.create table User (Name: string, Surname: string, Age: int, IsHuman: bool) 

//2. Create Car Table
.create table Car (Model: string, Year: int, Price: decimal) 

//3. Create User JSON mapping
.create table User ingestion json mapping "User_evh_json_mapping"
'[{"column":"Name","path":"$[\'Name\']","datatype":""},{"column":"Surname","path":"$[\'Surname\']","datatype":""},{"column":"Age","path":"$[\'Age\']","datatype":""},{"column":"IsHuman","path":"$[\'IsHuman\']","datatype":""}]'

//4. Create Car JSON mapping
.create table Car ingestion json mapping "Car_evh_json_mapping"
'[{"column":"Model","path":"$[\'Model\']","datatype":""},{"column":"Year","path":"$[\'Year\']","datatype":""},{"column":"Price","path":"$[\'Price\']","datatype":""}]'
```


## Streaming GZIP data from C#

- Create a producer client: <br>
The [Event Hub Producer Client in C#](https://learn.microsoft.com/en-us/dotnet/api/azure.messaging.eventhubs.producer.eventhubproducerclient?view=azure-dotnet){:target="_blank"}  allows developers to create and send batches of events, ensuring optimal performance and resource utilization. To get started with the Event Hub Producer Client in C#, you should have the Azure.Messaging.EventHubs NuGet package installed in your C# project. Familiarity with asynchronous programming in C# will also be beneficial, as the client leverages async methods to handle event publishing efficiently. With these prerequisites in place, you can begin integrating the Event Hub Producer Client into your applications to enable robust event streaming and data ingestion capabilities.

```c#
public class EvenHubFeeder : IFeed
{
    private EventHubProducerClient _producerClient;

    public EvenHubFeeder(IConfiguration configuration)
    {
        EventHubProducerClientOptions producerOptions = new EventHubProducerClientOptions
        {
            RetryOptions = new EventHubsRetryOptions
            {
                Mode = EventHubsRetryMode.Exponential,
                MaximumDelay = TimeSpan.FromMilliseconds(double.Parse(configuration["MaximumDelayInMs"])),
                MaximumRetries = int.Parse(configuration["MaximumRetries"])
            },
            ConnectionOptions = new EventHubConnectionOptions
            {
                Proxy = string.IsNullOrWhiteSpace(configuration["ProxyAddress"]) ? null : new WebProxy
                {
                    Address = new Uri(configuration["ProxyAddress"])
                }
            }
        };
        _producerClient = new EventHubProducerClient(configuration["EvhConnectionString"], configuration["EvhName"], producerOptions);
    }
}
```
 > `EventHubProducerClient` mimics the `HttpClient` pattern so as a best practice, when your application pushes events regularly you should cache and reuse the the `EventHubProducerClient` for the lifetime of your application. 
  {: .prompt-tip }

- Compress the payload data:<br>
`GZIP` in .NET is a widely-used compression algorithm that allows developers to efficiently compress and decompress data, reducing the size of files and streams for storage or transmission. The .NET framework provides built-in support for GZIP through classes such as `GZipStream` in the `System.IO.Compression` namespace. These classes enable developers to easily apply GZIP compression to data streams, making it straightforward to compress data before sending it over a network or to decompress data received from a compressed source. By leveraging GZIP in .NET, applications can achieve significant performance improvements in terms of bandwidth usage and storage efficiency, while maintaining compatibility with other systems that support the GZIP format.

```c#
private static byte[] CompressJsonData(string jsonData)
{
    byte[] byteArray = Encoding.UTF8.GetBytes(jsonData);

    using (var memoryStream = new MemoryStream())
    {
        using (var gzipStream = new GZipStream(memoryStream, CompressionLevel.Optimal))
        {
            gzipStream.Write(byteArray, 0, byteArray.Length);
        }
        return memoryStream.ToArray();
    }
}
```


- Create EventData to add to the batch:<br>
The `EventData` class in C# is a fundamental component of the `Azure.Messaging.EventHubs` library, designed to encapsulate the data payload that you wish to send to an Azure Event Hub. Each instance of `EventData` represents a single event, containing the event's body as a byte array, along with optional properties and system properties that can be used for metadata and **routing** purposes. 

```c#
private static EventData CreateEventDataFromChange<T>(T change, string tableName, string mappingName)
{
    string serializedChange = JsonConvert.SerializeObject(change);
    byte[] compressedPayloadBytes = CompressJsonData(serializedChange);
    var eventData = new EventData(compressedPayloadBytes);
    
    //properties required to route the data to ADX
    eventData.Properties.Add("Table", tableName); //must match the table name, case sensitive
    eventData.Properties.Add("Format", "JSON"); 
    eventData.Properties.Add("IngestionMappingReference", mappingName); //must match the json mapping, case sensitive
    return eventData;
}
```
>[System properties](https://learn.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview#event-system-properties-mapping){:target="_blank"} expansion is not supported on Event Hub ingestion of compressed messages.
{: .prompt-warning }

- Send batch data to Event Hub:<br>
The `EventDataBatch` class in C# is a crucial component of the `Azure.Messaging.EventHubs library`, designed to optimize the process of sending multiple events to an Azure Event Hub. This class allows developers to group a collection of EventData instances into a single, manageable batch, ensuring that the events are transmitted efficiently and within the size constraints imposed by the Event Hub service. By using `EventDataBatch`, you can maximize throughput and minimize the number of network operations required to send large volumes of event data. The class provides methods to add events to the batch while automatically checking if the batch size exceeds the allowable limit, thus preventing errors and ensuring smooth operation. Utilizing `EventDataBatch` is essential for applications that need to handle high-frequency event generation and transmission, making it a key tool for building scalable and performant event-driven solutions.

```c#
 public async Task FeedAsync<T>(List<T> payload, string tableName, string mappingName)
 {
     var eventBatch = await _producerClient.CreateBatchAsync();

     foreach (var change in payload)
     {
         EventData eventData = CreateEventDataFromChange(change, tableName, mappingName);

         //try add to the batch. If the batch cannot take up more then send 
         if (!eventBatch.TryAdd(eventData))
         {
             if (eventBatch.Count == 0) //if there are no other events this change is too large no way to ever send it this way
             {
                 continue;
             }

             await TrySendBatch(eventBatch);
             eventBatch = await _producerClient.CreateBatchAsync(); //we just send the batch so it has been closed, recreate it

             if (!eventBatch.TryAdd(eventData) && eventBatch.Count == 0) //add current change to batch since it was not send and check if this item is too big
             {
                 continue; //if it is too big we cannot send it anyway. Have a fallback mechanism here
             }
         }
     }
     //send all or the whatever left
     if (eventBatch != null)
     {
         await TrySendBatch(eventBatch);
     }
 }
```

>You can find the complete code available [here](https://github.com/helnik/EventHubSamples/tree/master/GzipFeeder){:target="_blank"}
{: .prompt-info }

### Executing the code

Upon code execution we can see that data are successfully sent to Event Hub, but querying our Tables will yield no results.
![Incoming Messages](/assets/images/evh-to-adx/EvhIncomingMessages.jpg)
_Successful Incoming Messages_

## Enabling automatic ingestion

ADX Data Connection is a powerful feature that enables seamless and continuous data ingestion from various sources. By integrating data from sources such as Azure Event Hubs, Azure IoT Hub, and Azure Blob Storage, ADX Data Connection ensures that your data is always up-to-date and ready for real-time analytics and complex querying. This integration streamlines the process of ingesting large volumes of diverse data, making it an essential tool for building efficient, scalable, and real-time analytics solutions.

To [create a Data Connection](https://learn.microsoft.com/en-us/azure/data-explorer/create-event-hubs-connection?tabs=get-data%2Cget-data-2){:target="_blank"} select your database then Data Connections and in the upper left corner `+ Add data connection`. From the drop down select `Event Hub`. 

![New Data Connection](/assets/images/evh-to-adx/AdxCreateDataConnection.jpg)
_Data Connection creation_

>**Ensure**
>1. GZIP is selected as data compression
>2. Data routing is Enabled
>3. System-assigned identity is selected
{: .prompt-tip }

In Event Hub we can now see Outgoing messages
![Outgoing Messages](/assets/images/evh-to-adx/EvhOutgoingMessages.jpg)
_Successful Outgoing Messages_

And in ADX we can see the ingested data
![Ingested Data](/assets/images/evh-to-adx/AdxIngestedData.jpg)
_Ingested Data_

>Useful commands: <br>
>.show ingestion failures //used to display detailed information about any data ingestion errors that have occurred <br>
>.show ingestion mappings //retrieves and displays the defined mappings for data ingestion
{: .prompt-tip }

>[Ingestion batching policy](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/management/batching-policy){:target="_blank"} is a crucial configuration that optimizes the data ingestion process by grouping multiple small data ingestion requests into larger, more efficient batches. This policy helps to enhance throughput, reduce latency, and minimize the overall cost of data ingestion by leveraging the system's ability to handle large volumes of data more effectively. By fine-tuning parameters such as batch size, maximum delay, and maximum number of items per batch, users can achieve a balanced and efficient data ingestion pipeline tailored to their specific workload requirements. 

>By default there is a 5 min delay. For **testing only purposes** we can set up the ingestion to 10 seconds using the following command: <br>
>.alter table Car policy ingestionbatching '{"MaximumBatchingTimeSpan":"00:00:10"}' <br>
>.alter table User policy ingestionbatching '{"MaximumBatchingTimeSpan":"00:00:10"}'
{: .prompt-info }

