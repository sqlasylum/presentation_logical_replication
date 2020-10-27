# Logical Replication Presentation Code and Supporting Items 

## Getting started 

# Database.env and Database_standby.env files. 

The yaml files depend on these 2 files. Database.env and Database_Standby.env An example of each is included below. 

--Database.env 

```
# database.env
POSTGRES_USER=master
POSTGRES_PASSWORD=<password>
POSTGRES_DB=master
```

--Database_standby.env 
```
# database.env
POSTGRES_USER=standby
POSTGRES_PASSWORD=<password>
POSTGRES_DB=standby
```

## Starting up Containers
After you have placed these files In the directory of your docker-compose.yaml, then run. 

``` docker compose up ```

## Shutting down 

``` docker compose down ```

## Initialize the Bench tables 

``` pgbench -h localhost -U master -i -s 10 master ```

## Run Bench to have some work in the background some work on the tables(long run) -- not heavy just long for demo purposes 

``` pgbench -U master -h localhost -p 5432 -P 10 -r -T 18000 -c 1 -j 1 master ```

## Create publication on Master
``` create publication bench for table  pgbench_history with (publish = 'insert,update,delete'); ```

## Table Creation on the subscription 

``` 
CREATE TABLE if not exists  public.pgbench_history ( 
 tid    integer,                     
 bid    integer,                     
 aid    integer,                     
 delta  integer,                     
 mtime  timestamp without time zone, 
 filler character(22),
 id serial primary key
) 
```

## Create subscription run on standby

``` 
CREATE SUBSCRIPTION bench_history 
CONNECTION 'host=postgres_master port=5432 user=master dbname=master password=<password>' 
PUBLICATION bench WITH (
    copy_data = TRUE
); 
```

# Validation
Run counts on both Master and Standby

``` select count(*) from pgbench_history; ```

# Table Changes 
--Change table on Master

``` alter table pgbench_history add column pw_test char(10);  ```

--Check log, This is similar to the error you will see. 
``` posres_standby | 2020-10-25 21:49:52.798 UTC [104] ERROR:  logical replication target relation "public.pgbench_accounts" is missing some replicated column ```
--Change table on standby

``` alter table pgbench_history add column pw_test char(10); ```

# Config suggestions/items 

wal_level = 'logical'

--All dependant on the number of subscribers.  
max_wal_senders                
max_replication_slots
max_worker_processes           
max_logical_replication_workers 
max_sync_workers_per_subscription #of tables per subscription

# Dropping/removing 

--Drop replication slot 

``` select pg_drop_replication_slot('slot_name'); ```
``` drop subscriptiong <subscription_name> ``` 
``` drop publication <publication_name> ```

--If disconnected from publisher/subscriber 

``` 
ALTER SUBSCRIPTION bench_history disable;
ALTER SUBSCRIPTION bench_history SET (slot_name = NONE);
DROP SUBSCRIPTION bench_history; 
```

# Monitoring 
To check Latency of replication.
Connect to Publisher and execute

```Select * from pg_stat_replication;```

--The Write lag, Flush Lag and Replay Lag should all be very low if anything at all.

--See a list of Publications 

``` Select * from pg_publication ```

--See a list of Publication Tables 

``` Select * from pg_publication_tables ```

--See a list of Subscriptions 

``` Select * from pg_subscription ```

--See a list of replication slots on master

``` Select * from pg_replication_slots ``` 

--Check to see if they are active. 

--Check amount of distance behind on a replication 

``` 
SELECT redo_lsn, slot_name,restart_lsn, 
round((restart_lsn-redo_lsn) / 1024 / 1024 / 1024, 2) AS GB_behind 
FROM pg_control_checkpoint(), pg_replication_slots; 

```





