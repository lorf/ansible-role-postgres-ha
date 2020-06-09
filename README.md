postgres-ha
===========

With this role, you will transform your standalone postgresql server to N-node postgres cluster with automated failover. You only need one working postgresql server and other hosts with clean CentOS 7 or CentOS 6 minimal install.

Alternatively, this role can create a database cluster for you from scratch. If no postgres database is detected, it will be created.

What it will do:
- install the cluster software stack (pcs, corosync, pacemaker)
- add IPs of cluster hosts to /etc/hosts files
- create a pcs cluster from all play hosts
- install database binaries if needed
- init master database if needed
- alter postgresql configuration if needed
- sync slave databases from master host
- make sure the DB replication is working
- create cluster resources for database, floating IP and constraints
- check again that everything is working as expected

Automated failover is setup using PAF pacemaker module: https://github.com/dalibo/PAF

What you should know
--------------------

- The role is idempotent. I've made many checks to allow running it multiple times without breaking things. You can run it again safely even if the role fails. The only thing you need to check before the run is the `postgres_ha_cluster_master_host` variable. But don't worry, if the specified host is not the master database, the role will fail gracefully without disrupting things.

- During the run, the role will alter your postgresql.conf and pg_hba.conf to enable replication. You can review the changes to postgresql.conf in [defaults/main.yml](defaults/main.yml) (`postgres_ha_postgresql_conf_vars` variable). In pg_hba.conf, the host ACL statements will be added for every cluster node. They will be added before all previously existing host ACL statements.

- The postgres replication is asynchronnous by default. If you want synchronnous replication, alter the `postgres_ha_postgresql_conf_vars` variable by adding `synchronous_standby_names` parameter. Please see postgresql manual for more info. Also note that if the last synchronnous replica disconnects from master, the master database will stop serving requests.

- You should have at least a basic understanding of clustering and how to work with `pcs` command. If the role fails for some reason, it is relatively easy to recover from it.. if you understand what logs are trying to say and/or how to run appropriate recovery actions. See cleanup section for more info.

- You need to alter firewall settings before running this role. The cluster members need to communicate among each other to form a cluster and to replicate postgres DB. I recommend adding some firewall role before the postgres-ha role.

- If the master datadir is empty on the first run, the role will init an empty datadir. Slave nodes will then download this empty database. If the datadir is not empty, the initdb will be skipped. This means that you can run this role on clean CentOS installs that don't have any postgresql database installed. The result will be fully working empty database cluster.

- On the first run, the datadirs on slave nodes will be deleted without prompt. Please make sure you specify the correct `postgres_ha_cluster_master_host` at least for this first run (slave datadirs will NEVER be deleted after first initial sync is done).

- If you plan to apply the role to higher number of servers (7+) please be aware that the servers are downloading rpms packages simultaneously. This can be identified as DDoS and some repository providers may refuse your downloads. As a result, the role will fail. I recommend setting up your own repository mirror in such cases.

- Please don't change the cluster resource name parameters after the role has been applied. In next run, it will result in trying to create the new colliding resources.

- Fencing is not configured by this role. If you need one, you have to configure it manually after running the role.

Requirements
------------

This role works on CentOS 6 and 7. RHEL was not tested but should work without problem. If you need support for other distribution, I can help. Post an issue.

The postgresql binaries on your primary server should be installed from the official repository:

https://yum.postgresql.org/repopackages.php

Note: If you have binaries from other repo, you need to modify the `postgres_ha_repo_url` variable to change the postgres repository source and maybe also bindir and datadir paths in other role variables. If you need to change the installed package name(s), you need to directly modify `install pg*` task in `tasks/postgresql_sync.yml` file.

Role Variables
--------------

For all variables with description see [defaults/main.yml](defaults/main.yml)

Variables that must be changed:
- `postgres_ha_cluster_master_host`        -    the master database host (WARNING: please make sure you fill this correctly, otherwise you may lose data!)
- `postgres_ha_cluster_vip`                -    a floating IP address that travels with master database
- `postgres_ha_pg_repl_pass`               -    password for replicating postgresql data
- `postgres_ha_cluster_ha_password`        -    password for cluster config replication

Dependencies
------------

No other roles are required as a dependency. However you can combine this role with some other role that installs a postgresql database.

Example Playbook
----------------

The usage is relatively simple - install minimal CentOS-es, set the variables and run the role.

Two settings are required:
- `gather_facts=True`        - we need to know the IP addresses of cluster nodes
- `any_errors_fatal=True`    - it ensures that error on any node will result in stopping the whole ansible run. Because it doesn't make sense to continue when you lose some of your cluster nodes during transit.

```
    - name: install PG HA
      hosts: db?
      gather_facts: True
      any_errors_fatal: True
      vars:
        postgres_ha_cluster_master_host: db1
        postgres_ha_cluster_vip: 10.10.10.10
        postgres_ha_pg_repl_pass: MySuperSecretDBPass
        postgres_ha_cluster_ha_password: AnotherSuperSecretPass1234
      pre_tasks:
        - name: disable firewall
          service: name=firewalld state=stopped enabled=no
      roles:
         - postgres-ha
```

Cleanup after failure
---------------------

If the role fails repeatedly and you want to run it fresh as if it was the first time, you need to clean up some things.
Please note that default resource names are used here. If you change them using variables, you need to change it also in these commands.

- RUN ON ANY NODE:
```
pcs resource delete pg-vip
pcs resource delete postgres
#pcs resource delete postgres-ha   # probably not needed
#pcs resource cleanup postgres     # probably not needed

# Make sure no (related) cluster resources are defined.
```
- RUN ON ALL SLAVE NODES:
```
systemctl stop postgresql-9.6
# Make sure no postgres db is running.
systemctl status postgresql-9.6
ps aux | grep postgres
rm -rf /var/lib/pgsql/9.6/data
rm -f /var/lib/pgsql/9.6/recovery.conf.pgcluster.pcmk
rm -f /var/lib/pgsql/9.6/.*_constraints_processed   # name generated from postgres_ha_cluster_pg_res_name
```
- RUN ONLY ON MASTER NODE:
```
systemctl stop postgresql-9.6
rm -f /var/lib/pgsql/9.6/recovery.conf.pgcluster.pcmk
rm -f /var/lib/pgsql/9.6/.*_constraints_processed
rm -f /var/lib/pgsql/9.6/data/recovery.conf
rm -f /var/lib/pgsql/9.6/data/.synchronized
# Make sure no postgres db is running.
ps aux | grep postgres
systemctl start postgresql-9.6
systemctl status postgresql-9.6
# Check postgres db functionality.
```
- START AGAIN
```
# Check variables & defaults and run ansible role again.
```


License
-------

BSD

Author Information
------------------

Created by YanChi.

Originally part of the Danube Cloud project (https://github.com/erigones/esdc-ce).

