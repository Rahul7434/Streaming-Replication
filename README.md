# Streaming-Replication Postgresql
This replication is based on log shipping. In general primary server sends  WAL (Write-Ahead Log) data to Standby server. This setup provides Advantage high availability (standby server is ready to take care in case primary failure) and load balancing (read queries can be directed to standby server, reducing load on the primary) in postgresql.
#### Postgresql support two main types of streaming replication based on how the wal(write Ahead Log) data is sent and applied on the standby server.
- Streaming Replication(physical replication).
- Logical Replication
- Cascading Replication
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
##### Synchronous:- 
- Ensures that Transaction on the primary server are only considered committed after the data is writen to the wal on the strandby server.
1. Provides zero data loss.
2. Acknowledges commits only after at least one standby confirms it has received and written the WAL.
3. Slower than asynchronous replication due to the wait time for standby acknowledgment.
4. For Sayncronous replication need to add application name to the "syncronous_standby_name" parameter.
```
synchronous_standby_names = 'FIRST 1 (<standby_name1>, <standby_name2>)'
```
##### Asynchronous
- Transactions on the primary server are considered committed as soon as the WAL is written to the disk on the primary, without waiting for acknowledgment from standby servers.
1. Higher performance due to non-blocking behavior.
2. There is a risk of data loss in case the primary server crashes before the WAL reaches the standby.
3. This is the default mode of replication.
4. No specific setup for synchronous_standby_names is required.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Configuration: -
- Install Postgresql with same version on both servers Standby & primary, but do not initialize DB on standby serever (note- we can setup with different version but some functionality may be will give issue).
    - Go to on postgresql.org official website
    - Select Linux operating system
    - click on dirstribution "Red Hat/Rocky/AlmaLinux" it will open the page for download server based on versions.
    - select postgresql version that you want to install and select your RHL vesion platform version with architecture (32,64 bit).
    - After all it will provide you installation steps.
      ```
      # Install the repository RPM:
          sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
      # Disable the built-in PostgreSQL module:
          sudo dnf -qy module disable postgresql
      # Install PostgreSQL:
          sudo dnf install -y postgresql17-server

      # Optionally initialize the database and enable automatic start:
          sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
          sudo systemctl enable postgresql-17
          sudo systemctl start postgresql-17
      ```
- Both server should able to communiate over the network.
### Configure the Primary Server
- Enable WAL Archiving and Streaming
- Modify the postgresql.conf file on the primary server:
```
listen_addresses = '*'    --it will listen for any Ip addreses if we set it * and it's default Ip value is 127.0.0.1
 
wal_level = replica  --
 
max_wal_senders = <number_of_standby_servers> (e.g., 3) --Each standby server needs its own sender process to stream wal from prod.
 
wal_keep_size = <sufficient_size> (e.g., 64MB or 128MB)
 
archive_mode = on (optional if WAL archiving is used)

archive_command = '/bin/true' (placeholder if archiving isn't needed)
```
- Allow connections from standby server. Edit pg_hba.conf file add entry to allow connections. And restart the primary server to apply the changes.
```
host replication all <standby_server_ip>/32 md5
# To restart
systemctl restart postgresql-16 or pg_ctl /datadir/path/ restart 
```
- Create a Replication Role On the primary server, create a user with replication privileges:
```
CREATE ROLE replicator WITH REPLICATION PASSWORD 'your_password' LOGIN;
```
- Copy Data from Primary to Standby, Use pg_basebackup to create a replica of the primary:
  ```
  pg_basebackup -h <primary_server_ip> -U replicator -D <standby_data_directory> -Fp -Xs -P -R
  ```
    - Note:- The -R flag automatically creates the standby.signal file and a primary_conninfo entry in postgresql.auto.conf. other wiae you need to create standby.signal file and add primary_conninfo entry in postgresql.conf file.
    ```
    primary_conninfo = 'host=primary.example.com port=5432 user=replicator password=secretpassword application_name=my_standby'
    ```
      - Verify standby.signal File
Ensure the standby.signal file is present in the data directory.

      - Check postgresql.auto.conf
    The pg_basebackup command adds the following to postgresql.auto.conf:

### Configure Standby Server Parameters
```
# Modify the postgresql.conf file on the standby server (optional tweaks):
hot_standby = on
```
- Start the Standby Server
 ```
  systemctl start postgresql-16
 ```
- Verify Replication
```
# On the primary server, check replication status:
SELECT * FROM pg_stat_replication;
# On the standby server, check recovery status:
SELECT pg_is_in_recovery();
```
------------------------------------------------------------------------------------------------------------------------------------------------------------------------
### How communicate both servers
- when we Start the primary and standby servers.
- The standby server starts the "startup" process & "wal receiver".
- The "walreceiver" process sends a connection request to the primary server. If the primary server is not running, the walreceiver sends these requests periodically.
  
- When the primary server receives a connection request, it starts a walsender process and a TCP connection is established between the walsender and walreceiver.
- Then walreceiver sends the latest LSN (Log Sequence Number) of standby’s database cluster. 
- If the standby’s latest LSN is less than < the primary’s latest LSN (Standby’s LSN  <Primary’s LSN), the walsender sends WAL data from the former LSN to the latter LSN. These WAL data are provided by WAL segments stored in the primary’s pg_wal . The standby server then replays the received WAL data.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### Monitor Streaming replication

- To monitor Active replication connections on the primary server or it's [current state,sent_lsn , write_lsn, flush_lsn , sync_state ]

``` 
select * from pg_stat_replication;
 ```
- To monitor Current received wal on standby server.

```
select * from pg_stat_wal_receiver;
```
- To track Archive Activity'
  ```
  select * from pg_stat_archiver;
  ```
- To display infor about replication slot
  ```
  select * from pg_stat_replication_slot;
  ```
- To monitor standby server replication setup
```
select pg_last_wal_replay_lsn(), pg_is_in_recovery(), pg_current_wal_lsn();

```

- To Calculate Replication Lag on primary server.
```
SELECT client_addr,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, write_lsn)) AS write_lag,
       pg_size_pretty(pg_wal_lsn_diff(write_lsn, flush_lsn)) AS flush_lag,
       pg_size_pretty(pg_wal_lsn_diff(flush_lsn, replay_lsn)) AS replay_lag
FROM pg_stat_replication;
```
- To Monitor wal archiving or streaming delays on sandby server.
```
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
### Replication Slot
- On Postgresql server, old wal files that are no longer needed  are deleted. 
- I this case if standby server has network issue or if it is down and in this perticular time primary server will may be remove wal-files which are no longer needed from primary. 
- If we do not have Archiving enabled for wal files our data will be loss and we can't make standby server up because we do nothave wal file for that perticular time.
- To avoid this issue prostgresql provides the replication_slot, It ensures that which  wal-files are yet to be written needed to the standby server. It prevents those files from being deleted on the primary until they are succesfully written tothe standby.

- To create replication slot on primary :- 
```
SELECT * FROM pg_create_physical-replication_slot('slot_name');

```
- Add slot name to "primar_slot_name" parameter on standby server confuguration file.
```
	primary_slot_name = "slot_name"
```

```
-- Last WAL received
SELECT pg_last_wal_receive_lsn();

-- Last WAL replayed
SELECT pg_last_wal_replay_lsn();

-- Time of last replay
SELECT pg_last_xact_replay_timestamp();

-- Replay lag in seconds
SELECT EXTRACT(EPOCH FROM now() - pg_last_xact_replay_timestamp()) AS replay_lag;



-- Check if the server is in recovery mode (i.e., standby)
SELECT pg_is_in_recovery();

-- Shows details about the WAL receiver process on the standby
SELECT * FROM pg_stat_wal_receiver;

SELECT * FROM pg_stat_recovery_prefetch;

SELECT * FROM pg_replication_slots;

log_replication_commands = on

```
  
    
