Cassandra statefulset cluster to k8ssandra migration
----------------------------------
- A helpful tutorial for cassandra statefulset to k8ssandra migration.
- The Migration has no downtime


Some Cassandra Adminstrators are motivated to migrate from a statefulset cassandra to k8ssandra. Here is a way to step by step do the migration.

1) backup your statefulset cluster. ( see the backup-statefulset folder)
2) follow below steps mentioned below:

- cassandra sts cluster : run `nodetool status` chech all status.
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
- cassandra sts cluster : Alter these keyspaces to use NetworkTopologyStrategy system_auth,system_distributed,system_traces and other non-system/user keyspaces. ensure the DC name as seen in the nodetool status output.
```
cqlsh> ALTER KEYSPACE <keyspace_name> WITH replication = {'class': 'NetworkTopologyStrategy', 'datacenter1': 3};
```
- We need to make a new k8ssandra cluster for the cass-operator with the below manifest.
- It is optional to keep the same cassandra cluster name as source cluster as the cassandra clients using the new cluster will check for some cluster variables. ( please double check). We have used the same cluster name as source here.
```
# values.yaml
cassandra:
  version: "4.0.0"
  # version "3.11.12"
  clusterName: "cluster"
  allowMultipleNodesPerWorker: false
  additionalSeeds:
  # it is the cassandra-0 node IP from the cassandra statefulset (source cluster)
  - 172.31.4.217
  heap:
   size: 31g
  gc:
    g1:
      enabled: true
      setUpdatingPauseTimePercent: 5
      maxGcPauseMillis: 300
  resources:
    requests:
      memory: "59Gi"
      cpu: "7000m"
    limits:
      memory: "60Gi"
  datacenters:
  - name: k8s-1
    size: 3
    racks:
    - name: r1
      affinityLabels:
        topology.kubernetes.io/zone: europe-west-1a
    - name: r2
      affinityLabels:
        topology.kubernetes.io/zone: europe-west-1b
    - name: r3
      affinityLabels:
        topology.kubernetes.io/zone: europe-west-1c
  ingress:
    enabled: false
  cassandraLibDirVolume:
    storageClass: standard-rwo
    size: 100Gi
stargate:
  enabled: false
# add medusa config later when migration is a success.
medusa:
  enabled: false
reaper-operator:
  enabled: false
kube-prometheus-stack:
  enabled: false
reaper:
  enabled: false
```
- make the new k8ssandra cluster
```
helm install k8ssandra charts/k8ssandra -n cassandra --create-namespace -f values.yaml
```
- do a `nodetool status` after 10 mins in the source cluster cassandra node. Warch the owns( its 0%)
```
Datacenter: k8s-1
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.0.3.10      78.16 KiB  16      0.0%              c63b9b16-24fe-4232-b146-b7c2f450fcc6  europe-west-1a
UN  10.0.2.66      69.14 KiB  16      0.0%              b1409a2e-cba1-482f-9ea6-c895bf296cd9  europe-west-1b
UN  10.0.1.77      69.13 KiB  16      0.0%              78c53702-7a47-4629-a7bd-db41b1705bb8  europe-west-1c
Datacenter: datacenter1
=====================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens  Owns (effective)  Host ID                               Rack
UN  172.31.4.217   10.2 GiB   16      100.0%            9a9b5e8f-c0c2-404d-95e1-372880e02c43  europe-west-1a
UN  172.31.38.15   10.2 GiB   16      100.0%            1e6a9077-bb47-4584-83d5-8bed63512fd8  europe-west-1b
UN  172.31.22.153  10.2 GiB   16      100.0%            d6488a81-be1c-4b07-9145-2aa32675282a  europe-west-1c
```
- Alter the keyspaces in the new k8ssandra cluster.

system_auth,system_distributed,system_traces and other non-system/user keyspaces. 
```
cqlsh> ALTER KEYSPACE <keyspace_name> WITH replication = {'class': 'NetworkTopologyStrategy', 'datacenter1': '3', 'k8s-1': '3'};
```
Run rebuild
```
   # kubectl exec -it pod/k8s-1-r1-sts-0 -c cassandra -n k8ssandra -- nodetool rebuild k8s-1
   # kubectl exec -it pod/k8s-1-r1-sts-1 -c cassandra -n k8ssandra -- nodetool rebuild k8s-1
   # kubectl exec -it pod/k8s-1-r1-sts-2 -c cassandra -n k8ssandra -- nodetool rebuild k8s-1
```
- final output. Watch the owns ( 100%)
```
Datacenter: k8s-1
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens  Owns (effective)  Host ID                               Rack
UN  10.0.3.10      78.16 KiB  16      0.0%              c63b9b16-24fe-4232-b146-b7c2f450fcc6  europe-west-1a
UN  10.0.2.66      69.14 KiB  16      0.0%              b1409a2e-cba1-482f-9ea6-c895bf296cd9  europe-west-1b
UN  10.0.1.77      69.13 KiB  16      0.0%              78c53702-7a47-4629-a7bd-db41b1705bb8  europe-west-1c
Datacenter: datacenter1
=====================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens  Owns (effective)  Host ID                               Rack
UN  172.31.4.217   10.32 GiB  16      100.0%            9a9b5e8f-c0c2-404d-95e1-372880e02c43  europe-west-1a
UN  172.31.38.15   10.32 GiB  16      100.0%            1e6a9077-bb47-4584-83d5-8bed63512fd8  europe-west-1b
UN  172.31.22.153  10.32 GiB  16      100.0%            d6488a81-be1c-4b07-9145-2aa32675282a  europe-west-1c
```
- check the data is present in the new cluster.
- decommission the old cluster. These are commands if the cluster is in the default namespace
```
   # kubectl exec -it pod/cassandra-0 -c cassandra -n default -- nodetool decommission
   # kubectl exec -it pod/cassandra-1 -c cassandra -n default -- nodetool decommission
   # kubectl exec -it pod/cassandra-2 -c cassandra -n default -- nodetool decommission
```
