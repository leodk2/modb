# Thesis prep notes

## Possible levels of integration

1. Could use S3 object store to store a binary files of snapshots

2. Could store only snapshotted data in a relational database, using vmodb query engine. Would allow for querying somewhat older state

3. Could circumvent the entire vmodb database to query a rdbms directly through vmodb querying constructs. How to ensure consistency across databases? 2PC? Can it be done faster?

In any case, a big focus should be on modularity to allow each microservice to decide its own way of storing its data. It should, for example, be possible for one to use postgres for storage and querying, while another uses S3 for snapshots, while another just uses the current vmodb storage and query systems. 

We could also extend this and use an external rdbms to store events with timestamps and other metadata, which would allow for atleast external systems to, essentially, make continous queries. One could even use the database driver for the database in the vms directly, to make queries from a vms, though, ideally, this would be baked into the vms sdk.
