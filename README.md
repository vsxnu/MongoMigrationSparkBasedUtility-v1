# MongoDB to Azure Cosmos DB for MongoDB Migration - Spark Utility User Guide


## Introduction

This guide provides a detailed walkthrough of the data migration process from MongoDB (native) to Azure Cosmos DB for MongoDB (RU/vCore) using Azure Databricks and Spark.


**Important Note**: 
The Azure Cosmos DB Migration for MongoDB extension within Azure Data Studio plays a crucial role in evaluating MongoDB workloads for migration to Azure Cosmos DB for MongoDB. This extension empowers you to perform a comprehensive assessment of your MongoDB workload, assisting in the identification of essential steps for a seamless transition to Azure Cosmos DB. By utilizing this extension, you can do a holistic evaluation of your MongoDB workload, gaining actionable insights that contribute to a successful migration onto the Azure Cosmos DB platform. It is strongly recommended to assess your MongoDB workload before initiating the migration process. Refer to [Azure Cosmos DB Migration for MongoDB extension - Azure Data Studio | Microsoft Learn](https://learn.microsoft.com/en-us/azure-data-studio/extensions/database-migration-for-mongo-extension?view=sql-server-ver16) for steps to assess your MongoDB workload.


Migrations can be done in 2 ways:

- **Offline Migration**: A snapshot based bulk copy from source to target. New data added/updated/deleted on the source after the snapshot will not be copied to the target. The application downtime required will depend on the time taken for the bulk copy activity to complete.

- **Online Migration**: Apart from the bulk data copy activity done in the offline migration, a change stream monitors all additions/updates/deletes and stores them into an intermediate data store. After the bulk data copy is completed the data in the intermediate is copied to the target to make sure all updates done during the migration process are also copied to the target. The application downtime required will be minimal.

This tool supports the following sources:
- Native MongoDB (Online + Offline)
- MongoDB Atlas (Online + Offline)
- AWS DocumentDB (Online + Offline)
- Azure Cosmos DB MongoDB RU (only Offline)
- Azure Cosmos DB MongoDB vCore (only Offline)

The supported targets are:
- Azure Cosmos DB MongoDB RU
- Azure Cosmos DB MongoDB vCore


The migration process involves the following phases:

![02_01.MIG_Journey.png](/Image/02_01.MIG_Journey.png)

- **Pre-migration**: This step involves discovery and assessing readiness of your resources to for migration. Assessment involves finding out whether you're using the features and syntax that are supported. It also includes making sure you're adhering to the limits and quotas. The aim of this stage is to create a list of incompatibilities and warnings, if any. Refer to [Azure Cosmos DB Migration for MongoDB extension - Azure Data Studio | Microsoft Learn](https://learn.microsoft.com/en-us/azure-data-studio/extensions/database-migration-for-mongo-extension?view=sql-server-ver16) for steps to assess your MongoDB workload.

- **Migration**: This step is where you configure the migration environment, Spark Migration Utility and start executing the migration.
- **Monitor Progress**: Migration can be long running process and can take several hours to complete, in this this phase you monitor the various metrics and makes sure the migration is running as per expectation.  
- **Cut Over**: This phase is only applicable to online migrations and is used to cut over the incoming traffic from the current source to the new target.
- **Post Migration**: This step involves post migration steps, like optimizing the index policies, configuring your Azure Cosmos DB Mongo account for global distribution, etc. 


## Spark Migration Utility Overview

![02_02.Overview.png](/Image/02_02.Overview.png)

The Spark Migration Utility is running as a job in Databricks. The state of each migration is persisted in a Azure Cosmos DB NOSQL account which must be provisioned upfront. You need to create Azure Cosmos DB NoSQL Account and Databases. Collections will be provisioned based on JSON configuration. This Azure Cosmos DB NoSQL account contains collections which persists migration status, any errors as well as any documents which could not be migrated. 

> **IMPORTANT:**
> It’s crucial to delete the intermediate and status NoSQL Account and Databases promptly after migration. Failing to do so will result in higher costs.

## Prerequisites

Note: The Spark Migration Utility only works with Azure Cosmos DB RU based or vCore MongoDB as a target.

### Connectivity

The Databricks cluster must be able to connect to the source MongoDB, the Azure Cosmos DB NoSQL account  and the Azure Cosmos DB Mongo account.

Recommended is to deploy Databricks in a custom VNET which is either peered through VPN or ExpressRoute with your MongoDB instance. In cases where the source is not within a VPN environment, to ensure optimal data transfer and access speed, consider locating your Databricks cluster within the same region as the source data.

When dealing with a Virtual Private Network (VPN) environment, consider the following options:

- **Place Azure Databricks Cluster within the Same VPN (Recommended)**:
  For optimal performance and security, it is advisable to deploy your Azure Databricks (ADB) cluster within the same VPN as the source data. This approach ensures direct and secure communication between the cluster and the data source.  
- **VPN Peering with Source VPN**:
An alternative is to position your Databricks cluster within a VPN and then establish a VPN peering connection with the source VPN. This arrangement eliminates the need for IP address whitelisting and guarantees secure communication.  
- **Utilize NAT Gateway**:
While not recommended due to increased latency, you can place your Databricks cluster within a VPN and utilize a Network Address Translation (NAT) Gateway. This gateway ensures that all requests from cluster nodes carry defined IP addresses, which can then be whitelisted for access.

### Connection Strings  

The Spark Migration Utility uses connection strings to connect the following resources.

- Source MongoDB Cluster/endpoint
- Target Azure Cosmos DB MongoDB account
- Azure Cosmos DB NoSQL account

Verify the connection strings by using appropriate utilities to test connections upfront.

### Migration configuration

The Spark Migration Utility uses a JSON based configuration file as input. A detailed explanation of each option and sample JSON configuration files can be found [here](/Samples/VCORE_Sample_migrate_Collections_Offline.json) </br>

## Tutorial: Configuring Spark Migration Utility

### Cosmos NoSQL Intermediate Configuration

1. Log in to the [Azure Portal](portal.azure.com)
2. Click "Create a resource."
3. Type "cosmosdb" in the search box, locate "Azure Cosmos DB" and click "Create."
   
    ![02_03.cosmos_nosql_01](/Image/02_03.cosmos_nosql_01.png)

5. Choose NoSQL
   
    ![02_03.cosmos_nosql_02](/Image/02_03.cosmos_nosql_02.png)

7. Select correct Subscription, Resource Group, Account Name and Location. You can leave default the rest.
8. Choose Connectivity Method for your network environment on Networking Tab.
9. Review + Create 
*(You can refer to the tutorial: [Quickstart: Create an Azure Cosmos DB account, database, container, and items from the Azure portal](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/quickstart-portal#create-account))* 
10. Once created, click "Go to resource."
11. Navigate to the “Settings” section in the Left Nav, then click on “Keys” to copy the URI and Primary Key for future use.


### JSON Configuration

The Spark Migration Utility uses a JSON configuration file as input. 


Refer to  the [config file details](/configuration%20file%20details) for more details.

We offer a variety of sample configuration files that serve as boilerplate templates in the **Samples** folder. For instance, you can find a sample JSON file to migrate selected collections for vCore offline migration [here](/Samples/VCORE_Sample_migrate_Collections_Offline.json). Feel free to explore and adapt these templates to your needs!

Use the appropriate connection string in the URL for both the source server and destination server.

Use the URI and Primary Key that you copied from your Cosmos DB NoSQL account earlier as the url and key for both the intermediate server and the status server.

**Note:**
- When the source is Azure Cosmos DB MongoDB RU add "mongoImplementationType": "MONGODB_API_FOR_COSMOSDB" below 'indexCreation'.
- When the source is AWS DocumentDB add "mongoImplementationType": "MONGODB_API_FOR_DOCUMENTDB" below 'indexCreation'.


### Databricks Configuration

#### Create Azure Databricks cluster

1. Log in to the [Azure Portal](portal.azure.com)
2. Click "Create a resource."
3. Type "databricks" in the search box, locate "Azure Databricks" and click "Create."

    ![02_04.databricks_01.png](/Image/02_04.databricks_01.png)

5. Choose your subscription and resource group. If the resource group doesn't exist, create one for this job.
6. Provide a name for the Databricks workspace.
7. Opt for the same region as your MongoDB database (if hosted on Azure) or your Azure Cosmos DB for MongoDB account.

    ![02_04.databricks_02.png](/Image/02_04.databricks_02.png)

9. (Optional) If you desire a specific VNet for the job Click on "Networking." Select "Yes" for "Deploy Azure Databricks workspace in your own Virtual Network (VNet)."
    - Choose the VNet's name.
    - Specify a name for the public subnet.
    - Determine an available IP address range from your VNet's address space.
    - Assign an IP address range to both the public and private subnets. For example, if your VNet's address space is 10.7.0.0/16, you could use 10.7.101.0/24 for the public subnet and 10.7.100.0/24 for the private subnet.
      
      ![02_04.databricks_03.png](/Image/02_04.databricks_03.png)

10. Click "Review + create."
11. Click "Create."
12. Once created, click "Go to resource."

    For detail information, please refer to the [Databricks Network Overview](https://learn.microsoft.com/en-us/azure/databricks/security/network/) and [Private Link Overview](https://learn.microsoft.com/en-us/azure/databricks/administration-guide/cloud-configurations/azure/private-link)
    


#### Configure Azure Databricks for migration

1. Open the Azure Databricks resource created in the above step.
2. Click "Launch Workspace."

    ![02_05.config_01.png](/Image/02_05.config_01.png)

4. Click "Workflows." From the left blade
5. Click "Create Job."

    ![02_05.config_02.png](/Image/02_05.config_02.png)

7. In the dialog, complete the following:

- Assign a name to the task.
- Select "JAR" for Type.
- For the main class: ```com.microsoft.azure.cosmosdb.migration.MongoToCosmosMigration```

    ![02_05.config_03.png](/Image/02_05.config_03.png)


6. Upload the utility jar from the [Git repository](https://github.com/AzureCosmosDB/MongoMigrationSparkBasedUtility/releases) (latest release version)

    ![02_05.config_04.png](/Image/02_05.config_04.png)

8. Add the following dependent jars using Maven (Same or above than below versions)

    - org.mongodb.spark:mongo-spark-connector_2.12:10.1.1
    - com.azure.cosmos.spark:azure-cosmos-spark_3-3_2-12:4.17.2
  
    ![02_05.config_05.png](/Image/02_05.config_05.png) 

8. You should see a total of 3 JAR files after upload
   
    ![02_05.config_06.png](/Image/02_05.config_06.png)

10. Create the Job by clicking "Create."
11. Provide a name and save the job.
12. After the job is created, click on the job name in the Workflows screen.
13. Click on "Configure" in the "Compute" section on the right-hand side pane.
    
    ![02_05.config_07.png](/Image/02_05.config_07.png)

15. Select the Worker Type between Standard_DS3_v2 or Standard_DS5_v2
    
    ![02_05.config_08.png](/Image/02_05.config_08.png)

17. Put the cluster name, choose Multi Node or Single Node, Workers amount and so on.
18. In Advanced options, add the following Spark configuration options(edit the values with <..> before pasting)  and then click "Confirm / Create".

    ***DS3_v2 4 vcore cluster***
    
    ```
    spark.driver.extraJavaOptions -XX:+UseG1GC -XX:G1HeapRegionSize=16M -XX:+PrintFlagsFinal -XX:+PrintReferenceGC -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -verbose:gc -XX:+PrintGCDetails -XX:+PrintAdaptiveSizePolicy -XX:AdaptiveSizePolicyOutputInterval=1 -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:MaxDirectMemorySize=4096M -XX:-ResizePLAB -XX:+UseCompressedOops -XX:InitiatingHeapOccupancyPercent=45 -XX:+UnlockExperimentalVMOptions -XX:G1MixedGCLiveThresholdPercent=85 -XX:ParallelGCThreads=16 -XX:ConcGCThreads=4
    spark.databricks.delta.preview.enabled true
    spark.executor.cores 4
    spark.executor.memory 4g
    spark.executor.memoryOverheadFactor .1
    spark.memory.offHeap.size 2g
    spark.memory.offHeap.enabled true
    spark.executor.instances 4
    spark.task.maxFailures 1
    spark.driver.memory 6g
    spark.executor.extraJavaOptions, -XX:-UseParallelGC -XX:+UseG1GC -XX:+PrintFlagsFinal -XX:+PrintReferenceGC -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintAdaptiveSizePolicy -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=25 -XX:MaxGCPauseMillis=500 -XX:+ExplicitGCInvokesConcurrent
    spark.driver.memoryOverheadFactor .1
    spark.databricks.libraries.enableMavenResolution false
    spark.driver.cores 4
    ```
        
    ***DS5_v2 16 vcore cluster***
    
    ```
    spark.driver.extraJavaOptions -XX:+UseG1GC -XX:G1HeapRegionSize=16M -XX:+PrintFlagsFinal -XX:+PrintReferenceGC -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -verbose:gc -XX:+PrintGCDetails -XX:+PrintAdaptiveSizePolicy -XX:AdaptiveSizePolicyOutputInterval=1 -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:MaxDirectMemorySize=4096M -XX:-ResizePLAB -XX:+UseCompressedOops -XX:InitiatingHeapOccupancyPercent=45 -XX:+UnlockExperimentalVMOptions -XX:G1MixedGCLiveThresholdPercent=85 -XX:ParallelGCThreads=16 -XX:ConcGCThreads=4
    spark.databricks.delta.preview.enabled true
    spark.executor.cores 16
    spark.executor.memory 16g
    spark.executor.memoryOverheadFactor .1
    spark.memory.offHeap.size 16g
    spark.memory.offHeap.enabled true
    spark.executor.instances 120
    spark.task.maxFailures 1
    spark.driver.memory 48g
    spark.executor.extraJavaOptions, -XX:-UseParallelGC -XX:+UseG1GC -XX:+PrintFlagsFinal -XX:+PrintReferenceGC -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintAdaptiveSizePolicy -XX:+UnlockDiagnosticVMOptions -XX:+G1SummarizeConcMark -XX:InitiatingHeapOccupancyPercent=25 -XX:MaxGCPauseMillis=500 -XX:+ExplicitGCInvokesConcurrent
    spark.driver.memoryOverheadFactor .1
    spark.databricks.libraries.enableMavenResolution false
    spark.driver.cores 16
    ```
    
    ![02_05.config_09.png](/Image/02_05.config_09.png)


01. Go to Catalog and Click Add
   
     ![02_06.JSON_01.png](/Image/02_06.JSON_01.png)

3. Select DBFS
   
     ![02_06.JSON_02.png](/Image/02_06.JSON_02.png)
   
5. Drag & Drop your JSON configuration file for the migration.
   
     ![02_06.JSON_03.png](/Image/02_06.JSON_03.png)
   
   Copy the path of the uploaded file. Do not select any button(s).
   
7. Go back to your JOB and select Tasks menu.
   
     ![02_06.JSON_04.png](/Image/02_06.JSON_04.png)
   
9. Put your uploaded JSON file path like below in Parameters section.
   ```["-Dmigrator.parametersJsonFile=<Your File Path and File Name>"]```
   
     ![02_06.JSON_05.png](/Image/02_06.JSON_05.png) 

### Spark Scheduler

If you are planning to migrate multiple container of varying sizes, it is better to set the scheduler to fair. To enable that you need to do the following. Create a file with the name [**fairscheduler.xml**](/Samples/fairscheduler.xml) with following content in it.

```
<?xml version="1.0"?>
<allocations>
  <pool name="fair">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
```

Once you have this file ready, you need to upload it to databricks data section similar to the way you uploaded the json file. Also you need to go to the cluster configuration and add following properties to it. 
```spark.scheduler.allocation.file <Your File Path and File Name>```

![02_10.Spark_Scheduler.png](/Image/02_10.Spark_Scheduler.png) 

## Run Migration Job

Click "Run Now".

![02_07.RUN.png](/Image/02_07.RUN.png) 

## Complete the migration (Monitoring)

As the task begins its execution, navigate to the Spark (UI) within Azure Databricks. The first step of the execution is partitioning the source collection documents in Spark. This allows Spark to migration documents more efficiently. Be aware that it might take several minutes or longer before the partitioning is finished, and the migration starts. When you click on the running Job and select Spark UI observe that the task triggers one or more bulk copy jobs.

Once the migration is completed your Job status will show Succeeded. You can also validate this by examining the status in the Azure Cosmos DB NoSQL account: 

1. Navigate to the Azure Cosmos DB NoSQL account
2. Click on Data Explorer and select the migration database you have specified in the JSON configuration file
3. Click on migration status container and click on new SQL query
4. Run: ```SELECT * FROM c where c.database = 'yourdbname'```
5. In the JSON retrieved you can examine the status.

    ``` JSON
    {
        "id": "b1c4318b-7310-...",         --> Identifier
        "migrationType": "BULK_COPY",			 --> Type Bulk Copy or Stream Changes
        "parentId": null,
        "sourceName": "sourceServer",			 --> Source Name as provided
        "database": "db63828765864653",		 --> Database being migrated
        "collection": "bigCollection",		 --> Collection  being migrated
        "sessionId": "c4ea826f-5ba9-4dbe-801e-f4fe4fe8fbfb",
        "timestamp": "Tue Dec 26 12:09:20 UTC 2023",
        "streamingToIntermediateCheckpoint": "/tmp/streaming/intermediate/17...",
        "streamingToDestinationCheckpoint": "/tmp/streaming/destination/17035...",
        "sourceDocumentCount": 50,				--> documents to be migrated
        "destinationDocumentCountAfterBulkCopy": -1,	--> documents migrated in bulk copy
        "destinationDocumentCountAtEnd": -1,	-->	documents  migrated after resume finish
        "finalDocumentCount": 0,								
        "status": "RUNNING",							--> Current Status
        "_rid": "prEEAOjMMToCAAAAAAAAAA==",
        "_self": "dbs/prEEAA==/colls/prEEAOjMMTo=/docs/prEEAOjM....",
        "_etag": "\"1900f671-0000....",
        "_attachments": "attachments/",
        "_ts": 1703592560
    }
    
    ```

## Cleanup Migration Resources

Once the migration job is complete, intermediate and status databases are not required. It is prudent to delete those in order to save costs. To automatically delete all the intermediate and status databases, change the main class to ```com.microsoft.azure.cosmosdb.migration.CleanupMigrationResources``` and run the Job. Use the same JSON that was used for original migration. This job will delete everything in status and intermediate databases.

> **IMPORTANT:**
> After completing the migration, it’s crucial to delete the status Azure Cosmos DB NoSQL account promptly. Failing to do so will result in higher costs.



## Online Migration


Online Migration follows the same steps as offline migration. The only difference is that the Spark Migration Utility will keep running after the bulk copy job is finished. It will stream data changes from the source MongoDB cluster to the intermediate Azure Cosmos DB NoSQL account and from there to the destination Azure Cosmos DB Mongo account. 

> **WARNING:**
> Failing to delete the intermediate Azure Cosmos DB NoSQL account promptly after migration will result in higher costs.

- This Spark Migration Utility uses [Change Stream](https://www.mongodb.com/docs/manual/changeStreams/) of MongoDB. **Change Stream need to be available** on Source Database. **(Change Stream is available on MongoDB 3.6 or later.)** 

- Online Migration needs to use **Multi Node** instead of Single Node. 

    ![02_11.Online_Migration_03.png](/Image/02_11.Online_Migration_03.png) 

    In the JSON configuration file two properties must be changed to true:
    
    ```
    "startStreamToIntermediate": true
    "startStreamToDestination": true
    ```

    ![02_11.Online_Migration.png](/Image/02_11.Online_Migration.png) 

 During online migration, for each collection in the source, the tool will create a corresponding collection in the Intermediate Database. These intermediate collections will always adopt the name of the destination collection. Therefore, it’s important not to attempt migrating two collections with the same name within a single JSON file.


[**Sample JSON File**](/Samples/VCORE_Sample_migrate_Collections_Online.json)

Once these properties are changed upload the JSON configuration file to Databricks and start the job. 

The first stage of the job execution is exactly the same as with offline migration. A bulk copy job is started. Once the bulk copy job is finished, when opening Spark UI you will see two new jobs running for each of the collections you are migrating:

- Source-to-Intermediate-<collection name>
- Intermediate-to-destination-<collection name>

The source-to-intermediate job acquires a change stream cursor from the source MongoDB cluster and streams data changes from the oplog to the intermediate Azure Cosmos DB NoSQL account. These changes are stored in the database and container which are specified in the Task block, intermediate section of the JSON configuration file.

The intermediate-to-destination job will read from this Azure Cosmos DB NoSQL container and replicate documents to the destination Azure Cosmos DB Mongo account.

Note: Please validate upfront that the oplog size of the source MongoDB cluster is sufficient.

When using Online Migration your Databricks job(s) will keep running and streaming data changes until the jobs are stopped as part of the cut over process.

Remember to [cleanup up the migration resources](#cleanup-migration-resources) promptly to avoid high costs.


### Monitoring Online migrations 

When you have started an online migration the migration tool will first do a bulk copy and then run separate streaming jobs, from source to intermediate and from intermediate to destination. To understand whether bulk copy has completed please navigate to your running Job in Databricks and open the Spark UI. If you don’t see a BULK COPY job instance running than this phase is completed.

You can then validate the document count between source and target. Use a mongo shell to connect to your Cosmos DB account and run:

For a single collection:

```
    use database;     
    db.getCollection(“name”).stats().count 
```
 
For all collections:

```
    db.getCollectionNames().sort().forEach(function(collName)     
    {     
     var count = db.getCollection(collName).stats().count;     
     	 print(collName + ": " + count)     
    } 

```

You should see the count on target increasing, also the gap between source and target should be reducing. If you see a discrepancy please navigate to the [Troubleshooting](#troubleshooting) section.

Also if your Azure Cosmos DB for MongoDB vCore account has diagnostic logging enabled you can also validate the incoming requests:
 
```
    VCoreMongoRequests 
    | where DatabaseName == name' 
    | summarize count() by OperationName 
```


Wait for the source and target counts to align. Once they are close, initiate the cut-over process by stopping new updates to the source server. Allow some time for the target to catch up with the source documents. When the counts match, it indicates that it’s time to terminate the job and redirect traffic to the source.

## Post-migration optimization

After you migrate the data stored in MongoDB database to Azure Cosmos DB for MongoDB, you can connect to Azure Cosmos DB and manage the data. You can also perform other post-migration optimization steps such as optimizing the indexing policy, update the default consistency level, or configure global distribution for your Azure Cosmos DB account.


## Recommendations

### Environment

- There should be minimal network latency overhead between source MongoDB cluster, Databricks cluster, Azure Cosmos DB NoSQL and Azure Cosmos DB Mongo RU/vCore. Make sure Databricks cluster and Azure Cosmos DB are always in the same Azure region.
- Use a dynamic Databricks cluster for your migration. This simplifies migration job creating since you can clone your migration jobs. Cloning your migration jobs in Databricks makes it easier to manage any JAR file dependencies. Only use a dedicated cluster in case a dynamic cluster is the performance bottleneck for your migration.
- When preparing for migration it is highly recommended to connect to the secondary read replica of the source MongoDB cluster if available. You can achieve this by appending ```/?readPreference=secondary"``` to the connection string.
- Make sure the [oplog retention size](https://www.mongodb.com/docs/manual/core/replica-set-oplog/) of the source Mongo is able to sore operations for at least 4-5 days. With a too small oplog retention size and a high amount of write operations online migration might fail or might be unable to read all documents from the change stream in time.
- In case the source has a high amount of write operations per minute you might need to scale the RU of the Azure Cosmos DB NoSQL account. You can validate that by [monitoring with Insights](https://learn.microsoft.com/en-us/azure/cosmos-db/use-metrics). If you see the normalized RU maxing out you should scale the RU higher.
- In case Point in Time Restore is enabled for Azure Cosmos DB MongoDB RU and you want to migrate unique indexes from the source, make sure you [pre create the collection and the unique indexes](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/custom-commands#create-collection) before you start migration. The Spark based Migration Utility will first create a collection and then any unique indexes. This will not work when Point In Time Restore is enabled.
- When migrating to Azure Cosmos DB vCore make sure you have added ``` "mongType": "VCORE" ``` to the destination section of the tasks block in the [JSON configuration file](/Samples\Sample_migrate_VCORE_Offline.json). This will tell the Spark based Migration Utility to use vCore optimized settings.

## Troubleshooting

### Test Connecivity from Databricks Cluster

To test connectivity of Source & target from Databricks. 

1. Click on your Cluster Name from Compute blade of Databricks workspace.
2. Click Apps
3. Click Web Terminal
   
   ![02_12.Connectivity_01.png](/Image/2_12.Connectivity_01.png)

- If the "Web Terminal" is disabled, please follow below steps to enable it.
  - Go to the **admin settings** page.
  - Click the **Workspace settings** tab.
  - In the **Advanced** section, click the **Web Terminal** toggle.
    
     ![02_12.Connectivity_02.png](/Image/2_12.Connectivity_02.png)

4. Use the terminal to Checking Connectivity with Telnet

- ```telnet <IP> <Port>```

5. Run Mongosh to check connectivity on databricks cluster (Run the following commands to install Mongosh)

```
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt-get update

sudo apt-get install -y mongodb-mongosh mongodb-org-tools
```

### Scenario: Migration is running slowly

When a migration is running slowly you need to investigate if there is bottleneck which prevents your migration from completing quicker.  

1. Examine metrics at the source MongoDB cluster. The Spark Migration Utility might put a lot of pressure on the source MongoDB cluster. Important metrics to examine are:
    - Disk Read IO. If you see the read IO spiking the disk can’t keep up.  
    - CPU percentage. If you see the CPU percentage spiking the CPU is unable to keep up.
    - MongoDB connection. You can examine the serverStatus to see number of incoming connections.
      - Disk Read IO. If you see the read IO spiking the disk can’t keep up.  
      - CPU percentage. If you see the CPU percentage spiking the CPU is unable to keep up. 
      - MongoDB connection. You can examine the serverStatus to see number of incoming connections.
      - serverStatus — MongoDB Manual and whether the max amount of connections has been reached.

    To mitigate source MongoDB cluster issues, please validate with your resource provider.

2. Next step is examining the target Azure Cosmos DB MongoDB account.

    For RU:
    1. In the Portal navigate to Azure Cosmos DB and open the account.
    2. Go to Insights.
    3. Select the database and collection you are migrating to.
    4. Click on Throughput and examine Highest RU Consuming Shard (%) By ShardKeyRangeId graph.
    5. When you see one or more shards spiking at 100% for a longer period of time you have either not enough RU provisioned or the Spark Migration Utility job in Databricks should use a less powerful configuration (less cores and less workers).
    6. You can validate this by clicking on Requests and examine the Failed Client Requests by Operation Type graph. If you see failed errors here this means Azure Cosmos DB Mongo API is retyring this operations server side.
    
    For vCore:
    1. In the Portal navigate to Azure Cosmos DB and open the account.
    2. Navigate to Monitoring and then Metrics.
    3. Select metric CPU Percent and select Max aggregation.
    4. Add new metric and select Memory Percent and select Max aggregation.
    5. Add new metric and select IOPS and select Max aggregation.
    6. In the top right select the time period of the migration.
    7. Examine the graph.

  If you see any spikes near or at 100% the vCore account is facing resource constraints. To overcome the resource constraints [scale up your vCore cluster](https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/vcore/how-to-scale-cluster).

3. Next step is examining the Migration JSON file. When migrating large source collections we recommend to only configure [one collection in the JSON file](/Samples\VCORE_Sample_migrate_Collections_Offline.json) and create a Databricks Job per collection to migrate. Access the Databricks UI to gain insights into the job's tasks and their progress:

    ![02_09.Spark_UI_01.png](/Image/02_09.Spark_UI_01.png) 
    
    - Spark UI – Jobs: Monitor different job states, including:  
      - Active Jobs (jobs which are currently being executed)
      - Completed Jobs (jobs which are completed)
      - Failed Jobs (jobs which failed due to an error). 
    
          ![02_09.Spark_UI_02.png](/Image/02_09.Spark_UI_02.png) 
    
    - Spark UI – Executors: For each executor (node), review the following details:  
      - Memory Usage  
      - Active, Failed, and Completed Tasks  
      - Task Time (GC Time) - A high GC time with a red executor indicates potential issues. 
    
          ![02_09.Spark_UI_03.png](/Image/02_09.Spark_UI_03.png) 
    
    - Spark UI – Logs: For failed tasks, inspect error logs to identify any problems.
      - Executor State and Loss Reason: Examine executor state, loss reason, and code to understand failures.
      - Source Database Throttling: Look for exceptions in the logs that might indicate source database throttling requests. 



### Scenario: Migration completed but not all documents are copied

To validate if any errors occurred due to e.g. duplicate documents:

1. Navigate to the Cosmos NoSQL account  
2. Click on Data Explorer and select the migration database you have specified in the JSON configuration file
3. Click on migration_status_error container and click on new SQL query
4. Run: ```SELECT * FROM c where c.database = 'yourdbname'```  

If there are any documents returned, examine the error property which contains a detailed error description. 

In case of Online data migration make sure you have set the property IdUnique to true in the JSON migration configuration file. This ensures upserts are always copied, even if a property in the unique index has changed. 

### Scenario: Online Migration job failed

In case of Online Migration a job might fail due to for example a connection timeout or reset due to a maintenance event. In this case you can resume the online migration using the following steps:

1. Navigate to the Azure Cosmos DB NoSQL account
2. Click on Data Explorer and select the migration database you have specified in the JSON configuration file
3. Click on migration_status container and click on new SQL query
4. Run: ```SELECT c.sessionId FROM c where c.collection = 'yourcollectionname'```
5. Copy the sessionId value
6. Go to Databricks Studio and click on the failed job
7. Click on Tasks
8. Change the main class to com.microsoft.azure.cosmosdb.migration.ResumeStreams
9. Add an additional parameter ```"-Dmigrator.resumeSessionId=<yoursessionid>"```
10. Your parameters should look like:
    ```["-Dmigrator.parametersJsonFile=/FileStore/tables/config.json", "-Dmigrator.resumeSessionId=yoursessionId"]```
11. Click save and click Run to execute the job. 

## Known Issues

### GC in the cluster: The cluster has run out of memory

- Cause of the Issue
  - The cluster is small.
  - Write Throughput is low.
  - Our tooling is not doing memory management optimally.
- Fix for the Issue
  - Try to get a cluster with same # cores but higher memory.
  - Increase the RU at the destination.
  - Report back GC Issues.

### Migration is successful but the count of documents in source is more than the count of documents at destination

- Cause of the Issue
  - Possibly there were errors while migrating some of the records
- Fix for the Issue
  - Goto status db and look for stats in _status and_error collection.If the errors were transient, then running the tool with Resume option will fix. Else, running the Job with the main class CompareAndUpdate will do a full DB scan and update the missing records. Since it is doing full DB scan it will take lot of time.

### Migration is successful but the count of documents in source is less than the count of documents at destination

- Cause of the Issue
  - Source DB is getting modified
- Fix for the Issue
  - Do online migration to detect if the source db is not getting modified. On mongo, count() operator given an estimate of number of documents, use .count_documents() to get correct number of documents.  

### Connection Issues with the mongo source: Timeouts, Socket Exception, Lock and Wait exception

- Cause of the Issue
  - VPN
  - Transient
  - Source is resource constraint.
- Fix for the Issue
  - VPN Peering.
  - Dedicated Source for Read.
  - Use a smaller cluster to read.

### Different issues at Cosmos

- Cause of the Issues 429, 413, 403
  - 429 means RU provisioned is lower than what is required by the current cluster size.
  - 413 means document size is too large
  - 403 means 20G limit of unsharded collection has been reached.
- Fix for the Issue
  - To remedy 429, either reduce the cluster size or increase RU provision.  
  - To remedy 413, increase the max allowed document size to 16 MB.  
  - To remedy 403, shard the collection.

### Shard Key incompatibility: Complex Type with > 8 fields.  

- Cause of the Issue
  - The shard key is not supported in Azure Cosmos DB.
- Fix for the Issue
  - Either create an unsharded collection if the size of the collection is < 20G or modify the shard key so that it is supported in Azure Cosmos DB.

### Unique Key Constraint Violation Exception  

- Cause of the Issue
  - Empty shard key value are allowed in mongo native but not in Azure Cosmos DB MongoDB RU.
  - There are two shard key values, one of type String, second of type ObjectId, but both have same value.  
- Fix for the Issue
  - Config has a parameter: uniqueConstraintViolationHandling. You can use this to handle records which violate this constraint. 



## How To

### Read status server

Assuming the entry in config looks like this:

```
"status": {
    "serverName": "statusServer",
    "database": "database_name",
    "collection": "migration_status",
    "throughput": 4000
},
```

a. Goto the portal UI of the Azure Cosmos DB NoSQL account for statusServer

b. In the data explorer find the database named database_status

c. Within the db database_status look for the collection migration_status

d. Now run the query: select * from c where c.collection = “target_collection_name”

e. The result will look like this:

```
{
    "id": "38804944-2c30-486c-85b6-76309e7df9d8",
    "migrationType": "BULK_COPY",
    "parentId": null,
    "sourceName": "sourceDBServer",
    "database": "sourceDBName",
    "collection": "sourceCollection",
    "sessionId": "e0e1cc6d-1c47-4705-be5a-533d89a2df6d",
    "timestamp": "Wed Aug 02 17:22:23 UTC 2023",
    "streamingToIntermediateCheckpoint": "/tmp/streaming/intermediate/1690996943511/ sourceDBName / sourceCollection /",
    "streamingToDestinationCheckpoint": "/tmp/streaming/destination/1690996943511/ destinationDBName / destinationCollection /",
    "sourceDocumentCount": 825583,
    "destinationDocumentCountAfterBulkCopy": 825583,
    "destinationDocumentCountAtEnd": 825583,
    "finalDocumentCount": 825583,
    "status": "SUCCESSFUL",
    "_rid": "cAEQAPZM2WUCAAAAAAAAAA==",
    "_self": "dbs/cAEQAA==/colls/cAEQAPZM2WU=/docs/cAEQAPZM2WUCAAAAAAAAAA==/",
    "_etag": "\"0200fdc0-0000-0700-0000-64caa2c40000\"",
    "_attachments": "attachments/",
    "_ts": 1691001540
}
```

The keys which are relevant are: collection, sourceDocumentCount, … finalDocumentCount, status.
If the status is successful and the finalDocumentCount matches sourceDocumentCount, then the job is successful.
If the result looks like this:

```
{
       "id": "af65102d-d073-44c4-b801-536aa6fdc2a0",
       "migrationType": "BULK_COPY",
       "parentId": null,
       "sourceName": "sourceDBServer",
       "database": "sourceDBName",
       "collection": "sourceCollection",
       "sessionId": "e0e1cc6d-1c47-4705-be5a-533d89a2df6d",
       "timestamp": "Wed Aug 02 17:22:48 UTC 2023",
       "streamingToIntermediateCheckpoint": "/tmp/streaming/intermediate/1690996943511/ sourceDBName / sourceCollection /",
       "streamingToDestinationCheckpoint": "/tmp/streaming/destination/1690996943511/ destinationDBName / destinationCollection /",
       "sourceDocumentCount": 874,
       "destinationDocumentCountAfterBulkCopy": 862,
       "destinationDocumentCountAtEnd": -1,
       "finalDocumentCount": 0,
       "status": "PARTIAL",
       "_rid": "cAEQAPZM2WUFAAAAAAAAAA==",
       "_self": "dbs/cAEQAA==/colls/cAEQAPZM2WU=/docs/cAEQAPZM2WUFAAAAAAAAAA==/",
       "_etag": "\"0200724d-0000-0700-0000-64ca91ea0000\"",
       "_attachments": "attachments/",
       "_ts": 1690997226
}
```

In this case the status is PARTIAL and the document count after bulk copy is less than that in the source, it means that the job was not successful. Run the below query on the collection to look in the collection: migration_status_error for the reasons behind these missing records :
```SELECT * FROM c where c.collection = ‘sourceCollection’```

In the resulting records, look for the key “error”, one such example is:

```
"error": "com.mongodb.MongoWriteException: Write operation error on server agys-stay-poc-eastus.mongo.cosmos.azure.com:10255. Write error: WriteError{code=16, message='Error=16, Details='Response status code does not indicate success: RequestEntityTooLarge (413); Substatus: 0; ActivityId: 94982c55-81c8-4067-a459-3736049a3280; Reason: (Message: {\"Errors\":[\"Request size is too large\"]}\r\nActivityId: 94982c55-81c8-4067-a459-3736049a3280, Request URI: /apps/248685e1-d438-403e-a3dc-6993b5ab9c6b/services/e06197e9-51eb-4cc0-a20e-fde549009559/partitions/3d408a23-3090-4d19-ba01-f50bf758c738/replicas/133274793010762545p/, RequestStats: Microsoft.Azure.Cosmos.Tracing.TraceData.ClientSideRequestStatisticsTraceDatum, SDK: Windows/10.0.17763 cosmos-netstandard-sdk/3.18.0);', details={}}."
```

In this case the request size is too large so it looks like the size of the document is larger than the max limit at Azure Cosmos DB RU which is 2 MB (extendable to 16 MB).
