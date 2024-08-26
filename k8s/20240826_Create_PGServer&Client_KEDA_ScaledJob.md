# Create PG Server, KEDA ScaledJob, PG Client


#### Create PG Server
```yaml

## Specify Custom envs
TEST_NS=dev
TEST_NAME=test-postgres

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
  labels:
    app: $TEST_NAME
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
  labels:
    app: $TEST_NAME
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
  labels:
    app: $TEST_NAME
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
```


#### Create PG Client
```yaml
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

SELECT * FROM jobs;
INSERT INTO jobs (status) VALUES ('pending');

SELECT COUNT(*) FROM jobs WHERE status = 'pending';

DELETE FROM jobs WHERE status = 'pending';


## Optional: Delete PG Client
kubectl delete pod -n $TEST_NS $TEST_NAME-pg-client
```


##### Create KEDA ScaledJob
```yaml
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
    parallelism: 1   # [max number of desired pods](https://kubernetes.io/docs/concepts/workloads/controllers/job/#controlling-parallelism)
    completions: 1   # [desired number of successfully finished pods](https://kubernetes.io/docs/concepts/workloads/controllers/job/#controlling-parallelism)
    activeDeadlineSeconds: 36000  # Duration in seconds relative to the startTime that the job may be active before the system tries to terminate it
    
    backoffLimit: 6  # The number of retries before marking this job failed. Defaults to 6
    template:
      metadata:
        annotations:
          sidecar.istio.io/inject: "false" ## Disable istio, because it cannot work inside the Job
        labels:
          app: $TEST_NAME
      spec:
        restartPolicy: Never
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
        imagePullSecrets:
        - name: ds365reg

  pollingInterval: 30  # Optional. Default: 30 seconds
  successfulJobsHistoryLimit: 5  # Optional. Default: 100 How many completed jobs should be kept.
  failedJobsHistoryLimit: 5  # Optional. Default: 100
  minReplicaCount: 0                          # Optional. Default: 0
  maxReplicaCount: 1                          # Optional. Default: 100
  scalingStrategy:
    strategy: "accurate"                        # Optional. Default: default. Which Scaling Strategy to use. 
    
  triggers:
    - type: postgresql ## https://keda.sh/docs/2.15/scalers/postgresql/
      metadata:
        host: $TEST_NAME.$TEST_NS.svc.cluster.local
        port: "5432"
        sslmode: disable #require
        query: "SELECT COUNT(*) FROM jobs WHERE status = 'pending'"
        targetQueryValue: "1" ## when exceeds the targetQueryValue, KEDA will scale up the job, and when it is below the targetQueryValue, KEDA will scale down.
      authenticationRef:
        name: $TEST_NAME-keda-auth
EOF

kubectl get scaledjob -n $TEST_NS $TEST_NAME-scaledjob
kubectl describe scaledjob -n $TEST_NS $TEST_NAME-scaledjob


## Optional: Delete KEDA ScaledJob
kubectl delete scaledjob -n dev-casino $TEST_NAME-scaledjob
kubectl delete TriggerAuthentication -n dev-casino $TEST_NAME-keda-auth
```


#### Enable Test Job
```yaml
kubectl exec -it $TEST_NAME-pg-client -n $TEST_NS sh
psql --set=sslmode=require --host $PG_HOST --port $PG_PORT -U $POSTGRES_USER -d $POSTGRES_DB

SELECT * FROM jobs;
INSERT INTO jobs (status) VALUES ('pending');

SELECT COUNT(*) FROM jobs WHERE status = 'pending';

DELETE FROM jobs WHERE status = 'pending';


#### Check Jobs
```yaml
kubectl describe scaledjob -n $TEST_NS $TEST_NAME-scaledjob

kubectl get jobs -n $TEST_NS
kubectl get pods -n $TEST_NS
```


#### Disable Test Job
```yaml
### Optional: Run into PG Server 
POD_NAME=$(kubectl get pods -n $TEST_NS -l "app=$TEST_NAME" -o jsonpath="{.items[0].metadata.name}")
echo $POD_NAME
kubectl exec -it $POD_NAME -n $TEST_NS sh
psql -w -d $POSTGRES_DB -U $POSTGRES_USER -c 'SELECT * FROM jobs;'
psql -w -d $POSTGRES_DB -U $POSTGRES_USER -c "DELETE FROM jobs WHERE status = 'pending';"
```


#### Delete all created test objects
```yaml
## Delete KEDA ScaledJob
kubectl delete scaledjob -n dev-casino $TEST_NAME-scaledjob
kubectl delete TriggerAuthentication -n dev-casino $TEST_NAME-keda-auth

## Delete PG Client
kubectl delete pod -n $TEST_NS $TEST_NAME-pg-client

## Delete PG Server
kubectl delete -n $TEST_NS service $TEST_NAME
kubectl delete -n $TEST_NS deployment $TEST_NAME
kubectl delete -n $TEST_NS cm $TEST_NAME-config
kubectl delete -n $TEST_NS secret $TEST_NAME-secret
kubectl delete -n $TEST_NS pvc $TEST_NAME-pv-claim
kubectl delete -n $TEST_NS pv  $TEST_NAME-pv-volume

## Delete related Jobs & PODs
kubectl delete jobs -n $TEST_NS -l app=$TEST_NAME
kubectl delete pods -n $TEST_NS -l app=$TEST_NAME
```
























