

# Restore from medusa backup #

To restore from the backup ensure the backup step 1 is followed and you have backup copy in the s3. The cassandra pods should be labled in their sts yaml.
```
          labels:          
            cassandra: restore
```
After the pods restart, please add the following snippet to the <b>initContainer>/b> of the cassandra sts yaml and replace with the right value for the env variables for example BACKUP_NAME. 
- For the first time there is no RESTORE_KEY value so you can set any value to the variable that will be used again. 
- The medusa-cass-yaml container block below is used to extract the cassandra.yml as a template from a running statefulset cassandra pod. (its the vanilla statefulset cassandra)
- Then the template is set with the cassandra pod IP and ported to the medusa-restore container. 
- As and when you save the sts with these changes, the restore is fired from the s3 store against the BACKUP_NAME. The server-config is a shared folder so the /etc/cassandra/cassandra.yaml is passed from  medusa-cass-yaml container to medusa-restore container.
```
        - name: medusa-cass-yaml
          image: alpine:3.15
          imagePullPolicy: IfNotPresent
          command:
            - /bin/sh
            - '-c'
          args:
            - >-
              cp ./data/scripts/* ./scripts;apk add --update --no-cache --quiet curl coreutils; apk --quiet upgrade;./scripts/configure_k8s.sh;
              sleep 2; kubectl get pod -l cassandra=restore --field-selector=status.phase=Running -n cassandra | awk -F" " '{print $1}' | grep -v NAME | head -1 > /tmp/firstPod;
              sleep 5; kubectl exec pod/"$(cat /tmp/firstPod | sed 's/\r$//')" -c cassandra -n cassandra -- tar cf - /etc/cassandra/cassandra.yaml | tar xvf - -C /tmp && cat /tmp/etc/cassandra/cassandra.yaml > /tmp/cassandra.yaml.template;
              sleep 4; cat /tmp/cassandra.yaml.template | sed "s/10\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/$POD_IP/g" > /etc/cassandra/cassandra.yaml;
              sleep 2; cat /etc/cassandra/cassandra.yaml; sleep 10;
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.podIP
          resources:
            requests:
              cpu: 20m
              memory: 200Mi
          volumeMounts:
            - name: cache-volume
              mountPath: /scripts
            - name: data-scripts
              mountPath: /data/scripts
            - name: server-config
              mountPath: /etc/cassandra

        - name: medusa-restore
          image: docker.io/k8ssandra/medusa:0.11.3
          imagePullPolicy: IfNotPresent
          env:
            - name: MEDUSA_MODE
              value: RESTORE
            - name: RESTORE_KEY
              value: anyrandomkeyFirstTimeOnly                                    <------- update this value and put some arbitrary name if you want to restore. The container saves this valye and compares next restore so change it every time
            - name: BACKUP_NAME
              value: backup-06-03-2022-00-36-16                                   <------- update the value with the candidate you want to restore. ensure the bucket has this folder in all the cass node folders.
          resources: {}
          volumeMounts:
            - name: cassandra-medusa
              mountPath: /etc/medusa
            - name: server-config
              mountPath: /etc/cassandra
            - name: cassandra-data
              mountPath: /var/lib/cassandra
            - name: podinfo
              mountPath: /etc/podinfo
            - name: google-storage-s3-json
              mountPath: /etc/medusa-secrets
            - name: kube-api-access-d9chp
              readOnly: true
              mountPath: /var/run/secrets/kubernetes.io/serviceaccount
```

   
