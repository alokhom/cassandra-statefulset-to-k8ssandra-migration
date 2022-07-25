Backup of Stateful cassandra:
------------------------------
- The procedure allows to migrate from Cassandra in a vanilla STS to K8ssandra
- The backup will be used in case something goes wrong after the migration (although you could restore that backup to the K8ssandra cluster as well)

Audience: 
----------
- Those who want to migrate from STS cassandra to k8ssandra and are referring to https://k8ssandra.io/blog/tutorials/cassandra-database-migration-to-kubernetes-zero-downtime/


What is Addresses:
-----------------------------
- How to use medusa on statefulset cassandra?


What you will need:
-------------------
- a s3 storage for backup/restore test on the existing STS cluster. ( public or onprem )
- Pick an appropriate s3 intergation. https://github.com/thelastpickle/cassandra-medusa/tree/master/docs
- Below Steps.

Steps:
------
1) A prior cassandra sts must be runnign similar to https://kubernetes.io/docs/tutorials/stateful-application/cassandra/. 

For example when you run the command..
```
$ nodetool status
Datacenter: datacenter1
=====================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load      Tokens  Owns (effective)  Host ID                               Rack
UN  172.31.4.217   10.2 GiB  16      100.0%            9a9b5e8f-c0c2-404d-95e1-372880e02c43  us-west-2c
UN  172.31.38.15   10.2 GiB  16      100.0%            1e6a9077-bb47-4584-83d5-8bed63512fd8  us-west-2b
UN  172.31.22.153  10.2 GiB  16      100.0%            d6488a81-be1c-4b07-9145-2aa32675282a  us-west-2a
```
Proceed with below steps if above is a success.

2) Backup your statefulset cluster. ( see the backup-statefulset directory in this repo)
3) Restore test the medusa backup copy. ( see restore directory in this repo )
4) Steps below for k8ssandra migration(it has some common things with the step 1, see the migration directory in the repo)

Suggestions:
-----------
Any suggestions for improvement are welcome. 



