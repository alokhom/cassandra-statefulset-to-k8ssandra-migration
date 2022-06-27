Backup statefulset cassandra.
-------------------------------

1) chose the bucket where you want to backup your cassandra statefulset cluster using medusa. Make it from the options supported.
   - https://github.com/thelastpickle/cassandra-medusa/tree/master/docs

2) In this example we are using gcs setup. The idea is to create a credentials.json file using the gcloud cli (console) for the bucket you have created. Make the file medusa.ini as follows. 
   - We will copy the medusa.ini to the statefulset cluster's medusa's container using configMap volumeMount.
   - We will copy this medusa_gcp_key.json to the statefulset cluster's  using a secrets envFrom into the medusa container and have it over /etc/medusa/medusa_gcp_key.json path.
     ```
     [storage]
     storage_provider = google_storage
     bucket_name = gcs_bucket_name
     key_file = /etc/medusa/medusa_gcp_key.json
     ```

3) Insert a medusa container "block" into the statefulset cluster yaml. This will not impact your cluster data or cluster availability but you need to insert it carefully along with valid outcomes of step-2 above. The pods may go in for a rolling restart as it is a sts. You can check the latest medusa container image tag here https://hub.docker.com/r/k8ssandra/medusa/tags and also refer the repo https://github.com/thelastpickle/cassandra-medusa


     ```
        - name: medusa
          image: docker.io/k8ssandra/medusa:0.12.2
          ports:
            - containerPort: 50051
              protocol: TCP
          env:
            - name: MEDUSA_MODE
              value: GRPC
          resources: {}
          volumeMounts:
            - name: server-config
              mountPath: /etc/cassandra
            - name: cassandra-medusa
              mountPath: /etc/medusa
            - name: cassandra-data
              mountPath: /var/lib/cassandra
            - name: google-storage-s3-json
              mountPath: /etc/medusa-secrets
          livenessProbe:
            exec:
              command:
                - /bin/grpc_health_probe
                - '-addr=:50051'
            initialDelaySeconds: 10
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            exec:
              command:
                - /bin/grpc_health_probe
                - '-addr=:50051'
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
          securityContext: {}
     ```

4) One of the option for medusa to work is to use a jolokia based driver to the jvm. Do the following in the cassandra sts script. the jar should be available in the cassandra container in the path /usr/share/java path.

     - Copy the initContainer to the sts cluster
     ```
      initContainers:
        - name: install-jolokia-jvm-agent
          image: busybox
          command:
            - sh
            - '-c'
            - >-
              wget -O /usr/share/java/jolokia-jvm-1.6.2-agent.jar
              http://search.maven.org/remotecontent?filepath=org/jolokia/jolokia-jvm/1.6.2/jolokia-jvm-1.6.2-agent.jar
          resources: {}
          volumeMounts:
            - name: jolokia-share
              mountPath: /usr/share/java
     ```
     - Add the volume to the sts cluster manifest
     ```
      volumes:
        - name: jolokia-share
          emptyDir: {}
        - name: server-config
          emptyDir: {}
        - name: cassandra-medusa
          configMap:
            name: scripts
            items:
              - key: medusa.ini
                path: medusa.ini
        - name: google-storage-s3-json
          secret:
            secretName: google-storage-s3-json
            defaultMode: 420
        - name: medusa-scripts
          configMap: 
            defaultMode: 0755
            name: scripts
            items:
              - key: get_cassandra_node_names.sh
                path: get_cassandra_node_names.sh
     ```
     - Add volumeMounts to initContainers - install-jolokia-jvm-agent & container cassandra of sts so that the jar file is shared from the initcontainer to the cassandra container.
     ```
          volumeMounts:
            - name: jolokia-share
              mountPath: /usr/share/java
     ```
     - Add volumeMounts to container cassandra
     ```
          volumeMounts:
            - name: medusa-scripts
              mountPath: /scripts/medusa-scripts/
     ```
     - Add the environment variables to the main sts cluster cassandra container. Ensure the value for variable CASSANDRA_DC. Refer [link]([https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/cassandra/cassandra-statefulset.yaml)
     ```
                  - name: CASSANDRA_DC
                    value: datacenter1
                  - name: CASSANDRA_ENDPOINT_SNITCH
                    value: GossipingPropertyFileSnitch
                  - name: JVM_EXTRA_OPTS
                    value: -javaagent:/usr/share/java/jolokia-jvm-1.6.2-agent.jar=port=8778,host=localhost
     ```
5) Add the configMap to the namespace where the statefulset manifests resides.
     - In cassandra_backup.yaml, ensure cassandra datacenter name is correctly mapped in the value for cassandraDatacenter.
     - Ensure the cassandra hostname is replaced. For example it is cassandra-0.cassandra.default.svc.cluster.local for cassandra-0 node in default namespace. 
     - Important: Follow the instructions in the configMap code insert.py.
     - The Cassandra Administrator is requested to add the right jmx password as follows in the file jmxremote.password.
     ```
       jmx_user jmx123
     ```
     - The Cassandra Administrator is requested to add the right jmx user as follows in the file jmxremote.access.
     ```
       jmx_user readwrite
     ```
     - Ensure the correct nodetool username as jmx_user and configure it in the medusa.ini 
     ```
       nodetool_username = <jmx_user>
     ```

     ```
     kind: ConfigMap
     apiVersion: v1
     metadata:
       name: scripts
       # namespace: cassandra
     data:
       configure_k8s.sh: |
         #!/bin/sh
         curl -sLO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && chmod +x kubectl && mv ./kubectl /usr/bin/kubectl
         echo "kubectl installed"

       cassandra_backup.yaml: |
         apiVersion: cassandra.k8ssandra.io/v1alpha1
         kind: CassandraBackup
         metadata:
           name: medusa-daily-timestamp
           namespace: cassandra
         spec:
           backupType: "differential"
           name: medusa-daily-timestamp
           cassandraDatacenter: dc1

       get_cassandra_node_names.sh: |
         #!/bin/sh
         apt-get update > /dev/null
         apt-get install sudo -y > /dev/null
         apt-get install dnsutils -qq >/tmp/dns.out;
         if [ -f /tmp/cassandrahostlist ];then rm /tmp/cassandrahostlist; fi
         cat /tmp/cassandraIPlist | while read line;do dig -x $line +short | awk -F"." '{print $1}' >> /tmp/cassandrahostlist;done
         sleep 1;

       ascertain_cassandra_nodes.sh: |
         #!/bin/sh 
         # change namesapce to default if it is needed. Check which namespace the cassandra sts pods are running in the source cluster.
         kubectl exec -it cassandra-0 -n cassandra -c cassandra -- bash -c "[[ -f /secrets/jmxremote.password ]] && nodetool -h ::FFFF:127.0.0.1 -u jmx_user -pwf /secrets/jmxremote.password status | grep UN | cut -d ' ' -f3 > /tmp/cassandraIPlist";sleep 1;

         kubectl exec -it cassandra-0 -n cassandra -c cassandra -- bash -c "./scripts/medusa-scripts/get_cassandra_node_names.sh";sleep 1;
         kubectl exec -it cassandra-0 -n cassandra -c cassandra -- bash -c "cat /tmp/cassandrahostlist" | tee /tmp/hostlist;
         echo "nodes acertained";sleep 5;

       copy_yaml.sh: |
         #!/bin/sh

         # the idea was to copy the cassandra yaml from cassandra node to medusa container in cassandra nodes.  for example a 3 node cassandra. 

         # change namesapce to default if it is needed. the cassandra node should be available  in the namespace. 

         for node in $(cat /tmp/hostlist)
         do
           nodename="$(echo $node| sed 's/\r$//')"
           sleep 2
           kubectl exec pod/$nodename -n cassandra -c cassandra -- tar cf - /etc/cassandra/cassandra.yaml | kubectl exec -i pod/$nodename -n cassandra -c medusa -- bash -c 'tar xvf - -C /tmp && if [ -f /etc/cassandra/cassandra.yaml  ];then rm -f /etc/cassandra/cassandra.yaml; fi; cp /tmp/etc/cassandra/cassandra.yaml /etc/cassandra && ls /etc/cassandra && sleep 2';sleep 5;
         done


       backup_with_medusa.sh: |
         #!/bin/sh
         cp insert.py client_candidate.py
         sleep 1
         dateTime=backup-"$(date +"%m-%d-%Y-%H-%M-%S")"
         for node in $(cat /tmp/hostlist)
         do
           nodename="$(echo $node| sed 's/\r$//').cassandra"
           sleep 5
           echo "............ starting backup of $nodename ...."
           sleep 5
           cat client_candidate.py | sed -e s/localhost/"$nodename"/g  > client.py
           sleep 5
           python ./client.py "$dateTime"
           sleep 5
           echo "............ backup of $nodename complete...."
           sleep 5
         done
         sleep 20;

       insert.py: |
         import time
         from datetime import datetime
         import medusa_pb2
         import medusa_pb2_grpc
         backupName = sys.argv[1]

         <<<< copy paste and align the entire code replacing this line "as a code block" from the link https://github.com/thelastpickle/cassandra-medusa/blob/master/medusa/service/grpc/client.py when you conduct the test and also follow the License restrictions.  >>>>

         if __name__ == '__main__':
             logging.basicConfig()
             client_stub = Client('localhost:50051')
             print("-------------- health_check --------------")
             client_stub.health_check()
             print("-------------- get_backups --------------")
             client_stub.get_backups()
             print("-------------- backing up : ",backupName)
             client_stub.backup(backupName,"full")
             print("-------------- back up complete : ",backupName)


       medusa.ini: |
         [cassandra]
         stop_cmd = /opt/cassandra/bin/cassandra stop
         start_cmd = /opt/cassandra/bin/cassandra start
         config_file = /etc/cassandra/cassandra.yaml
         cql_username = cassandra
         cql_password = cassandra
         nodetool_username = jmx_user
         nodetool_password_file_path = /secrets/jmxremote.password
         ;nodetool_host = cassandra-0.cassandra.default.svc.cluster.local
         nodetool_port = 7199
         nodetool_flags = "-h ::FFFF:127.0.0.1"
         sstableloader_bin = /opt/cassandra/bin/sstableloader
         nodetool_ssl = false
         check_running = nodetool -u jmx_user -pwf /secrets/jmxremote.password  version
         resolve_ip_addresses = True
         use_sudo = False

         [storage]
         storage_provider = google_storage
         region = europe-west1
         bucket_name = cassandrastsbackup
         key_file = /etc/medusa-secrets/medusa_gcp_key.json
         prefix = .cassandra
         max_backup_age = 5
         max_backup_count = 0
         transfer_max_bandwidth = 50MB/s
         concurrent_transfers = 1
         multi_part_upload_threshold = 104857600
         backup_grace_period_in_days = 10
         use_sudo_for_restore = False

         [monitoring]
         ;monitoring_provider = <Provider used for sending metrics. Currently either of "ffwd" or "local">


         [ssh]
         username = root
         key_file = /tmp/hostCerts/ssh_host_ed25519_key
         cert_file = /tmp/hostCerts/ssh_host_ed25519_key-cert.pub


         [checks]
         ;expected_rows = <Number of rows expected to be returned when the query runs. Not checked if not specified.>


         [logging]
         ; Controls file logging, disabled by default.
         enabled = 1
         file = medusa.log
         level = DEBUG
         ; Control the log output format
         format = [%(asctime)s] %(levelname)s: %(message)s
         ; Size over which log file will rotate
         maxBytes = 20000000
         ; How many log files to keep
         backupCount = 50


         [grpc]
         ; Set to true when running in grpc server mode.
         ; Allows to propagate the exceptions instead of exiting the program.
         enabled = 1

         [kubernetes]
         ; The following settings are only intended to be configured if Medusa is running in containers, preferably in Kubernetes.
         enabled = 1
         ;cassandra_url = <URL of the management API snapshot endpoint. For example: http://127.0.0.1:8080/api/v0/ops/node/snapshots>
         cassandra_url = http://127.0.0.1:8778/jolokia/
         ; Enables the use of the management API to create snapshots. Falls back to using Jolokia if not enabled.
         use_mgmt_api = 0
     ```


4. Add the cronJob for the backup. Ensure it in the right namespace where the configmap is and statefulset.
```
apiVersion: batch/v1
kind: CronJob
metadata: 
  name: medusa-grpc-backup
spec: 
  schedule: "35 0 * * 0-6"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate: 
    spec: 
      template: 
        metadata: 
          name: medusa-grpc-backup
        spec:
          initContainers:
            - name: medusa-copy
              image: "alpine:3.15"
              resources:
                requests:
                  cpu: 20m
                  memory: 200Mi
              imagePullPolicy: IfNotPresent
              command: ["/bin/sh", "-c"]
              args:
                - cp ./data/scripts/* ./scripts;
                  apk add --update --no-cache --quiet curl coreutils; apk --quiet upgrade;
                  ./scripts/configure_k8s.sh;
                  sleep 2;
                  sh ./scripts/ascertain_cassandra_nodes.sh;
                  sleep 2;
                  sh ./scripts/copy_yaml.sh;
                  sleep 2;
                  echo "yamls copied from cass containers to medusa containers...";
                  sleep 10;
              volumeMounts:
                - name: cache-volume
                  mountPath: "/scripts"
                - name: data-scripts
                  mountPath: /data/scripts
          containers:
            - name: medusa-grpc-backup
              image: "python:3.8-slim-buster"
              resources:
                requests:
                  cpu: 20m
                  memory: 200Mi
              imagePullPolicy: IfNotPresent
              command: ["/bin/sh", "-c"]
              args:
                - apt update -qq && apt-get install -qq --no-install-recommends git curl > /dev/null;
                  rm -rf /var/lib/apt/lists/*;
                  git clone https://github.com/thelastpickle/cassandra-medusa.git;
                  cd cassandra-medusa;
                  pip install -r requirements-grpc.txt > /dev/null;
                  cd medusa/service/grpc;
                  python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. medusa.proto;
                  cp /data/scripts/client.py .;
                  sleep 2;
                  sh /data/scripts/configure_k8s.sh;
                  sleep 2;
                  sh /data/scripts/ascertain_cassandra_nodes.sh;
                  sleep 2;
                  sh /data/scripts/backup_with_medusa.sh;
                  sleep 2;
              volumeMounts:
                - name: data-scripts
                  mountPath: /data/scripts
          volumes:
            - name: data-scripts
              configMap: 
                defaultMode: 0755
                name: scripts
```


5. Checklist.
     - Check the manifests are in the same namespace and correct the manifests if its not.
     - Run the Cronjob.
     - Ensure that the cronjob is able to work successfully given the above configuration 1-4.
     - Check backups inside the bucket post running the ConfigMap. 
     - cronjob's job and its pod logs should show appropriate logs.
     - backups should be steadily watched and also CronJob run-on-demand so you get some valid full backups. 
     - Statefulset cassandra restore will be handled in the last if at all there is a failure to migrate.
