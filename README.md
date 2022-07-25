Backup of StatefulSet cassandra prior to K8ssandra migration:
--------------------------------------------------------------
- The procedure allows to migrate from Cassandra in a vanilla STS to K8ssandra
- The backup will be used in case something goes wrong after the migration (although you could restore that backup to the K8ssandra cluster as well)

Toolkit used:
-------------------
- A s3 storage for backup/restore test on the existing STS cluster. ( public or onprem ). Pick an appropriate s3 intergation. https://github.com/thelastpickle/cassandra-medusa/tree/master/docs
- The repo uses medusa toolkit on the STS cluster manifest. https://github.com/thelastpickle/cassandra-medusa
- Implement the below Steps.

Steps:
------
1) A prior cassandra sts must be runnign similar to https://kubernetes.io/docs/tutorials/stateful-application/cassandra/. For example when you run the command..
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

2) Backup your statefulset cluster. ( see backup-statefulset directory)
3) Restore test the medusa backup copy. ( see restore directory )
4) Steps below for k8ssandra migration(it has some common things with the step 1, see the migration directory in the repo)
5) Set the replicas for STS cluster to 0 and keep the PV for a the time till you are confident to delete them to save cost. 
```
kubectl scale statefulsets <sts-cassandra-name> --replicas=0
```


References:
----------
https://k8ssandra.io/blog/tutorials/cassandra-database-migration-to-kubernetes-zero-downtime/
