# Tips How to set up Kubernetes ENV for Local Developement
Current tips help to set up Local Kubernetes Cluster using `Windows10 WSL2` and `k3d` tool


## Install WSL2
N.B.: The steps below were helpful to me, but there are a lot of other [examples](https://docs.microsoft.com/en-us/windows/wsl/install) how to do that and you may want to install that by yourself.

#### - Install Distro `WSL2 Ubuntu`:
```sh
cmd.exe
wsl --install -d Ubuntu
sudo apt update && sudo apt upgrade
```

#### - Enable [docker for WSL2](https://docs.docker.com/desktop/windows/wsl/)
```sh
wsl.exe -l -v
wsl.exe --set-version Ubuntu 2
wsl.exe --set-default-version 2
wsl --set-default Ubuntu

## Configure DockerDesktop: Settings > Resources > WSL Integration - Ubuntu ON
```

#### - Check that your Distro is OK, troubleshooting [here](https://github.com/docker/for-win/issues/6971#issuecomment-636358053) 
`wsl -l -v`
```sh
  NAME                   STATE           VERSION
* Ubuntu                 Running         2
  docker-desktop         Running         2
  docker-desktop-data    Running         2
```


## Configure Your Distro
Open your `Distro` shell and install nessecary tools

#### - Install [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.22.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

#### - Instal [Helm](https://helm.sh/docs/intro/install)
```sh
curl -O https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz
tar -zxvf helm-v3.7.1-linux-amd64.tar.gz
ls linux-amd64
sudo mv linux-amd64/helm /usr/local/bin/helm
```

#### - Install [k3d](https://k3d.io/v5.3.0/#install-current-latest-release) (script that installs `k3s`):
`curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | TAG=v5.3.0 bash`


#### - Create [k3d cluster](https://k3d.io/v5.3.0/usage/commands/k3d_cluster/)  
```sh
k3d cluster list
k3d cluster create test -p "8081:80@loadbalancer"
```

#### - View Cluster PODs
`kubectl get pods -A`
```sh
kube-system   local-path-provisioner-84bb864455-4bgpc   1/1     Running     0          4h41m
kube-system   coredns-96cc4f57d-q2lxd                   1/1     Running     0          4h41m
kube-system   helm-install-traefik-crd--1-vwxvl         0/1     Completed   0          4h41m
kube-system   helm-install-traefik--1-86jjs             0/1     Completed   1          4h41m
kube-system   svclb-traefik-77brf                       2/2     Running     0          4h40m
kube-system   metrics-server-ff9dbcb6c-dm6hb            1/1     Running     0          4h41m
kube-system   traefik-55fdc6d984-59mq7                  1/1     Running     0          4h40m
```

##  *Congrats!!! You have already configured Local K3S Cluster :+1:*
\
\
\
\
\
## Addons
The next steps are optional, use these steps if you want to: 


### - Install cert-manager
```sh
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager -n cert-manager
```


### - Install Echoserver
Current steps install exposed outside `echoserver` microservice with mapped secrets inside.

#### - Define echoserver namespace
```yaml
ECHOSERVER_NS=pf
kubectl create ns $ECHOSERVER_NS
```

#### - Create echoserver secrets
```yaml
kubectl apply -n $ECHOSERVER_NS -f - <<EOF
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: demo-secret1
stringData:
  username: demo-secret1-username
  password: demo-secret1-password
---
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: demo-secret2
stringData:
  username: demo-secret2-username
  password: demo-secret2-password
EOF
```

#### - Create echoserver config
```yaml
kubectl apply -n $ECHOSERVER_NS -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: echoserver
  name: echoserver
data:
  nginx.conf: |
    pcre_jit on;
    events {
        worker_connections  1024;
    }
    env SECRET1_USERNAME;
    env SECRET1_PASSWORD;
    env SECRET2_USERNAME;
    env SECRET2_PASSWORD;
    
    http {
        include       mime.types;
        default_type  application/octet-stream;
        client_body_temp_path /var/run/openresty/nginx-client-body;
        proxy_temp_path       /var/run/openresty/nginx-proxy;
        fastcgi_temp_path     /var/run/openresty/nginx-fastcgi;
        uwsgi_temp_path       /var/run/openresty/nginx-uwsgi;
        scgi_temp_path        /var/run/openresty/nginx-scgi;
        sendfile        on;
        keepalive_timeout  65;
        include /etc/nginx/conf.d/*.conf;
    }

  default.conf: |
    error_log stderr debug;
    server {
     listen 8080;
        location / {
          types { } default_type "text/plain; charset=utf-8";
          set \$response "\n------ Echoserver Response: POD_IP='\$server_addr:\$server_port' ------\n\n--- YOUR Request Details ---\n-ENDPOINT: \$request_method:\$request_uri\n";
            content_by_lua_block {
                ngx.req.read_body()
                local request_body = ngx.req.get_body_data()
                ngx.say(ngx.var.response ..
                "\n-HEADERS: " .. ngx.req.raw_header() ..
                "\n-BODY: " .. tostring(request_body) ..
                "\n\n--- ECHOSERVER ENVs ---"
                .. "\n"
                .. "SECRET1_USERNAME='" .. tostring(os.getenv("SECRET1_USERNAME")) .. "'\n"
                .. "SECRET1_PASSWORD='" .. tostring(os.getenv("SECRET1_PASSWORD")) .. "'\n"
                .. "SECRET2_USERNAME='" .. tostring(os.getenv("SECRET2_USERNAME")) .. "'\n"
                .. "SECRET2_PASSWORD='" .. tostring(os.getenv("SECRET2_PASSWORD")) .. "'\n"                
                .. "\n")
                return ngx.exit(ngx.HTTP_OK)
            }
        }
        location /reload {
            types { } default_type "text/plain; charset=utf-8";
            content_by_lua_block {
              -- reload nginx context
              os.execute("/usr/local/openresty/nginx/sbin/nginx -s reload")
              ngx.say("RELOAD SUCCESS")
              return ngx.exit(ngx.HTTP_OK)
            }
        }
    }
EOF
```

#### - Define echoserver type
```sh
CLUSTER_EXT_URL=localhost
echo $CLUSTER_EXT_URL

WORKLOAD_TYPE=Deployment ## details: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
#WORKLOAD_TYPE=StatefulSet ## details: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
#WORKLOAD_TYPE=DaemonSet    ## details: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

WORKLOAD_SUFFIX=$(echo "$WORKLOAD_TYPE" | tr '[:upper:]' '[:lower:]')
echo "$WORKLOAD_SUFFIX"

SERVICE_PORT=8090
#SERVICE_PORT=8091
#SERVICE_PORT=8092
```

#### - Create and expose echoserver workload
```yaml
kubectl apply -n $ECHOSERVER_NS -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: echoserver-$WORKLOAD_SUFFIX
  labels:
    app: echoserver
spec:
  ports:
  - port: $SERVICE_PORT
    name: http
    targetPort: 8080 ## echoserver nginx.conf port
  selector:
    app: echoserver
    workload: echoserver-$WORKLOAD_SUFFIX

---

apiVersion: apps/v1
kind: $WORKLOAD_TYPE
metadata:
  name: echoserver
  labels:
    app: echoserver
    workload: echoserver-$WORKLOAD_SUFFIX
spec:
  #replicas: 1
  #serviceName: "echoserver-$WORKLOAD_SUFFIX" ## field is required for StatefulSet ONLY!
  selector:
    matchLabels:
      app: echoserver
      workload: echoserver-$WORKLOAD_SUFFIX
  template:
    metadata:
      labels:
        app: echoserver
        workload: echoserver-$WORKLOAD_SUFFIX
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: echoserver
        env:
        - name: SECRET1_USERNAME
          valueFrom:
            secretKeyRef:
              key: username
              name: demo-secret1
        - name: SECRET1_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: demo-secret1

        - name: SECRET2_USERNAME
          valueFrom:
            secretKeyRef:
              key: username
              name: demo-secret2
        - name: SECRET2_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: demo-secret2
        image: openresty/openresty
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080 ## echoserver nginx.conf port
        volumeMounts:
        - name: config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        - name: config
          mountPath: /usr/local/openresty/nginx/conf/nginx.conf
          subPath: nginx.conf
        - name: tmp
          mountPath: /var/run/openresty/
        - name: logs
          mountPath: /usr/local/openresty/nginx/logs/
        resources:
          requests:
            cpu: "100m"
      volumes:
      - name: config
        configMap:
          name: echoserver
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
  name: echoserver-$WORKLOAD_SUFFIX
spec:
  rules:
  - host: $CLUSTER_EXT_URL
    http:
      paths:
      - backend:
          service:      
            name: echoserver-$WORKLOAD_SUFFIX
            port:
              number: $SERVICE_PORT
        path: /echo-$WORKLOAD_SUFFIX
        pathType: Prefix
  tls:
  - hosts:
    - $CLUSTER_EXT_URL
EOF
```

#### - Access echoserver outside the cluster
`curl http://localhost:8081/echo-$WORKLOAD_SUFFIX`

#### - Delete Demo Resources
```sh
kubectl delete ingress -n $ECHOSERVER_NS --all
kubectl delete deployment -n $ECHOSERVER_NS echoserver
kubectl delete daemonset -n $ECHOSERVER_NS echoserver
kubectl delete statefulset -n $ECHOSERVER_NS echoserver
kubectl delete service -n $ECHOSERVER_NS --all
kubectl delete cm -n $ECHOSERVER_NS echoserver
kubectl delete secrets -n $ECHOSERVER_NS demo-secret1 demo-secret2
```

### - Install Postgres Operator
*Theory*:
- [Spilo](https://github.com/zalando/spilo) is a Docker image that provides PostgreSQL HA and Patroni bundled together:
  + [PostgreSQL Streaming Replication](https://hevodata.com/learn/postgresql-streaming-replication/) based on [Replication Slots (WAL files)](https://hevodata.com/learn/postgresql-replication-slots/) 

- [Patroni](https://github.com/zalando/patroni#how-patroni-works) is a template for PostgreSQL HA based on python scripts:
  + [Standby](https://opensource.zalando.com/postgres-operator/docs/user.html#setting-up-a-standby-cluster) cluster is a Patroni feature that first clones a database, and keeps replicating changes in readonly mode
  + [patroni spec](https://buildmedia.readthedocs.org/media/pdf/patroni/latest/patroni.pdf)

- [Zalando](https://github.com/zalando/postgres-operator) is a postgres-operator that manages Patroni HA Replica PODs depend on created [postgresql](https://github.com/zalando/postgres-operator/blob/master/docs/reference/cluster_manifest.md) object (mainfest):
  + Zalando is not well documented and developed rapidly, reading [sources](https://github.com/zalando/postgres-operator/tree/master/pkg) is a must!
  + [PostgreSQL on K8s at Zalando: Two years in production](https://av.tib.eu/media/52142) video with working details and known issues

- [Pgpool-II](https://www.pgpool.net/docs/latest/en/html/intro-whatis.html) manages a pool of PostgreSQL servers to achieve
  + [PgBouncer vs Pgpool-II](https://scalegrid.io/blog/postgresql-connection-pooling-part-4-pgbouncer-vs-pgpool/#:~:text=PgBouncer%20allows%20limiting%20connections%20per,overall%20number%20of%20connections%20only.&text=PgBouncer%20supports%20queuing%20at%20the,i.e.%20PgBouncer%20maintains%20the%20queue)
#### - Install [Zalando](https://github.com/zalando/postgres-operator) postgres-operator
```sh
git clone https://github.com/zalando/postgres-operator.git
cd /mnt/d/Project/github/zalando/postgres-operator/charts/postgres-operator
kubectl create ns pgo
helm install pgo . -n pgo
```


#### - Create ConfigMap with [Custom ENVs for all Cluster PODs](https://github.com/zalando/postgres-operator/blob/master/docs/administrator.md#custom-pod-environment-variables) by default
```yaml


## Optinonal:  Create ConfigMap with default ENVs which will be added into PG Statefulset
kubectl apply -n pgo -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-pod-config
data:
  ## --- Backup Settings ---
  AWS_ENDPOINT: http://storage-minio.s3.svc.cluster.local:9000 #
  AWS_ACCESS_KEY_ID: minio
  AWS_SECRET_ACCESS_KEY: minio123
  AWS_REGION: minio
  AWS_S3_FORCE_PATH_STYLE: "true" # needed for MinIO ONLY

  WAL_S3_BUCKET: foundation-pf
  WALE_S3_BUCKET: foundation-pf
  WAL_BUCKET_SCOPE_PREFIX: ""
  WAL_BUCKET_SCOPE_SUFFIX: ""

  WALG_DISABLE_S3_SSE: "true"

  USE_WALG_BACKUP: "true"
  USE_WALG_RESTORE: "true"

  #WALE_S3_ENDPOINT: http://storage-minio.s3.svc.cluster.local:9000
  #WALE_S3_PREFIX: s3://foundation-pf/spilo/postgres-db-pg-cluster
  #WALG_S3_ENDPOINT: http://storage-minio.s3.svc.cluster.local:9000
  #WALG_S3_PREFIX: s3://foundation-pf/spilo/postgres-db-pg-cluster

  BACKUP_SCHEDULE: '*/3  * * * *' ## Every 3 minutes
  BACKUP_NUM_TO_RETAIN: "5"

  ## --- Clone Settings ---
  ## Clone creds can be specified in the "postgresql" object
  #CLONE_AWS_ENDPOINT: http://storage-minio.s3.svc.cluster.local:9000  
  #CLONE_AWS_ACCESS_KEY_ID: minio
  #CLONE_AWS_SECRET_ACCESS_KEY: minio123
  
  CLONE_AWS_REGION: minio
  CLONE_WAL_S3_BUCKET: "foundation-pf"
  CLONE_WAL_BUCKET_SCOPE_SUFFIX: ""
  CLONE_WAL_BUCKET_SCOPE_PREFIX: ""
  CLONE_AWS_S3_FORCE_PATH_STYLE: "true" # needed for MinIO
  CLONE_METHOD: CLONE_WITH_WALE
  #CLONE_WITH_WALE: "true"  ## Enable cloning for every new cluster by default !!!
  
  ## Other optional clone params
  #CLONE_WALE_ENV_DIR: "/tmp/wal-g"
  #CLONE_USE_WALG_RESTORE: "true"
  #CLONE_SCOPE: postgres-db-pg-cluster  
  ##CLONE_WAL_BUCKET_SCOPE_SUFFIX: "/889918f8-0c89-455d-b0bb-8cf0b799c011"
  ##CLONE_TARGET_TIME: "2025-12-19T12:40:33+00:00"
EOF

## Link ConfigMap inside the OperatorConfiguration
kubectl edit OperatorConfiguration -n pgo pgo-postgres-operator
...
configuration:
  kubernetes:
    ## Do NOT specify namespace if ConfigMap must be taken from the namespace where 'postgresql' is created
    #pod_environment_configmap: postgres-pod-config   
    pod_environment_configmap: pgo/postgres-pod-config
...

## Restart Zalando Operator
kubectl delete pods -n pgo --all
```

#### - View Zalando Operator logs
```bash
## View Operator logs
POD_NS=pgo
POD_LABEL=app.kubernetes.io/name=postgres-operator
POD_NAME=$(kubectl get pods -n $POD_NS -l "$POD_LABEL" -o jsonpath="{.items[0].metadata.name}")
klo -n $POD_NS $POD_NAME
```


#### - Create 'SiteA' `postgresql` cluster 
```yaml
## Optional: cleanup backups using Minio UI: http://localhost:8081

SITEA_NS=sa
kubectl create ns $SITEA_NS
SITEA_NAME=postgres-db-site-a

## Create secrets with pre-defined passwords
cat <<EOF | kubectl apply -f -
apiVersion: v1
type: Opaque
kind: Secret
metadata:
  name: standby.$SITEA_NAME.credentials.postgresql.acid.zalan.do
  namespace: $SITEA_NS
stringData:
  password: standbyPwd
  username: standby
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
type: Opaque
kind: Secret
metadata:
  name: postgres.$SITEA_NAME.credentials.postgresql.acid.zalan.do
  namespace: $SITEA_NS
stringData:
  password: postgresPwd
  username: postgres
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
type: Opaque
kind: Secret
metadata:
  name: conjuruser.$SITEA_NAME.credentials.postgresql.acid.zalan.do
  namespace: $SITEA_NS
stringData:
  password: conjuruserPwd
  username: conjuruser
EOF


## SiteA: Create postgresql : https://github.com/zalando/postgres-operator/blob/v1.8.1/manifests/complete-postgres-manifest.yaml
kubectl apply -n $SITEA_NS -f - <<EOF
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: $SITEA_NAME ## Cluster name prefix must match with "teamId" value !!!
spec:
  teamId: "postgres-db"
  volume:
    size: 128Mi
  numberOfInstances: 2
  users:
    ## Create users, set up roles
    conjuruser:  # database owner
    - superuser
    - createdb
    conjurdb_user: []  # ordinary role for application
  ## Create db & assign owner
  databases:
    conjurdb: conjuruser  # dbname: owner

  postgresql:
    version: "14"
    parameters: ## addons for 'postgresql.conf': https://github.com/postgres/postgres/blob/master/src/backend/utils/misc/postgresql.conf.sample
      log_statement: "all"
      log_replication_commands: "on"

  patroni: ## addons for patroni 'postgres.yml': https://github.com/zalando/patroni/blob/master/postgres0.yml
    synchronous_mode: true
    synchronous_mode_strict: true

  ## Create DB with already created users inside: https://github.com/zalando/postgres-operator/blob/master/docs/user.md#default-nologin-roles
  #preparedDatabases:
  #  foo: {}

  ## Create DB as clone from S3 backup
  #clone:
  ##  #uid: "889918f8-0c89-455d-b0bb-8cf0b799c011"
  #  cluster: $SITEA_NAME
  #  timestamp: "2022-06-15T16:50:00+00:00"
  #  s3_endpoint: http://storage-minio.s3.svc.cluster.local:9000
  #  s3_access_key_id: minio
  #  s3_secret_access_key: minio123
  #  s3_wal_path: "s3://foundation-pf/spilo/$SITEA_NAME/wal"
  
EOF


## SiteA: Check Statefulset ENVs
kubectl get statefulset -n $SITEA_NS $SITEA_NAME -o yaml | grep env -A100
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- env

## SiteA: View Master logs
kubectl logs -f -n $SITEA_NS $SITEA_NAME-0
  #... INFO: no action. I am (postgres-db-original-0) the leader with the lock
  
## SiteA: View Replica logs
kubectl logs -f -n $SITEA_NS $SITEA_NAME-1
  #... INFO: no action. I am a secondary (postgres-db-r1-1) and following a leader (postgres-db-r1-0)
  
## SiteA: View Patroni replicas
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- patronictl list

+ Cluster: postgres-db-site-a (7109447695371919429) +---------+----+-----------+
| Member               | Host        | Role         | State   | TL | Lag in MB |
+----------------------+-------------+--------------+---------+----+-----------+
| postgres-db-site-a-0 | 10.42.0.192 | Leader       | running |  1 |           |
| postgres-db-site-a-1 | 10.42.0.194 | Sync Standby | running |  1 |         0 |
+----------------------+-------------+--------------+---------+----+-----------+

## - Connect as appuser
DB_NAME=conjurdb
DB_USERNAME=conjuruser

## - Connect as superuser
DB_NAME=postgres
DB_USERNAME=postgres

DB_PASSWORD=$(kubectl get secret -n $SITEA_NS "$DB_USERNAME.$SITEA_NAME.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo -e "DB_NAME='$DB_NAME'\nDB_USERNAME='$DB_USERNAME'\nDB_PASSWORD='$DB_PASSWORD'\n"

## SiteA: Create table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " \
CREATE TABLE test ( \
    id bigserial primary key, \
    name varchar(20) NOT NULL, \
    notes text NOT NULL, \
    added timestamp default NOW() \
);"

"

## SiteA: Insert into table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " INSERT INTO test(name, notes) VALUES ('test_name', 'test_notes'); "

## SiteA(Leader): Select from table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

## SiteA(Replica): Select from table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-1 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

```


#### - Create 'SiteB' `postgresql` cluster 
```yaml
## Optional: cleanup backups using Minio UI localhost:8081

SITEB_NS=sb
kubectl create ns $SITEB_NS
SITEB_NAME=postgres-db-site-b

## Create secrets with pre-defined passwords
cat <<EOF | kubectl apply -f -
apiVersion: v1
type: Opaque
kind: Secret
metadata:
  name: standby.$SITEB_NAME.credentials.postgresql.acid.zalan.do
  namespace: $SITEB_NS
stringData:
  password: standbyPwd
  username: standby
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
type: Opaque
kind: Secret
metadata:
  name: postgres.$SITEB_NAME.credentials.postgresql.acid.zalan.do
  namespace: $SITEB_NS
stringData:
  password: postgresPwd
  username: postgres
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
type: Opaque
kind: Secret
metadata:
  name: conjuruser.$SITEB_NAME.credentials.postgresql.acid.zalan.do
  namespace: $SITEB_NS
stringData:
  password: conjuruserPwd
  username: conjuruser
EOF


## Create SiteB as a STANDBY : https://github.com/zalando/postgres-operator/blob/v1.8.1/manifests/standby-manifest.yaml
kubectl apply -n $SITEB_NS -f - <<EOF
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: $SITEB_NAME ## Cluster name prefix must match with "teamId" value !!!
spec:
  teamId: "postgres-db" ## same value as in the site-a
  volume:
    size: 128Mi
  numberOfInstances: 2

  postgresql:
    version: "14"
    parameters: ## addons for 'postgresql.conf': https://github.com/postgres/postgres/blob/master/src/backend/utils/misc/postgresql.conf.sample
      log_statement: "all"
      log_replication_commands: "on"

  standby: 
    # s3_wal_path: "s3://mybucket/spilo/acid-minimal-cluster/abcd1234-2a4b-4b2a-8c9c-c1234defg567/wal/14/"
    standby_host: "$SITEA_NAME.$SITEA_NS.svc.cluster.local"
    # standby_port: "5432"

  patroni: ## addons for patroni 'postgres.yml': https://github.com/zalando/patroni/blob/master/postgres0.yml, https://patroni.readthedocs.io/en/latest/SETTINGS.html#postgresql
    synchronous_mode: true
    synchronous_mode_strict: true


# Enables change data capture streams for defined database tables
#  streams:
#  - applicationId: test-app
#    database: foo
#    tables:
#      data.state_pending_outbox:
#        eventType: test-app.status-pending
#      data.state_approved_outbox:
#        eventType: test-app.status-approved
#      data.orders_outbox:
#        eventType: test-app.order-completed
#        idColumn: o_id
#        payloadColumn: o_payload



## Create Cluster clone
# restore a Postgres DB with point-in-time-recovery
# with a non-empty timestamp, clone from an S3 bucket using the latest backup before the timestamp
# with an empty/absent timestamp, clone from an existing alive cluster using pg_basebackup

#clone:
##  #uid: "889918f8-0c89-455d-b0bb-8cf0b799c011"
#  cluster: $SITEA_NAME
#  timestamp: "2022-06-15T16:50:00+00:00"
#  s3_endpoint: http://storage-minio.s3.svc.cluster.local:9000
#  s3_access_key_id: minio
#  s3_secret_access_key: minio123
#  s3_wal_path: "s3://foundation-pf/spilo/$SITEA_NAME/wal"

EOF


## SiteB: Check Statefulset ENVs
kubectl get statefulset -n $SITEB_NS $SITEB_NAME -o yaml | grep env -A100
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- env

## SiteB: View Master logs
kubectl logs -f -n $SITEB_NS $SITEB_NAME-0
  #... INFO: no action. I am (postgres-db-site-b-0), the standby leader with the lock
  
## SiteB: View Replica logs
kubectl logs -f -n $SITEB_NS $SITEB_NAME-1
  #... I am (postgres-db-site-b-1), a secondary, and following a standby leader (postgres-db-site-b-0)
  
## SiteB: View Patroni replicas
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- patronictl list

+ Cluster: postgres-db-site-b (7109447695371919429) --+---------+----+-----------+
| Member               | Host        | Role           | State   | TL | Lag in MB |
+----------------------+-------------+----------------+---------+----+-----------+
| postgres-db-site-b-0 | 10.42.0.196 | Standby Leader | running |  1 |           |
| postgres-db-site-b-1 | 10.42.0.198 | Replica        | running |  1 |         0 |
+----------------------+-------------+----------------+---------+----+-----------+

```

#### - Switch 'SiteB' to ACTIVE and update tables 
```yaml

## - Connect as appuser
DB_NAME=conjurdb
DB_USERNAME=conjuruser

## - Connect as superuser
DB_NAME=postgres
DB_USERNAME=postgres

DB_PASSWORD=$(kubectl get secret -n $SITEB_NS "$DB_USERNAME.$SITEB_NAME.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo -e "DB_NAME='$DB_NAME'\nDB_USERNAME='$DB_USERNAME'\nDB_PASSWORD='$DB_PASSWORD'\n"


## SiteB(Leader): Select from table
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

## SiteB(Replica): Select from table
kubectl exec -it -n $SITEB_NS $SITEB_NAME-1 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "


## SiteB: Switch to ACTIVE
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- curl -s -XPATCH -d '{ "standby_cluster": null}' localhost:8008/config | jq .


## SiteB: Insert into table
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " INSERT INTO test(name, notes) VALUES ('site-b_name', 'site-b_notes'); "


## SiteB(Leader): Select from table
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

## SiteB(Replica): Select from table
kubectl exec -it -n $SITEB_NS $SITEB_NAME-1 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

```


#### - Switch 'SiteA' to STANDBY to sync updates
```yaml

## Login into PG

## SiteA(Leader): Select from table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

## SiteA(Replica): Select from table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-1 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "


## SiteA: Switch to STANDBY
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- curl -s -XPATCH -d "{ \"standby_cluster\": { \"host\": \"$SITEB_NAME.$SITEB_NS.svc.cluster.local\", \"create_replica_methods\": [ \"basebackup_fast_xlog\" ] }}" localhost:8008/config | jq .
...
  "standby_cluster": {
    "host": "postgres-db-site-b.sb.svc.cluster.local",
    "create_replica_methods": [
      "basebackup_fast_xlog"
    ]
  }
...  


## SiteA(Leader): Insert row into table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " INSERT INTO test(name, notes) VALUES ('site-a_name', 'site-a_notes'); "


## SiteA(Leader): Select from table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "

## SiteA(Replica): Select from table
kubectl exec -it -n $SITEA_NS $SITEA_NAME-1 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " SELECT * FROM test; "


## SiteA: View Patroni replicas
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- patronictl list

```

#### - Switch back 'SiteB' to STANDBY and 'SiteA' to ACTIVE 
```yaml

## SiteA: Switch to ACTIVE
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- curl -s -XPATCH -d '{ "standby_cluster": null}' localhost:8008/config | jq .

## SiteA: View Patroni replicas
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- patronictl list

## SiteA(Leader): View reolication status
kubectl exec -it -n $SITEA_NS $SITEA_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " select * from pg_stat_replication; "


## SiteB: Switch to STANDBY
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- curl -s -XPATCH -d "{ \"standby_cluster\": { \"host\": \"$SITEA_NAME.$SITEA_NS.svc.cluster.local\", \"create_replica_methods\": [ \"basebackup_fast_xlog\" ] }}" localhost:8008/config | jq .

## SiteB: View Patroni replicas
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- patronictl list

## SiteB(Leader): View reolication status
kubectl exec -it -n $SITEB_NS $SITEB_NAME-0 -- psql -d $DB_NAME -U $DB_USERNAME \
-c " select * from pg_stat_replication; "

```



#### - Additional commands
```yaml
## - Use SiteA(Leader)
POD_NS=$SITEA_NS
POD_NAME=$SITEA_NAME-0
## - Use SiteA(Replica)
POD_NS=$SITEA_NS
POD_NAME=$SITEA_NAME-1

## - Use SiteB(Leader)
POD_NS=$SITEB_NS
POD_NAME=$SITEB_NAME-0
## - Use SiteB(Replica)
POD_NS=$SITEB_NS
POD_NAME=$SITEB_NAME-1


## View patroni logs
kubectl logs -f -n $POD_NS $POD_NAME


## View log files
kubectl exec -it -n $POD_NS $POD_NAME -- ls -la /home/postgres/pgdata/pgroot/pg_log

## View logs
kubectl exec -it -n $POD_NS $POD_NAME -- cat /home/postgres/pgdata/pgroot/pg_log/postgresql-3.log

## View patroni replicas
kubectl exec -it -n $POD_NS $POD_NAME -- patronictl list


## Reinit patroni replica $SITEA_NAME-1
kubectl exec -it -n $POD_NS $POD_NAME -- patronictl reinit $SITEA_NAME $SITEA_NAME-1

## Reinit patroni replica $SITEB_NAME-1
kubectl exec -it -n $POD_NS $POD_NAME -- patronictl reinit $SITEB_NAME $SITEB_NAME-1


## View patroni static settings: https://github.com/zalando/patroni/blob/master/postgres0.yml
kubectl exec -it -n $POD_NS $POD_NAME -- cat postgres.yml

## View Patroni dinamic settings: https://patroni.readthedocs.io/en/latest/SETTINGS.html#dynamic-configuration-settings 
kubectl exec -it -n $POD_NS $POD_NAME -- curl -s localhost:8008/config | jq .

## Edit patroni dynamic settings: 
kubectl exec -it -n $POD_NS $POD_NAME -- patronictl edit-config

## View PG config files
kubectl exec -it -n $POD_NS $POD_NAME -- ls /home/postgres/pgdata/pgroot/data/

## View pg_hba.conf(host-based authentication): https://www.postgresql.org/docs/current/auth-pg-hba-conf.html
kubectl exec -it -n $POD_NS $POD_NAME -- cat /home/postgres/pgdata/pgroot/data/pg_hba.conf

## View postgresql.conf:  https://github.com/postgres/postgres/blob/master/src/backend/utils/misc/postgresql.conf.sample
kubectl exec -it -n $POD_NS $POD_NAME -- cat /home/postgres/pgdata/pgroot/data/postgresql.conf

## View backup dir
kubectl exec -it -n $POD_NS $POD_NAME -- ls /home/postgres/pgdata/pgroot/data/pg_wal

## View wal-g backups 
kubectl exec -it -n $POD_NS $POD_NAME -- envdir "/run/etc/wal-e.d/env" wal-g backup-list

## View executable args
kubectl exec -it -n $POD_NS $POD_NAME -- cat /home/postgres/pgdata/pgroot/data/postmaster.opts


## Connect to the DB using pg-client POD
## - Use SiteA
SITE_NS=$SITEA_NS
SITE_NAME=$SITEA_NAME
## - Use SiteB
SITE_NS=$SITEB_NS
SITE_NAME=$SITEB_NAME

## - Use Master
PG_HOST=$SITE_NAME.$SITE_NS.svc.cluster.local
## - Use Replica
PG_HOST=$SITE_NAME-repl.$SITE_NS.svc.cluster.local

## - Connect as DB owner
DB_NAME=conjurdb
DB_USERNAME=conjuruser
DB_SECRET=conjuruser
## - Connect as APP user
DB_USERNAME=conjurdb_user
DB_SECRET=conjurdb-user
## - Connect as superuser
DB_NAME=postgres
DB_USERNAME=postgres
DB_SECRET=postgres

## Resolve DB Password
DB_PASSWORD=$(kubectl get secret -n $SITE_NS "$DB_SECRET.$SITE_NAME.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo -e "\nPG_HOST='$PG_HOST' \\n\
DB_NAME='$DB_NAME' \n\
DB_USERNAME='$DB_USERNAME' \n\
DB_PASSWORD='$DB_PASSWORD' \n"

## Create pg-client POD
kubectl delete pod pg-client -n $SITE_NS
kubectl run pg-client --rm --tty -i --restart='Never' -n $SITE_NS --image bitnami/postgresql \
--env="PGPASSWORD=$DB_PASSWORD" --command -- \
psql --set=sslmode=require --host $PG_HOST -U $DB_USERNAME -d $DB_NAME

-- list all tables
SELECT * FROM information_schema.tables;
SELECT * FROM information_schema.tables WHERE table_catalog = 'conjurdb' and table_schema = 'public';
-- list 
\q


## Cleanup postgresql resources manually if Zalando was stuck
## - Use SiteA
PG_NS=$SITEA_NS
PG_NAME=$SITEA_NAME

## - Use SiteB
PG_NS=$SITEB_NS
PG_NAME=$SITEB_NAME


kubectl delete postgresql -n $PG_NS $PG_NAME
kubectl delete statefulset -n $PG_NS $PG_NAME
kubectl delete service -n $PG_NS $PG_NAME
kubectl delete service -n $PG_NS $PG_NAME-repl
kubectl delete service -n $PG_NS $PG_NAME-config
kubectl delete pdb -n $PG_NS postgres-$PG_NAME-pdb
kubectl delete secret -n $PG_NS conjurdb-user.$PG_NAME.credentials.postgresql.acid.zalan.do
kubectl delete secret -n $PG_NS conjuruser.$PG_NAME.credentials.postgresql.acid.zalan.do
kubectl delete secret -n $PG_NS postgres.$PG_NAME.credentials.postgresql.acid.zalan.do
kubectl delete secret -n $PG_NS standby.$PG_NAME.credentials.postgresql.acid.zalan.do
kubectl delete pvc -n $PG_NS pgdata-$PG_NAME-0
```


#### - Connect to the 'clonned' DB and check that test data are present
```bash
## Connect to the DB conjur
PG_HOST=$CLUSTER_CLONE_NAME.$CLUSTER_CLONE_NS.svc.cluster.local
DB_NAME=conjurdb
## - Connect as DB owner
DB_USERNAME=conjuruser
DB_SECRET=conjuruser
## - Connect as APP user
#DB_USERNAME=conjurdb_user
#DB_SECRET=conjurdb-user
## - Connect as superuser
DB_NAME=postgres
DB_USERNAME=postgres
DB_SECRET=postgres

DB_PASSWORD=$(kubectl get secret -n $CLUSTER_NS "$DB_SECRET.$CLUSTER_NAME.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo -e "\nPG_HOST='$PG_HOST' \\n\
DB_NAME='$DB_NAME' \n\
DB_USERNAME='$DB_USERNAME' \n\
DB_PASSWORD='$DB_PASSWORD' \n"
kubectl delete pod pg-client -n $CLUSTER_CLONE_NS

kubectl run pg-client --rm --tty -i --restart='Never' -n $CLUSTER_CLONE_NS --image bitnami/postgresql \
--env="PGPASSWORD=$DB_PASSWORD" --command -- \
psql --set=sslmode=require --host $PG_HOST -U $DB_USERNAME -d $DB_NAME

-- list all tables
SELECT * FROM information_schema.tables WHERE table_catalog = 'conjurdb' and table_schema = 'public';
SELECT * FROM information_schema.tables;

-- list data in the table
SELECT * from public.test;
```



### - Install [conjur-oss](https://kloeckner-i.github.io/db-operator)
```sh

CONJUR_NS=conjur
kubectl create namespace "$CONJUR_NS"

# Add conjur repo: https://cyberark.github.io/helm-charts/index.yaml
helm repo add cyberark https://cyberark.github.io/helm-charts
helm search repo cyberark

## Generate init key
DATA_KEY="$(docker run --rm cyberark/conjur data-key generate)"
echo "DATA_KEY = $DATA_KEY"

## Get DB creds
PG_HOST=postgres-db-pg-cluster.pf.svc.cluster.local
DB_NAME=conjurdb
## Connect as DB owner
DB_USERNAME=conjuruser
DB_SECRET=conjuruser
## Connect as APP user
#DB_USERNAME=conjurdb_user
#DB_SECRET=conjurdb-user
DB_PASSWORD=$(kubectl get secret -n pf "$DB_SECRET.postgres-db-pg-cluster.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo "DB_PASSWORD = $DB_PASSWORD"

## Create custom values
cat << EOF  > conjur-oss_custom-values.yml  
logLevel: "debug" ## Authentication Errors are shown ONLY IN DEBUG MODE !!!
dataKey: "$DATA_KEY"  ## generate value: docker run --rm cyberark/conjur data-key generate
authenticators: "authn,authn-k8s/testAuthID"
account:
  ## maps to CONJUR_ACCOUNT env variable
  name: setupUser
  create: false ## "true" value does not work properly: POD fails in case of restarting  !!!
ssl:
  hostname: "conjur-oss"
service:
  external:
    enabled: false
internal:
  type: ClusterIP

## Use already existing DB
database:
  url: "postgres://$DB_USERNAME:$DB_PASSWORD@$PG_HOST:5432/$DB_NAME"

## Install PG Cluster
#postgres:
#  persistentVolume:
#    create: true
#    size: 2Gi
#    storageClass: local-path
EOF
cat ./conjur-oss_custom-values.yml

## Install conjur-oss
helm install oss -n $CONJUR_NS cyberark/conjur-oss --version=2.0.4 -f ./conjur-oss_custom-values.yml

kubectl get pod -n conjur

POD_NS=conjur
POD_LABEL=app=conjur-oss
POD_NAME=$(kubectl get pods -n $POD_NS -l "$POD_LABEL" -o jsonpath="{.items[0].metadata.name}")


## Create account
kubectl exec --namespace conjur $POD_NAME --container=conjur-oss conjurctl account create "setupUser" | tail -1

Created new account 'setupUser'
API key for admin: 30pgvre3172ks7zqj13q11zm2rn16m5zq423rkb7j1r67nqn6azrsz
```


### - Install [db-operator](https://kloeckner-i.github.io/db-operator)
TODO: wait for `v.1.5.0` helm version [here](https://kloeckner-i.github.io/db-operator/index.yaml) which contains my [PR-130](https://github.com/kloeckner-i/db-operator/pull/130): 
```sh
git clone https://github.com/zalando/postgres-operator.git
cd /mnt/d/Project/github/kloeckner-i/db-operator/charts/db-operator 
kubectl create ns db-oper
helm install db-oper . -n db-oper
```


### - Install [reloader](https://github.com/stakater/Reloader/blob/master/deployments/kubernetes/chart/reloader/values.yaml)
```sh
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update
kubectl create ns reloader
helm install reloader stakater/reloader  -n reloader --set reloader.ignoreConfigMaps=true
```


### - Install Minio
```sh
helm repo add minio https://helm.min.io/

cat <<EOF >>values-s3.yaml
image:
  repository: docker.io/minio/minio
mcImage:
  repository: docker.io/minio/mc
helmKubectlJqImage:
  repository: docker.io/bskim45/helm-kubectl-jq
mode: standalone
resources:
  requests:
    memory: 256Mi
replicas: 1
service:
  type: NodePort
  nodePort: 31900
persistence:
  storageClass: local-path
  size: 500Mi
securityContext:
  enabled: true
makeBucketJob:
  securityContext:
    enabled: true
updatePrometheusJob:
  securityContext:
    enabled: true
configPathmc: "/tmp"    # avoids /etc permission denied for non-root
EOF

kubectl create ns s3

helm install storage minio/minio --version 8.0.10 --namespace s3 --values values-s3.yaml --set accessKey=minio --set secretKey=minio123 --set buckets[0].name=foundation-pf --set buckets[0].policy=none --set buckets[0].purge=true


kubectl apply --namespace s3 -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "false"
  name: minio
spec:
  rules:
  - host: localhost
    http:
      paths:
      - backend:
          service:
            name: storage-minio
            port:
              number: 9000
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - localhost
EOF

## Access Minio UI
http://localhost:8081/
```


### - Install Velero
##### - Prepare [velero.yaml](https://github.com/vmware-tanzu/helm-charts/blob/main/charts/velero/values.yaml) and install `velero` helm chart
```yaml
cat <<EOF >>velero.yaml
image:
  tag: v1.8.1
configuration:
  provider: aws # Cloud provider being used (e.g. aws, azure, gcp).
  backupStorageLocation:
    name: aws
    default: true
    provider: velero.io/aws
    bucket: foundation-pf
    config:
      region: minio
      s3ForcePathStyle: true
      publicUrl: http://localhost:8081/
      s3Url: http://storage-minio.s3.svc.cluster.local:9000
credentials:
  useSecret: true
  secretContents:
    cloud: |
      [default]
      aws_access_key_id = minio
      aws_secret_access_key = minio123      
snapshotsEnabled: false
configMaps:
  restic-restore-action-config:
    labels:
      velero.io/plugin-config: ""
      velero.io/restic: RestoreItemAction
    data:
      image: gcr.io/heptio-images/velero-restic-restore-helper:v1.1.0
deployRestic: true ## use restic backup tool : https://restic.readthedocs.io/en/latest/manual_rest.html

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.2.0
    imagePullPolicy: IfNotPresent
    volumeMounts:
      - mountPath: /target
        name: plugins 
EOF


kubectl create ns velero

helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero --namespace velero --version 2.29.4 -f velero.yaml

## Install velero tool
wget https://github.com/vmware-tanzu/velero/releases/download/v1.8.1/velero-v1.8.1-linux-amd64.tar.gz
tar -zxvf velero-v1.8.1-linux-amd64.tar.gz
sudo mv velero-v1.8.1-linux-amd64/velero /usr/local/bin/.

```


##### - Debug Velero backups
```yaml

## Install dummy-service helm chart: https://vladimir22.github.io/dummy-service/index.yaml
helm repo add vladimir22 https://vladimir22.github.io/dummy-service
HELM_VERSION=1.0.2
helm repo update vladimir22
helm install ds -n default vladimir22/dummy-service --version $HELM_VERSION --set db.host=$DB_HOST --set db.adminUsername=$DB_USERNAME --set db.adminPassword=$DB_PASSWORD
## Check DB status: curl http://localhost:8081/dummy-service/db

## Backup default namespace
BACKUP_NAME=ns-default
velero backup create $BACKUP_NAME --include-namespaces default

velero backup describe $BACKUP_NAME --details
velero backup logs $BACKUP_NAME
## Check backups using UI Minio: http://localhost:8081/

## Delete dummy-service helm chart
helm delete ds -n default

## Restore default namespace
velero restore create --from-backup $BACKUP_NAME --include-namespaces default

## Check DB status again: curl http://localhost:8081/dummy-service/db
```
