# Thesis prep notes

==These are the rambling of a usually tired man==


Quick outline of three levels of integration and increasing difficulty. Each might be expanded further below.
## Possible levels of integration

1. Could use S3 object store to store a binary files of snapshots

2. Could store only snapshotted data in a relational database, using vmodb query engine. Would allow for querying somewhat older state

3. Could circumvent the entire vmodb database to query a rdbms directly through the vmodb querying interfaces. How to ensure consistency across databases? 2PC? Can it be done faster?

==Maybe we can use the native postgres distributed 2PC mechanisms to achieve consistency and isolation?==

In any case, a big focus should be on modularity to allow each microservice to decide its own way of storing its data. It should, for example, be possible for one to use postgres for storage and querying, while another uses S3 for snapshots, while another just uses the current vmodb storage and query systems. This would include creating interfaces for storage providers (i.e. how to store data) and
querying (i.e. how to find the data we want). Sometimes these might be tightly coupled. E.g. it does not make sense to store checkpoints in S3, but use Postgres to query data (it would probably not even make sense to store checkpoints in postgres and query state in postgres, as the state would be not be consistent with the changes made in the current transaction), as we would need to replicate the entire state into Postgres. Maybe we just have to rely on the programmer not being stupid? Maybe we could have to levels of interfaces, one `IStorageProvider` and one `IQueryProvider`(`IQueryStorageProvider`?), which would inhierit from `IStorageProvider`. Then create two contructors for the VMS bootstrap, one that uses IStorageProvider and assumes the default vMODB querying system, and one that uses IQueryProvider, that uses both the storage and the query functionality defined in that interface. We just need to add two fields to the VMS one for IStorageProvider and one for IQueryProvider.


We could also extend this and use an external rdbms to store events with timestamps and other metadata, which would allow for atleast external systems to, essentially, make continous queries. One could even use the database driver for the database in the vms directly, to make queries from a vms, though, ideally, this would be baked into the vms sdk.


## External storage with S3 and querying with vMODB


## External storage with Postgres and querying with vMODB


## External storage and querying with Postgres

Here are some thoughts when it comes to using an external DBMS to both query and store state
### How to ensure isolation with an external database

We could implement multiversioning on top of the database. For each data item, we store both the data item, and the tid that changed that made the change resulting in the current state. We store the history of the dataitem through the increasing tid. When reading, a transaction cannot read a change with a larger tid than itself, and will only read the first sealed dataitem with `tid < currTid`, ensuring isolation and read-your-writes consistency. If the `tid == currTid`, then the change is part of this transaction, and the change should be visible to the client. I.e.
 ``` sql
select *
from customer
where
    (changedTid < @currTid and isSealed is TRUE) or
    (changedTid == @currTid and isSealed is FALSE);
 ```
 It does not make sense to check `changedTid == currTid and isSealed is TRUE`, since the transaction would have committed before it had run to completion.
 
 The primary key would then be the tuple `(pk_1,pk_2,...,pk_n, tid)`, where `pk_1,pk_2,...,pk_n` are the fields with the `@Id` annotation. When reading dataitems by id, the database integration layer would then automatically get the latest version of the dataitem transparrently from the developer by adding a where clause and limiting the number of returned rows to 1.
I suppose this might fall under some sort of eventual consistency? Since we store the tid that made a change, we can very easily roll-back changes for transactions that should abort. Or maybe some sort of flag on the data item to show that a transaction failed? I think we can copy the way changes are rolled back in the current transaction abort protocol. Also, how to signal that a dataitem is deleted? There must be something better than deleting the all versions of the dataitem. If there is no better way, how would we roll back such a deletion?

Maybe we can use [postgres temporary tables](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-TEMPORARY), but it would require us to replicate the entire table state to the temporary table.

To ensure that the table doesn't grow indefinitely, we might be able to use some sort of garbage collection to remove old versions of dataitems, that are no longer needed, similar to how it is already done in the current vMODB multiversion database (I think).

With this I don't think we need 2PC. We still need the sealing to mark a transaction as committed. To make this fast we should have an index on `(tid, isSealed)`. 

### How to ensure consensus

Maybe we could model it as a nested transaction? The coordinator/proxy would be the client, and each microservice would each be a client in their own subtransacitons. This would probably require blocking while waiting for a result from each sub transaction through some sort of coordinator message. This would lead to high latency. Not good enough.

The coordinator knows when a transactionDAG is finished, maybe we could then multicast a transaction commit/abort message to the relevant VMSes, when a (batch of) transaction(s) have should commit/abort. We can rely on the VMS transaction scheduler to not schedule transactions that use the same partitions of data, providing a sort of isolation.

How to communicate backwards in transactionDAGs to commit? Maybe the coordinator can multicast a commit transaction commit message to the VMSes? How is consistency achieved now?
