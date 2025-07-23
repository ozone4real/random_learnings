# Logical replication

There are 2 kinds of replication:

- Logical replication: Logical changes (inserts, updates, etc) in one database are being streamed to other databases to replicate them.
The exact physical structure of the database isn't replicated. Logical replication is  based upon a replication identity (usually a primary key).
- Physical replication: The exact structure of data (byte-by-byte) and their physical location is replicated.

In a logical replication setup, there must be at least one publisher database (master) and one or more subscriber databases (replicas).
To prevent conflict, the replica databases should ideally be readonly by applications, writes to the replicas must be based on the master.
Logical replication is implemented by streaming decoded WAL records from master to replicas.


## PUBLICATION

- Creating a publication for the whole database (changes in all tables are replicated, including tables created in the future.):

```sql
CREATE PUBLICATION kisi_pub FOR ALL TABLES;
```

- Creating a publication for specific tables:

```sql
CREATE PUBLICATION users_pub FOR TABLE users;
```

- Creating a publication for only specific operations

```sql
CREATE PUBLICATION FOR role_assignments_pub FOR TABLE role_assignments WITH (publish = 'delete, update')
```


# SUBSCRIBING

To create a logical replica (subscriber), the exact schema of the master (publisher) database has to be first created 
in the replica database (using pg_dump + pg_restore/psql copy). Note: For the replication initialization to succeed by default, only the schema
should be created in the replica. So the tables must be empty. This is because by default when replication is initiated, the data in the tables of the master
is first copied to the replicas and then subsequent changes to the master are streamed to the replicas. If the tables in the replicas already
contain data, the initial COPY may result in conflict and hence failure. You could pass a copy_data=false flag to skip copying data if the replica already has
data; in that case the replication will start only from when the subscription/replication slot was created.

To copy the master schema to the replica:

pg_dump -s master_db | psql -d replica_db

## Creating a subscription

A subscription must be associated with a publication. The publication is specified when creating the subscription.

```sql
CREATE SUBSCRIPTION kisi_sub
CONNECTION 'host=192.992.34.56 dbname=master_db application_name=kisi_sub'
PUBLICATION kisi_pub;
```

## Replication slots

A replication slot ensures wal records are streamed from the publisher to subscriber before they are deleted/archived from the publisher.
When a subscription is created, a replication slot is automatically created on the publisher. There are cases where the replication slots may
need to be created manually.
If the publisher and subscriber db are in the same host/cluster, the subsctiption creation described above will hang. See [postgres docs](https://www.postgresql.org/docs/current/sql-createsubscription.html).
To prevent that, create the subscription without a replication slot:

```sql
CREATE SUBSCRIPTION kisi_sub
CONNECTION 'host=192.992.34.56 dbname=master_db application_name=kisi_sub'
PUBLICATION kisi_pub
WITH(create_slot=false);
```

Then create the replication slot in the publisher:

```sql
SELECT * FROM pg_create_logical_replication_slot('kisi_sub', 'pgoutput');
```

Then on the subscriber, update the subscription with the slotname:

```sql
ALTER SUBSCRIPTION sub1 SET (slot_name='kisi_sub');
```

Enable the subscription and refresh the publication:

```sql
ALTER SUBSCRIPTION kisi_sub ENABLE;
ALTER SUBSCRIPTION kisi_sub REFRESH PUBLICATION;
```

## Commands for querying/tracking replication

```sql
SELECT * FROM pg_subscription;
SELECT srsubstate, srrelid::regclass FROM pg_subscription_rel WHERE srsubstate != 'r';  -- Tracks which tables haven't completed initial copy/isn't ready for replication if subscription was created with copy_data=true (the default)
SELECT * FROM pg_subscription_rel -- monitor subscription sync status per relation;
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_stat_replication;
SELECT * FROM pg_replication_slots;
SELECT * FROM pg_publication;
```


### LIMITATIONS OF LOGICAL Replication

- DDL statements/ schema changes aren't replicated
- Changes to large objects (TOAST) aren't replicated.
- Sequences aren't replicated. If a pkey is based on a serial sequence, the pkey will be replicated as part of the table.
The sequence itself will remain at the start value on the replica. If the replica is readonly then this won't be an issue. 
But if the replica is written to (inserts specifically), then there'll be pkey conflicts, since the sequence will begin from the start
which may conflict with already replicated pkeys.


### CAVEATS

You can't shutdown/delete a primary with active logical replication slots. The replication slots must be deleted first either directly from the publisher or by dropping
the subscription from the subscriber