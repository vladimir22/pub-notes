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

### - Install Postgres

#### - Install [postgres-operator](https://github.com/zalando/postgres-operator)
```sh
git clone https://github.com/zalando/postgres-operator.git
cd /mnt/d/Project/github/zalando/postgres-operator/charts/postgres-operator
kubectl create ns pgo
helm install pgo . -n pgo
```

#### - Install [minimal-postgres](https://github.com/zalando/postgres-operator/blob/master/manifests/minimal-postgres-manifest.yaml) DB
```yaml
kubectl apply -n pf -f - <<EOF
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: postgres-db-pg-cluster
spec:
  teamId: "postgres-db"
  volume:
    size: 128Mi
  numberOfInstances: 2
  users:
    zalando:  # database owner
    - superuser
    - createdb
    foo_user: []  # role for application foo
  databases:
    foo: zalando  # dbname: owner
  preparedDatabases:
    bar: {}
  postgresql:
    version: "14"
EOF
```

#### - Test DB access
```sh
DB_HOST=postgres-db-pg-cluster.pf.svc.cluster.local
DB_NAME=postgres
DB_USERNAME=postgres
## Get password from the generated secret
DB_PASSWORD=$(kubectl get secret -n pf "postgres.postgres-db-pg-cluster.credentials.postgresql.acid.zalan.do" -o jsonpath='{.data.password}' | base64 --decode)
## Connect to the DB
kubectl run pg-client --rm --tty -i --restart='Never' --namespace default --image bitnami/postgresql \
--env="PGPASSWORD=$DB_PASSWORD" --command -- \
psql --set=sslmode=require --host $DB_HOST -U $DB_USERNAME -d $DB_NAME
\conninfo
\q
## Delete client POD
kubectl delete pod pg-client
```

#### - Install [db-operator](https://kloeckner-i.github.io/db-operator)
TODO: wait for `v.1.5.0` helm version [here](https://kloeckner-i.github.io/db-operator/index.yaml) which contains my [PR-130](https://github.com/kloeckner-i/db-operator/pull/130): 
```sh
git clone https://github.com/zalando/postgres-operator.git
cd /mnt/d/Project/github/kloeckner-i/db-operator/charts/db-operator 
kubectl create ns db-oper
helm install db-oper . -n db-oper
```

#### - Install [reloader](https://github.com/stakater/Reloader/blob/master/deployments/kubernetes/chart/reloader/values.yaml)
```sh
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update
kubectl create ns reloader
helm install reloader stakater/reloader  -n reloader --set reloader.ignoreConfigMaps=true
```

