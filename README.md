# Postgres HA setup with Patroni
This document will walk through the steps to perform PostgreSQL HA setup (1 master and 2 standbys) with Patroni. 

### Environment details - 6 Linux VMs of following
|VM Name |	Purpose	| IP address| Description |
|---|---|---|-----|
|pgvm1 |	Postgres, Patroni	| 50.51.52.81|Postgresql Master node | 
|pgvm2	| Postgres, Patroni	| 50.51.52.82|Postgresql standby node 1 | 
|pgvm3	| Postgres, Patroni |	50.51.52.83|Postgresql standby node 2 | 
|pgvm4	| etcd |	50.51.52.84| Distributed configuration store |
|pgvm5 |	HAProxy	| 50.51.52.85| Single endpoint for connecting to the cluster's leader |
|pgvm6 |	pgbackrest repository	| 50.51.52.86| Backup repository server |

### VMs setup
OS release version of all the Linux VMs used for this setup.</br>
`root@pgvm1:~# lsb_release -a`</br>
`No LSB modules are available.`</br>
`Distributor ID: Ubuntu`</br>
`Description:    Ubuntu 18.04.5 LTS`</br>
`Release:        18.04`</br>
`Codename:       bionic`</br></br>


Execute following scripts to install the required packages.
1. Install Postgres 12, Patroni on VMs pgvm1, pgvm2 & pgvm3 by executing the following scripts.
[setup/install_postgres.sh](https://github.com/farisahamadh/pgsql-ha/blob/main/setup/install_postgres.sh)</br>
[setup/install_patroni.sh](https://github.com/farisahamadh/pgsql-ha/blob/main/setup/install_patroni.sh)</br>

2. Install etcd on pgvm4 by executing the following script.
[setup/install_etcd.sh](https://github.com/farisahamadh/pgsql-ha/blob/main/setup/install_etcd.sh)</br>

3. Install HAProxy on pgvm5 by executing the following script.
[setup/install_haproxy.sh](https://github.com/farisahamadh/pgsql-ha/blob/main/setup/install_HA.sh)</br>

4. Install pgbackrest on pgvm1,pgvm2,pgvm3,pgvm6 by executing the following script.
[setup/install_pgbackrest.sh](https://github.com/farisahamadh/pgsql-ha/blob/main/setup/install_pgbackrest.sh)</br>

### Configuration
###### etcd
etcd is an open source distributed key-value store used to hold and manage the critical information that distributed systems need to keep running. Patroni makes use of  etcd to keep the Postgres cluster up and running.

On pgvm4, Update etcd configuration  [/etc/default/etcd](https://github.com/farisahamadh/pgsql-ha/tree/main/config/pgvm4/etcd) with following values.
`ETCD_LISTEN_PEER_URLS="http://50.51.52.84:2380"`</br>
`ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://50.51.52.84:2379"`</br>
`ETCD_INITIAL_ADVERTISE_PEER_URLS="http://50.51.52.84:2380"`</br>
`ETCD_INITIAL_CLUSTER="etcd=http://50.51.52.84:2380"`</br>
`ETCD_INITIAL_CLUSTER_STATE="new"`</br>
`ETCD_INITIAL_CLUSTER_TOKEN="etcd-pg-cluster"`</br>
`ETCD_ADVERTISE_CLIENT_URLS="http://50.51.52.84:2379"`</br>

Save and close the file when fiinished and start etcd with the below command.
`systemctl start etcd`</br>

Check the status of of etcd with following commands.
`root@pgvm4:~# systemctl status etcd`</br>
`● etcd.service - etcd - highly-available key value store`</br>
`   Loaded: loaded (/lib/systemd/system/etcd.service; disabled; vendor preset: enabled)`</br>
`   Active: active (running) since Sat 2020-10-31 04:47:35 UTC; 5h 49min ago`</br>
`     Docs: https://github.com/coreos/etcd`</br>
`           man:etcd`</br>
` Main PID: 1582 (etcd)`</br>
`    Tasks: 11 (limit: 4632)`</br>
`   CGroup: /system.slice/etcd.service`</br>
`           └─1582 /usr/bin/etcd`</br>

`root@pgvm4:~# etcdctl cluster-health`</br>
`member 8e9e05c52164694d is healthy: got healthy result from http://50.51.52.84:2379`</br>
`cluster is healthy` </br>

`root@pgvm4:~# etcdctl member list`</br>
`8e9e05c52164694d: name=pgvm4 peerURLs=http://localhost:2380 clientURLs=http://50.51.52.84:2379 isLeader=true`</br>


###### Patroni and Potgresql

Create Patroni configuration files for pgvm1,pgvm2 and pgvm3 and ensure following configuration parameters are refering the correct server. 
For example,

> name:<b>pgvm1</b></br>
>restapi:</br>
>   listen: <b>50.51.52.81:8008</b> </br>
>   connect_address: <b>50.51.52.81:8008</b> </br>
>etcd:
>    host: <b>50.51.52.84:2379</b>
  - host replication replicator <b>50.51.52.81/0 md5</b> </br>
  - host replication replicator <b>50.51.52.82/0 md5</b> </br>
  - host replication replicator <b>50.51.52.83/0 md5</b> </br>
>  postgresql:
>  listen: <b>50.51.52.81:5432</b> </br>
>  connect_address: <b>50.51.52.81:5432</b> </br>

Files used in this setup are as follows.
pgvm1: [/etc/patroni.yml](https://github.com/farisahamadh/pgsql-ha/blob/main/config/pgvm1/patroni.yml)
pgvm2: [/etc/patroni.yml](https://github.com/farisahamadh/pgsql-ha/blob/main/config/pgvm2/patroni.yml)
pgvm3: [/etc/patroni.yml](https://github.com/farisahamadh/pgsql-ha/blob/main/config/pgvm3/patroni.yml)













