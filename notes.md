# Thesis prep notes

## Possible levels of integration

1. Could use S3 object store to store a binary files of snapshots

2. Could store only snapshotted data in a relational database, using vmodb query engine. Would allow for querying somewhat older state

3. Could circumvent the entire vmodb database to query a rdbms directly through vmodb querying constructs. How to ensure consistency across databases? 2PC? Can it be done faster?

==Maybe we can use the native postgres distributed 2PC mechanisms to achieve consistency and isolation?==

In any case, a big focus should be on modularity to allow each microservice to decide its own way of storing its data. It should, for example, be possible for one to use postgres for storage and querying, while another uses S3 for snapshots, while another just uses the current vmodb storage and query systems. This would include creating interfaces for storage providers (i.e. how to store data) and
querying (i.e. how to find the data we want). Sometimes these might be tightly coupled. E.g. it does not make sense to store checkpoints in S3, but use Postgres to query data (it would probably not even make sense to store checkpoints in postgres and query state in postgres, as the state would be not be consistent with the changes made in the current transaction), as we would need to replicate the entire state into Postgres. Maybe we just have to rely on the programmer not being stupid? Maybe we could have to levels of interfaces, one `IStorageProvider` and one `IQueryProvider`(`IQueryStorageProvider`?), which would inhierit from `IStorageProvider`. Then create two contructors for the VMS bootstrap, one that uses IStorageProvider and assumes the default vMODB querying system, and one that uses IQueryProvider, that uses both the storage and the query functionality defined in that interface. We just need to add two fields to the VMS one for IStorageProvider and one for IQueryProvider.


We could also extend this and use an external rdbms to store events with timestamps and other metadata, which would allow for atleast external systems to, essentially, make continous queries. One could even use the database driver for the database in the vms directly, to make queries from a vms, though, ideally, this would be baked into the vms sdk.
