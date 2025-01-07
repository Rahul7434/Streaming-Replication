# Streaming-Replication Postgresql
This replication is based on log shipping. In general primary server sends  WAL (Write-Ahead Log) data to Standby server. This setup provodes Advantage high availability (standby server is redy to take in caseprimary failure) and load balancing (read queries can be directed to standby server, reducing load on the primary) in postgresql.
#### Postgresql support two main types of streaming replication based on how the wal(write Ahead Log) data is sent and applied on the standby server.
- Synchronous Replication
- Asyncronous Replication
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
##### Synchronous Replication:- 
- Ensures that Transaction on the primary server are only considered committed after the data is writen to the wal on the strandby server.
1. Provides zero data loss.
2. Acknowledges commits only after at least one standby confirms it has received and written the WAL.
3. Slower than asynchronous replication due to the wait time for standby acknowledgment.
4. For Sayncronous need to add application name to the "syncronous_standby_name" parameter.
\'\'\'
synchronous_standby_names = 'FIRST 1 (<standby_name1>, <standby_name2>)'
\'\'\'
  
    
