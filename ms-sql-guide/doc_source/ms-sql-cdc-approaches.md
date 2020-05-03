# Primer on CDC Strategy in MS SQL Guide

## Purpose
The purpose of this guide is to approach a common business problem of tracking changes to the tables in SQL server. There could be use cases where an impact in one system needs to be propagated to another system in the business ecosystem. 
For example, new users on-boarding into a service provider’s server requires a confirmation via emails or sms. Such notification handlers may already be part of another system and there arises a need for the communication between the core system and notification system which we usually achieve by an event driven mechanism. 
Similarly another use case could be where we need to synchronize two system servers for a certain business need. 
We have more than one way to approach this and this is a guide which will help get started.


## Approach 1: Application sends changed information to an event b	us.


The following diagram shows the key components of the configuration for this scenario\.

![\[Diagram for approach 1: CDC with App\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-app.jpg?raw=true)

1. In this approach, as soon as an application service writes((insert, update, delete) into tables to be captured, the same application also publishes an event  into an event bus. The event may just contain id of the template, the operation(op) and timestamp. Alternatively, one may approach publishing the entire row if there is no need to join data from other tables. It will depend on your specific requirement.
2. A SyncApp (a custom built transformation tool) will be subscribed to the event bus.
3. The SyncApp processes the events and may read back into SQL tables for all other related data.

This is a straightforward approach. However, your business may not be so open ended but may be constrained by different circumstances.  For example, the application service may not belong to your team and you may not have access to modify the application to inject the code for publishing into the event bus. The other constraints could be that the table update may happen from more than one workflow in a legacy application which may be unknown to the existing team. The existing team is risk-averse and does not allow us to touch the application code.

In such cases, following approaches addresses capturing the changes off the SQL server.



## Approach 2: SQL Database Trigger

[\[Diagram for approach 2: CDC with DB trigger\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-db-trigger.jpg?raw=true)

1. In this approach, the application writes  (insert, update, delete)  into the tables to be captured.
Triggers are created for the tables to be tracked. The trigger gets the id, operation and timestamp into a table  say <CapturedTableName>_CDC.
2. A scheduled cron job fetches the changed data into <CapturedTableName>_CDC table and publishes an event.
3. The sync app then takes its event to processes the events as required.


Since database triggers are synchronous and it adds some latency to the business transaction, go for this approach if the business can tolerate it. This strategy can be chosen if the changes to the table are not frequent or the workflow transaction does not meet hard SLA requirements.
Lets see next approach.


## Approach 3: SQL CDC

[\[Diagram for approach 3: CDC with SQL CDC\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-app.jpg?raw=true)

Similar to approach 2, the application writes  (insert, update, delete)  into the tables to be captured.
CDC is enabled at DB and table level using system stored procedure sys.sp_cdc_enable_db and sys.sp_cdc_enable_table.
This creates asynchronous triggers for the tables to be tracked. Under the hood, a change capture instance is created and the capture process works off the logs asynchronously.
The capture process gets the id, operation and timestamp into a table  say CDC.CapturedTable
A scheduled custom job polls for the changed information from CDC_<CapturedTableName> table using appropriate system stored procedure like dc.fn_cdc_get_all_changes_<capture_instance>.  The scheduled custom job stored procedure receives the data and inserts into another table say Processor_CapturedTable.
Another scheduled cron job now polls for the non-processed changed information data into Processor_CapturedTabletable, publishes an event and marks the row as processed,
The sync app then takes its event to processes the events as required.

As we see here in step 3, there arises a need to create an additional job and a table because the SQL CDC mechanism captures the changes into a system table. 
One of the straight forward strategies to track the status of the processed rows is to capture in an additional column. Since CDC_<CapturedTableName> is a system table, its data will need to be taken into another table here i.e.Processor_CapturedTable where each row can be marked as processed.  Additionally, the custom scheduled SQL job will need to be built so that it selectively access the new data captured.

Summarizing:
CDC is an asynchronous capture of changes from the transaction log.
The pipeline to publish an event involves two polling jobs and creation of additional user tables.

More details are here at [SQL CDC](https://docs.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver15)


## Approach 4: SQL Server Broker Service (SSSB)

[\[Diagram for approach 4: CDC with SQL SSB\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-sssb.jpg?raw=true)

1. Similar to approach 2 or 3, the application writes (insert, update, delete) into the tables to be captured. 
2. In this approach, SQL Service Broker is enabled and SQL broker objects viz Message Types, Contracts, Queues, Services, Trigger can be created and configured via a stable  .NET Library - SqlTableDepenency
3. SQL broker uses an external application as its activation procedure. The external application here is SyncApp.
4. The syncApp gets notified that a message is ready to be processed which then takes it ahead as explained in previous approaches.



More details are here at [SQL Server Broker Service (SSSB)](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-service-broker?view=sql-server-ver15)

This approach provides 
+ SQL Native Asynchronous framework
+ It is reliable and scalable
+ Since it operates within SQL, it is faster.
+ Users need not implement the low level code to create and configure the SQL broker objects but use the NuGet library to avoid the boilerplate. User only needs to write handler logic after the event is received.

