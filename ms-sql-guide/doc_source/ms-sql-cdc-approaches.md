# Primer on CDC Strategy in MS SQL

## Purpose
The purpose of this guide is to approach a common business problem of tracking changes to the tables in the SQL server. 

There could be use cases where an impact in one system needs to propagate to another system in the business ecosystem. 
For example, new users onboarding into a service provider’s server require a confirmation via an email or an SMS. Such notification handlers may already be part of another system. There arises a need for the communication between the core system and the notification system, which we generally achieve by an event-driven mechanism.
Similarly, another use case could be where we need to synchronize two system servers for a specific business need. 
We have more than one way to approach this kind of business requirement, and this primer helps us get started.


## Approach 1: Application sends changed information to an event bus.


![\[Diagram for approach 1: CDC with App\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-app.jpg?raw=true)

1. In this approach, as soon as an application service writes (insert, update, delete) into tables to be captured, the same application also publishes an event on an event bus. The event may just contain the unique id of the record, the operation(op), and timestamp of op. Alternatively, one may consider publishing an entire row if there is no need to join data from other tables. It will depend on one's specific requirements.
2. A SyncApp (a custom-built transformation tool) is subscribed to the event(s).
3. The SyncApp processes the events and may read back into SQL tables for all other related data.

This is a straightforward approach. However, your business may not be so open-ended but may be constrained by different circumstances.  For example, the application service may not belong to your team, and you may not have access to modify the application to inject the code for publishing into the event bus. The other constraints could be that the table update may happen from more than one workflow in a legacy application, which may be unknown to the existing team. The existing team could be risk-averse and does not allow us to touch the application code.

In such cases, the following approaches address capturing the changes off the SQL server.


## Approach 2: SQL Database Trigger

![\[Diagram for approach 2: CDC with DB trigger\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-db-trigger.jpg?raw=true)

1. In this approach, the application writes  (insert, update, delete)  into the tables to be captured.
Triggers are created for the tables to be tracked. The trigger gets the id, operation, and timestamp into a table say CDC_CapturedTable.
2. A scheduled cron job fetches the changed data from the CDC_CapturedTable table and publishes an event.
3. The SyncApp, on receiving the event, processes as required.


Since database triggers are synchronous, and they add some latency to the business transaction, my recommendation will be to go for this approach only if the business SLA can endure some additional delay. Choose this strategy if the changes to the table are not frequent or the workflow transaction does not need to meet hard SLA requirements.

Let's see the next approach.

## Approach 3: SQL CDC

![\[Diagram for approach 3: CDC with SQL CDC\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-cdc.jpg?raw=true)

1. Similar to approach 2, the application writes (insert, update, delete) into the tables to be captured.
2. CDC is enabled at DB and table level using system stored procedure sys.sp_cdc_enable_db and sys.sp_cdc_enable_table respectively. This creates asynchronous triggers for the tables to be tracked. Under the hood, a change capture instance is created, and its capture process works off the transaction logs asynchronously.
3. The capture process intrinsically gets the id, operation, and timestamp into a table, say, CDC_CapturedTable.
4. A scheduled custom job polls for the changed information from CDC_CapturedTable using appropriate system stored procedures like dc.fn_cdc_get_all_changes_{capture_instance}. The scheduled custom job stored procedure receives the data and inserts it into another table, say, Processor_CapturedTable.
5. A second scheduled cron job now polls for the non-processed changed information data from Processor_CapturedTable, publishes an event, and marks the row as processed.
6. The SyncApp, on receiving the event, processes as required.

As we see here in step 3 and 4, there arises a need to create an additional job and a table because SQL CDC mechanism captures the changes into a system table. 
One of the straight-forward strategies to track the status of the processed rows is to set a flag in an additional column. Since the CDC_CapturedTable is a system table, its data will need to be taken into another table here i.e., Processor_CapturedTable, where each row can be marked as processed.  Additionally, the custom scheduled SQL job is needed to be built such that it selectively accesses the new data captured.

The summary of this approach is that the CDC is an asynchronous capture of changes from the transaction log.
The pipeline to publish an event involves two polling jobs and the creation of additional user tables.

More details can be read here at [SQL CDC](https://docs.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver15)


## Approach 4: SQL Server Service Broker (SSSB)

![\[Diagram for approach 4: CDC with SQL SSB\]](https://github.com/surbhi-nijhara/techtumblr/blob/master/ms-sql-guide/diag_source/ms-sql-cdc-sssb.jpg?raw=true)

1. Similar to approach 2 and 3, the application writes (insert, update, delete) into the tables to be captured. 
2. In this approach, SQL Service Broker is enabled and SQL broker-objects viz. Message Types, Contracts, Queues, Services and Trigger can be created and configured via a stable  .NET Library - [SqlTableDepenency](https://www.nuget.org/packages/SqlTableDependency/)
3. SQL broker uses an external application as its activation procedure. The external application here is SyncApp.
4. The SyncApp gets notified that a message is ready to be processed, which then takes it ahead, as explained in previous approaches.

More details are here at [SQL Server Broker Service (SSSB)](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-service-broker?view=sql-server-ver15)

Summary of this approach is that it provides 
+ SQL Native Asynchronous framework
+ It is reliable and scalable
+ Since it operates within SQL, it is faster.
+ Users need not implement the low-level code to create and configure the SQL broker-objects but use the NuGet library to avoid the boilerplate. The user only needs to write handler logic after the event is received.

An argument could be that enabling SSSB and having SQL Server now perform additional operations may impact the overall performance of SQL synchronous business transactions under a stress test. These additional operations could be what goes into the creation of all SQL broker objects like messages, queues, services, contracts, etc., followed by the execution of this additional internal messaging mechanism.

Folks who have used SQL Service broker have given good feedback. There has been no performance impact seen even when they have used service broker in use cases like data warehouse with SQL server configured on a relatively low-end machine. 
Performance benefits can also be read in detail [here](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008/dd576261(v=sql.100)). Check for more responses [here](https://stackoverflow.com/questions/61590787/sql-server-service-broker-performance-impact).

On a concluding note, one of the above four approaches should meet the business use case and constraint if any. 

