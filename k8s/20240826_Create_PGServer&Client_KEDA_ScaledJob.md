# Create PG Server, KEDA ScaledJob, PG Client

```yaml

## Specify Custom envs
TEST_NS=dev
TEST_NAME=postgres


## Create PG Server
kubectl apply -n $TEST_NS -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: $TEST_NAME-config
  labels:
    app: $TEST_NAME
data:
  POSTGRES_DB: postgresdb

---
apiVersion: v1
kind: Secret
metadata:
  name: $TEST_NAME-secret
stringData:
  POSTGRES_USER: postgres
  PGPASSWORD: password
  
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: $TEST_NAME-pv-volume
  labels:
    type: local
    app: $TEST_NAME
spec:
  storageClassName: manual
  capacity:
    storage: 10Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/data"

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: $TEST_NAME-pv-claim
  labels:
    app: $TEST_NAME
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $TEST_NAME
spec:
  replicas: 1
  selector:
    matchLabels:
      app: $TEST_NAME
  template:
    metadata:
      labels:
        app: $TEST_NAME
    spec:
      containers:
        - name: $TEST_NAME-container
          image: postgres:13
          imagePullPolicy: "IfNotPresent"
          lifecycle:
            postStart:
              exec:
                command: ["/bin/sh","-c","sleep 20 && PGPASSWORD=\$PGPASSWORD psql -w -d \$POSTGRES_DB -U \$POSTGRES_USER -c 'CREATE TABLE IF NOT EXISTS jobs (id SERIAL PRIMARY KEY, status VARCHAR(50));'"]
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
               configMapKeyRef:
                  name: $TEST_NAME-config
                  key: POSTGRES_DB
          envFrom:
          - secretRef:
              name: $TEST_NAME-secret
          
          resources:
            requests:
              cpu: 200m
              memory: 200Mi
            limits:
              cpu: 200m
              memory: 200Mi
          
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb

      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: $TEST_NAME-pv-claim

---
apiVersion: v1
kind: Service
metadata:
  name: $TEST_NAME
spec:
  type: ClusterIP
  ports:
    - port: 5432
  selector:
    app: $TEST_NAME
EOF

kubectl get pod -n $TEST_NS

POD_NAME=$(kubectl get pods -n $TEST_NS -l "app=$TEST_NAME" -o jsonpath="{.items[0].metadata.name}")
kubectl logs -n $TEST_NS $POD_NAME

kubectl describe pod -n $TEST_NS $POD_NAME


## Optional: Delete PG Server
kubectl delete -n $TEST_NS service $TEST_NAME
kubectl delete -n $TEST_NS deployment $TEST_NAME
kubectl delete -n $TEST_NS cm $TEST_NAME-config

kubectl delete -n $TEST_NS secret $TEST_NAME-secret

kubectl delete -n $TEST_NS pvc $TEST_NAME-pv-claim
kubectl delete -n $TEST_NS pv  $TEST_NAME-pv-volume


## Create KEDA ScaledJob
kubectl apply -n dev-casino -f - <<EOF
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: $TEST_NAME-keda-auth
spec:
  secretTargetRef:
  - parameter: password
    name: $TEST_NAME-secret
    key: PGPASSWORD
  - parameter: userName
    name: $TEST_NAME-secret
    key: POSTGRES_USER
  env:
  - parameter: userName
    name: userName

  configMapTargetRef:
  - parameter: dbName
    name: $TEST_NAME-config
    key: POSTGRES_DB
  env:
  - parameter: dbName
    name: dbName
---
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: $TEST_NAME-scaledjob
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    backoffLimit: 4
    template:
      spec:
        containers:
          - name: $TEST_NAME-worker
            image: busybox
            command: ["sh", "-c", "echo BEGIN Processing job $(date); sleep 10; echo END Processing job $(date)"]

            ports:
              - containerPort: 80
                protocol: TCP
            resources:
              limits:
                cpu: 10m
                memory: 10Mi
              requests:
                cpu: 10m
                memory: 10Mi

  pollingInterval: 30  # Optional. Default: 30 seconds
  successfulJobsHistoryLimit: 5  # Optional. Default: 100
  failedJobsHistoryLimit: 5  # Optional. Default: 100
  
  triggers:
    - type: postgresql
      metadata:
        #userName: postgres
        host: $TEST_NAME.$TEST_NS.svc.cluster.local
        port: "5432"
        #dbName: postgresdb
        sslmode: disable #require
        query: "SELECT COUNT(*) FROM public.jobs WHERE status = 'pending'"
        targetQueryValue: "1"
        activationQueryValue: "2" # Optional
      authenticationRef:
        name: $TEST_NAME-keda-auth
EOF

kubectl get scaledjob -n $TEST_NS $TEST_NAME-scaledjob

kubectl describe scaledjob -n $TEST_NS $TEST_NAME-scaledjob

kubectl get jobs -n $TEST_NS
kubectl get pods -n $TEST_NS


## Optional: Delete KEDA ScaledJob
kubectl delete scaledjob -n dev-casino $TEST_NAME-scaledjob
kubectl delete TriggerAuthentication -n dev-casino $TEST_NAME-keda-auth


## Create PG Client
kubectl apply -n $TEST_NS -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: $TEST_NAME-pg-client
spec:
  containers:
  - name: pg-client
    image: bitnami/postgresql

    command: ["sleep"]
    args: ["infinity"]

    env:
    - name: PG_HOST
      value: $TEST_NAME.$TEST_NS.svc.cluster.local
    - name: PG_PORT
      value: "5432"
    - name: POSTGRES_DB
      valueFrom:
       configMapKeyRef:
          name: $TEST_NAME-config
          key: POSTGRES_DB

    envFrom:
    - secretRef:
        name: $TEST_NAME-secret

    resources:
      requests:
        cpu: 50m
        memory: 50Mi
      limits:
        cpu: 50m
        memory: 50Mi
EOF

kubectl exec -it $TEST_NAME-pg-client -n $TEST_NS sh
psql --set=sslmode=require --host $PG_HOST --port $PG_PORT -U $POSTGRES_USER -d $POSTGRES_DB


INSERT INTO jobs (status) VALUES ('pending');
SELECT * FROM jobs;
SELECT COUNT(*) FROM jobs WHERE status = 'pending';

DELETE FROM jobs WHERE status = 'pending';


## Optional: Delete PG Client
kubectl delete pod -n $TEST_NS $TEST_NAME-pg-client



## Check Jobs ???


POD_NAME=$(kubectl get pods -n $TEST_NS -l "app=$TEST_NAME" -o jsonpath="{.items[0].metadata.name}")
echo $POD_NAME
kubectl exec -it $POD_NAME -n $TEST_NS sh
psql -w -d $POSTGRES_DB -U $POSTGRES_USER -c 'SELECT * FROM jobs;'
psql -w -d $POSTGRES_DB -U $POSTGRES_USER -c "INSERT INTO jobs (status) VALUES ('pending');"


psql -w -d $POSTGRES_DB -U $POSTGRES_USER -c "SELECT COUNT(*) FROM public.jobs WHERE status = 'pending';"




delete from jobs WHERE id =1;

```
























