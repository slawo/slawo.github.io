# Kubernetes kanidm replicated deployment

For a quick deployment of kanidm a stateful set can be used:

 - In order to store the socket, an empty volume is provided.
 - The configuration is generated from a replication secret which contains the keys of each of the pods and is stored in an empty volume as well.
 - the replication secret is generated using a shell script (this needs to be improved to automate both the generation of the secret and the renewal of the certs)

Prior to aplying the config run `kubectl -nidm create secret generic kanidm-replication`.

```
---
apiVersion: v1
kind: Service
metadata:
  namespace: idm
  name: kanidm-replica
spec:
  clusterIP: None
  selector:
    app: kanidm
  ports:
    - name: https
      protocol: TCP
      port: 8443
      targetPort: 8443
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  # nodes would be named as kanidm-0, kanidm-1, kanidm-2
  name: kanidm
  namespace: idm
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kanidm
  serviceName: kanidm-replica
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: Parallel
  template:
    metadata:
      labels:
        app: kanidm
      annotations:
        prometheus.io/port: "443"
        prometheus.io/scrape: "true"
    spec:
      restartPolicy: Always
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                      - kanidm
              topologyKey: "kubernetes.io/hostname"
      initContainers:
      - name: volume-permissions-db
        image: busybox
        command: ["sh", "-c", "chmod -R 750 /db"]
        volumeMounts:
          - name: kanidm-db
            mountPath: /db
      - name: kanidm-chown-db
        image: kanidm/server
        command: ["sh", "-c", "chown -R 1000:1000 /db"]
        volumeMounts:
          - name: kanidm-db
            mountPath: /db
      - name: kanidm-conf
        image: busybox
        env:
        - name: POD_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['apps.kubernetes.io/pod-index']
        - name: NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        command:
          - sh
          - -x
          - -c
          - |
            echo 'bindaddress = "0.0.0.0:443"' > /config/server.toml
            echo 'ldapbindaddress = "0.0.0.0:636"' >> /config/server.toml
            echo 'trust_x_forward_for = false' >> /config/server.toml
            echo 'db_path = "/db/kanidm.db"' >> /config/server.toml
            echo 'db_fs_type = "other"' >> /config/server.toml
            echo '#db_arc_size = 2048' >> /config/server.toml
            echo 'tls_chain = "/certs/tls.crt"' >> /config/server.toml
            echo 'tls_key = "/certs/tls.key"' >> /config/server.toml
            echo 'log_level = "info"' >> /config/server.toml
            echo 'domain = "idm.example.com"' >> /config/server.toml
            echo 'origin = "https://idm.example.com"' >> /config/server.toml
            echo 'role = "WriteReplica"' >> /config/server.toml
            echo '[online_backup]' >> /config/server.toml
            echo 'path = "/backups/"' >> /config/server.toml
            echo 'schedule = "0 22 * * *"' >> /config/server.toml
            echo 'versions = 7' >> /config/server.toml
            echo 'i_acknowledge_that_replication_is_in_development = true' >> /config/server.toml
            echo '[replication]' >> /config/server.toml
            echo "origin = \"repl://$NAME.kanidm-replica.$NAMESPACE.svc.cluster.local:8444\"" >> /config/server.toml
            echo 'bindaddress = "0.0.0.0:8444"' >> /config/server.toml
            for path in /replication_certs/*; do
                f=$(basename "$path")
                s=${f%.crt}
                if [ "$s" != "kanidm-$POD_INDEX" ] && [  "*" != "$s" ]; then
                    cert=$(cat ${path})
                    if [ "$cert" != "" ]; then
                        echo "[replication.\"repl://$s.kanidm-replica.$NAMESPACE.svc.cluster.local:8444\"]" >> /config/server.toml
                        echo 'type = "mutual-pull"' >> /config/server.toml
                        echo "partner_cert = \"$cert\"" >> /config/server.toml
                        if [ "$POD_INDEX" != "0" ]; then
                            echo 'automatic_refresh = true' >> /config/server.toml
                        fi
                    fi
                fi
            done
        volumeMounts:
          - name: config-volume
            mountPath: /config
          - name: replication-certs
            mountPath: /replication_certs
            readOnly: true
      - name: kanidm-conf-show
        image: busybox
        command:
          - sh
          - -x
          - -c
          - cat /config/server.toml
        volumeMounts:
          - name: config-volume
            mountPath: /config
      containers:
      - name: kanidm
        image: kanidm/server
        imagePullPolicy: IfNotPresent
        securityContext:
          runAsUser: 1000
          runAsGroup: 1000
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
        args: [ "kanidmd", "server", "-c", "/config/server.toml" ]
        env:
          - name: KANIDM_ADMIN_BIND_PATH
            value: "/var/run/kanidmd.sock"
        ports:
          - containerPort: 443
            name: https
          - containerPort: 636
            name: ldaps
          - containerPort: 8443
            name: replica
        volumeMounts:
          - name: backups
            mountPath: /backups
          - name: certs
            mountPath: /certs
            readOnly: true
          - name: config-volume
            mountPath: /config
            readOnly: true
          - name: kanidm-db
            mountPath: /db
          - name: socket-volume
            mountPath: /var/run
      # Run as a non-privileged user
      securityContext:
        fsGroup: 1000
      volumes:
        - name: backups
          nfs:
            server: 192.168.1.1 # IP to our NFS server
            path: /mnt/tank/kbackups # The exported directory THIS IS UNSAFE
        - name: certs
          secret:
            secretName: auth-tls
        - name: config-volume
          emptyDir: {}
        - name: replication-certs
          secret:
            secretName: kanidm-replication
        - name: socket-volume
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: kanidm-db
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
```

The script to generate the secret:
```
REPLICAS=3

for i in $(seq 1 $REPLICAS); do
    CRS=$(kubectl -nidm get sts kanidm -o=jsonpath='{.status.replicas}')
    r=$i
    if [ $CRS -gt $i ]
    then
        r=$CRS
    fi
    let idx=i-1
    kubectl -nidm scale statefulsets kanidm --replicas=$i
    echo "waiting for $i of $CRS pod kanidm-$idx to be ready..."
    kubectl -nidm wait --for=condition=Ready pod/kanidm-$idx --timeout=120s
    CERT=""
    n=0
    while [ -z "$CERT" ]; do
        sleep $((n++))
        CERT=$(kubectl -nidm exec -i -t pods/kanidm-$idx -ckanidm --  kanidmd show-replication-certificate -c /config/server.toml | sed -n '/certificate: "/p' | awk -F'"' '{print $2}')
        echo "replication certificate for pod kanidm-$idx is $CERT"
    done
    kubectl -nidm get secret kanidm-replication -o json | jq --arg crt "$(echo "$CERT" | base64 -w 0)" ".data[\"kanidm-$idx.crt\"]=\$crt" | kubectl -nidm apply -f -
done
```
