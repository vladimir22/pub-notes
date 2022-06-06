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
- [Spilo](https://github.com/zalando/spilo) is a Docker image that provides PostgreSQL and Patroni bundled together.
  + Notes:
    - Test

- [Patroni](https://github.com/zalando/patroni#how-patroni-works) is a template for PostgreSQL HA based on python scripts
  + Notes:
    - [Standby](https://opensource.zalando.com/postgres-operator/docs/user.html#setting-up-a-standby-cluster) cluster is a Patroni feature that first clones a database, and keeps replicating changes in readonly mode

- [Zalando](https://github.com/zalando/postgres-operator) is a postgres-operator that manages Patroni HA Replica PODs depend on created [postgresql](https://github.com/zalando/postgres-operator/blob/master/docs/reference/cluster_manifest.md) object (mainfest):
  + Notes:
    - Zalando is not well documented and developed rapidly, reading [sources](https://github.com/zalando/postgres-operator/tree/master/pkg) is a must!
    -  [PostgreSQL on K8s at Zalando: Two years in production](https://av.tib.eu/media/52142) video with working details and known issues

- Pgpool II
  + Notes:
    - [PgBouncer vs Pgpool-II](https://scalegrid.io/blog/postgresql-connection-pooling-part-4-pgbouncer-vs-pgpool/#:~:text=PgBouncer%20allows%20limiting%20connections%20per,overall%20number%20of%20connections%20only.&text=PgBouncer%20supports%20queuing%20at%20the,i.e.%20PgBouncer%20maintains%20the%20queue)
#### - Install [Zalando](https://github.com/zalando/postgres-operator) postgres-operator
```sh
git clone https://github.com/zalando/postgres-operator.git
cd /mnt/d/Project/github/zalando/postgres-operator/charts/postgres-operator
kubectl create ns pgo
helm install pgo . -n pgo
```


#### - Create ConfigMap with [Custom ENVs for all Cluster PODs](https://github.com/zalando/postgres-operator/blob/master/docs/administrator.md#custom-pod-environment-variables) by default
```yaml

## Create Default ENVs ConfigMap for PG Statefulset
kubectl apply -n $CLUSTER_NS -f - <<EOF
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
```

#### - Link created ConfigMap to Zalando Operator
```yaml
## Link ConfigMap inside the OperatorConfiguration
kubectl edit OperatorConfiguration -n pgo pgo-postgres-operator
...
configuration:
  kubernetes:
    pod_environment_configmap: pf/postgres-pod-config
...

## Restart Zalando Operator
kubectl delete pods -n pgo --all
```

#### - View Zalando Operator logs
```bash
POD_NS=pgo
POD_LABEL=app.kubernetes.io/name=postgres-operator
POD_NAME=$(kubectl get pods -n $POD_NS -l "$POD_LABEL" -o jsonpath="{.items[0].metadata.name}")
klo -n $POD_NS $POD_NAME
```


#### - Create 'original' empty `postgresql` cluster
```yaml
CLUSTER_NS=pf
CLUSTER_NAME=postgres-db-original 

kubectl apply -n $CLUSTER_NS -f - <<EOF
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: $CLUSTER_NAME ## Cluster name prefix must match with "teamId" value !!!
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
EOF

## Check Statefulset ENVs
kubectl get statefulset -n $CLUSTER_NS $CLUSTER_NAME -o yaml | grep env -A100
kubectl exec -it -n $CLUSTER_NS $CLUSTER_NAME-0 -- env

## View Cluster POD logs
kubectl logs -f -n $CLUSTER_NS $CLUSTER_NAME-0 
```


#### - Connect to the 'original' DB and write test data
```bash
## Connect to created database
PG_HOST=$CLUSTER_NAME.$CLUSTER_NS.svc.cluster.local
DB_NAME=conjurdb
## Connect as DB owner
DB_USERNAME=conjuruser
DB_SECRET=conjuruser
## Connect as APP user
#DB_USERNAME=conjurdb_user
#DB_SECRET=conjurdb-user
## Connect as postgres user
#PG_USERNAME=postgres
#DB_SECRET=postgres
DB_PASSWORD=$(kubectl get secret -n $CLUSTER_NS "$DB_SECRET.$CLUSTER_NAME.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo "DB_PASSWORD = $DB_PASSWORD"
kubectl delete pod pg-client
kubectl run pg-client --rm --tty -i --restart='Never' --namespace default --image bitnami/postgresql \
--env="PGPASSWORD=$DB_PASSWORD" --command -- \
psql --set=sslmode=require --host $PG_HOST -U $DB_USERNAME -d $DB_NAME

-- create table
CREATE TABLE test (
    test_id bigserial primary key,
    test_name varchar(20) NOT NULL,
    test_desc text NOT NULL,
    date_added timestamp default NOW()
);
-- insert data
INSERT INTO test(test_name, test_desc) VALUES ('test_name_value', 'test_desc_value');

-- list all tables
SELECT * FROM information_schema.tables WHERE table_catalog = 'conjurdb' and table_schema = 'public';

-- list data in the table
SELECT * from public.test;

\q
```


#### - View created backups
```bash
## View backups in the POD
kubectl exec -it -n $CLUSTER_NS $CLUSTER_NAME-0 -- envdir "/run/etc/wal-e.d/env" wal-g backup-list

## View Minio UI with created backups
http://localhost:8081/minio/foundation-pf/spilo/
```


#### - Create postgresql '[clone](https://github.com/zalando/postgres-operator/blob/master/docs/user.md#clone-from-s3)'
```bash
## Create postgresql clone
CLUSTER_CLONE_NS=pf
CLUSTER_CLONE_NAME=postgres-db-clone

kubectl create ns $CLUSTER_CLONE_NS

kubectl apply -n $CLUSTER_CLONE_NS -f - <<EOF
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: $CLUSTER_CLONE_NAME
spec:

  teamId: "postgres-db"
  volume:
    size: 128Mi
  numberOfInstances: 1
  users:
    ## Create users, set up permissions 
    conjuruser:  # database owner
    - superuser
    - createdb
    conjurdb_user: []  # role for application

  ## Create db & assign owner
  databases:
    conjurdb: conjuruser  # dbname: owner

  ## Create db with default users: https://github.com/zalando/postgres-operator/blob/master/docs/user.md#default-nologin-roles
  #preparedDatabases:
  #  foo: {}

  postgresql:
    version: "14"

  clone:
    #uid: "889918f8-0c89-455d-b0bb-8cf0b799c011"
    cluster: $CLUSTER_NAME
    timestamp: "2022-06-06T16:50:00+00:00"
    s3_endpoint: http://storage-minio.s3.svc.cluster.local:9000
    s3_access_key_id: minio
    s3_secret_access_key: minio123
    s3_wal_path: "s3://foundation-pf/spilo/$CLUSTER_NAME/wal"
EOF

## Check Statefulset ENVs
kubectl get statefulset -n $CLUSTER_CLONE_NS $CLUSTER_CLONE_NAME -o yaml | grep env -A100

## View PG Cluster logs
kubectl logs -f -n $CLUSTER_CLONE_NS $CLUSTER_CLONE_NAME-0
  ... INFO: no action. I am (postgres-db-clone-0) the leader with the lock
  
## View already created backups 
kubectl exec -it -n $CLUSTER_CLONE_NS $CLUSTER_CLONE_NAME-0 -- envdir "/run/etc/wal-e.d/env" wal-g backup-list
```


#### - Connect to the 'clonned' DB and check that test data are present
```bash
## Connect to the DB conjur
PG_HOST=$CLUSTER_CLONE_NAME.$CLUSTER_CLONE_NS.svc.cluster.local
DB_NAME=conjurdb
## Connect as DB owner
DB_USERNAME=conjuruser
DB_SECRET=conjuruser
## Connect as APP user
#DB_USERNAME=conjurdb_user
#DB_SECRET=conjurdb-user
DB_PASSWORD=$(kubectl get secret -n $CLUSTER_CLONE_NS "$DB_SECRET.$CLUSTER_CLONE_NAME.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
echo -e "\nPG_HOST='$PG_HOST' \\n\
DB_NAME='$DB_NAME' \n\
DB_USERNAME='$DB_USERNAME' \n\
DB_PASSWORD='$DB_PASSWORD' \n"
kubectl delete pod pg-client
kubectl run pg-client --rm --tty -i --restart='Never' --namespace default --image bitnami/postgresql \
--env="PGPASSWORD=$DB_PASSWORD" --command -- \
psql --set=sslmode=require --host $PG_HOST -U $DB_USERNAME -d $DB_NAME

-- list all tables
SELECT * FROM information_schema.tables WHERE table_catalog = 'conjurdb' and table_schema = 'public';

-- list data in the table
SELECT * from public.test;
```


#### - Optional: Cleanup postgresql resources manually if Zalando Operator stuck
```bash
## Optional: Cleanup postgresql resources manually if Zalando was stuck
PG_NS=$CLUSTER_CLONE_NS
PG_NAME=$CLUSTER_CLONE_NAME
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
