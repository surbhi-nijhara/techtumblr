
## Overview
The problem statement is to compare MongoAtlas and Azure Cosmos DB. 
 
# Technology Requirements
 
The important technology requirements are :
* Database as a service
* High Availability and Scalability
* Continuous Backup management
* Optimized Writing to Disk strategy
* Granular encryption and access control
 
# Business Requirements 
Performance is the main business objective of the solution that technology is required to solve.
 
# Recommendation: 
 
**MongoAtlas** which is a managed service of MongoDB and can be hosted in Azure is recommended.
 
The top reasons are supportability, 4.2 in-memory caching and vetted & burned in up- to-date versioning of Mongo database.
 
Microsoft Cosmos lags behind in the data space with capabilities and it may be 18-24 months before it catches up with Mongo 4.2.
 
_Note: At the time of writing this document, Current stable version is 4.2_
 
# Key Drivers
 
Following are the important considerations that we weighed for MongoAtlas and Cosmos:

## Pricing
MongoAtlas Billing Model depends on:

	* Mainly server size, 
	* Data transfer size and 
	* Number of servers.
	
With [Calculator](https://www.mongodb.com/cloud/atlas/compare),
Possible Price: 28GB RAM, 256GB Storage, 50 GB data size = $1598 /m
   (Includes cluster of 3)

Cosmos (with SQL API) Billing Model depends on: 

	* Mainly RU (r/s, w/s) and 
	* Item size based
                
 Possible Price:  400 r/s, 800 w/s, 256 GB storage, Item size 50 KB = $1234 /m

_Disclaimer: Cosmos Calculator gives pricing for SQL API (Not Mongo API).Cosmos with SQL API pricing estimate is provided to give a high level idea. Cosmos with Mongo API price may differ as per Azure support._

**Notes**: 
* The values for calculation are assumptions.
* Single region is considered. (Mongo Calculator is not supporting cross-region). Reference: Here
* CloudNative Backup for MongoAtlas is considered.
* MongoAtlas costs does not include support costs.
* Azure does not provide any SLAs for Cosmos with Mongo API.

**_Choice: MongoAtlas_**
                                             
## Version
**MongoAtlas** : Current stable version is 4.2

**Cosmos with Mongo API** : Currently supported is compatible with v3.6 and still 
has a bunch of not supported features. 
Key ones which are not supported are called out here as below.

	Sparse and Partial Indexing,
	Query ad map operators for aggregation expressions and 
	schema validation,
	Text search indexes, 
	Cursor Methods, 
	Bitwise Operators, 
	Administrative and diagnostic commands 
	
Full list of not-supported operations is at the end of this section in Cosmos (Mongo API with 3.6).

**_Choice: MongoDB_**

## Multi-doc ACID
MongoAtlas : Yes

Cosmos with Mongo API: No. If required, it will be needed to do it at app-tier. 
Support is provided by SQL API.

**_Choice: MongoDB_**

## Write to Disk strategy
Mongo Atlas: Takes in-memory snapshots and writes asynchronously to disk in 60 secs frequency.
Cosmos: Not supported.

**_Choice: MongoAtlas_**

#Sharding Mechanism
Mongo Atlas: Writes horizontally and A shard can store data in TBs
Cosmos: One partition has Upper limit of 20G. So if my collection is > 20G, then choose a partition key, else Cosmos will choose on its own.

Due to partition storage size, the level(high/low) of cardinality of the key may differ while choosing the key.

If I have to choose as best practice, then I should consider the key as comparatively a low cardinality , so that the partitioned subset resides in that partition

**_Choice: MongoAtlas_**

## Cardinality
Mongo and Cosmos sharding mechanisms are built differently, therefore the sharding key should be different between systems if you want to fully utilize the platforms.

Always consider the number of values your shard key can express. A sharding key that has only 50 possible values, is considered as a low cardinality, while one that might be able to express several million values might be considered a high cardinality key. 
High cardinality keys are preferable to low cardinality keys to avoid un-splittable chunks.
So in MongoDB you will want to have high cardinality partition keys to target chunks (logical partitions) of about 64MB,
where as in Cosmos DB you will target low cardinality partitions keys because the logical partitions are up to 10G

## Automated Backup Management
MongoAtlas: Only Cloud Provider Backs are supported, which will need to be defined.
1. Define Backup Policy
2. Enable PIT restores whichs replay the oplog to restore a cluster from a particular point in time within a window specified in the Backup Policy.
3. Scheduled Snapshots:Snapshot Scheduling and Retention policy has to be defined
4. On-Demand Snapshots are also possible,
5. Encryption: Atlas automatically encrypts Cloud Provider Snapshots for clusters using Encryption at Rest using Customer Key Management
 
Cosmos: Microsoft automatically backups all your CosmosDB databases every 4 hours. Only the two latest backups are stored. For restore, a support query has to be created.

**_Choice: MongoAtlas (if continuous snapshots and restore are required)_**

## Encryption
* Both DBs support Encryption at rest and inflight.
* Both provide Client side Encryption libraries.
* Both support own (customer-managed keys
* Key vaults are supported by both.
* MongoDB: Supports client field level encryption, while Cosmos does not.
* MongoDb also supports access control at field level, while Cosmos does not.
 
**_Choice: MongoAtlas_**

## Community support
Both CosmosDB and MongoDB Atlas have great developer community support. CosmosDB is well supported by the Azure community while MongoDB is robustly supported by the open source community.

**_Choice: MongoAtlas_**

## Validation for data Governance
MongoAtlas: yes
Cosmos : No. Needs to be implemented in app tier

**_Choice: MongoAtlas_**

## Data Type
MongoAtlas: Bson

Cosmos: Bson support with Mongo API 
**_Choice: Neutral_**

## Max Doc Size
Both support 16 GB

**_Choice: Neutral_**

## Cross-platform support
MongoDB also has Java drivers as well as driver for C#/.Net along with drivers for a wide variety of languages and frameworks.
Similarly, CosmosDB has great SDK available for Java as well as .Net Core along with Node, Python etc.

**_Choice: Neutral_**
