# MiniKube deployment

## Setup
- install minikube
- setup minikube context
```bash
kubectl config get-contexts

# switch to minikube
kubectl config use-context minikube

# switch to EKS
EKS=$(aws ssm get-parameters --names uniframe-dev-ssm-eks-cluster-name --query "Parameters[0].Value" | tr -d '"') ; aws eks update-kubeconfig --name $EKS
``` 
- startup minikube
```bash
# with vm=true if cannot enable ingress
minikube delete

minikube config set memory 8192
minikube config set cpus 4
# minikube start --vm=true --disk
minikube start --vm=true --memory=8192 --cpus=4 --disk-size=50g

minikube addons enable ingress

```

- install prometheus and Graphana
https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

**Default password is `prom-operator`**
https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
```



- create namespace nm
```bash
kubectl apply -f basic.yaml
```

- create postgres
```
# remove previous deployed postgresql
helm delete postgresql
kubectl get pvc -l "app=postgresql"
kubectl delete pvc -l "app=postgresql"

# Important: postgresql have to install in nm.
# otherwise, the service is postgres.NAMESPACE
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install postgresql -n nm\
    --set postgresqlUsername=postgres \
    --set postgresqlPassword=postgres \
    bitnami/postgresql

```

After install, run command below to forward 5432 port. N.B. it will run background, remember to kill the process
```bash
kubectl port-forward --namespace nm svc/postgresql 5432:5432 &
```

Then, you can use psql to login postgres with username and password `postgres`
```bash
psql --host localhost -U postgres -d postgres -p 5432
```
**In backend, we use host `postgresql` in sqlalchemy connection string, since it is inside k8s**

Then create a database `nm`
```
create database nm;
```


**I don't know how to change the host name, so I add SQLALCHEMY connection string to make backend to connect pg**


- install redis
``` bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis -n nm\
    --set replica.replicaCount=1\
    --set auth.password=abc
```
**N.B. when installing redis via helm chart, if password is not given, a random password will be generated**

Get help message
```
Redis&trade; can be accessed on the following DNS names from within your cluster:

    redis-master.nm.svc.cluster.local for read/write operations (port 6379)
    redis-replicas.nm.svc.cluster.local for read-only operations (port 6379)



To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace nm redis -o jsonpath="{.data.redis-password}" | base64 --decode)

To connect to your Redis&trade; server:

1. Run a Redis&trade; pod that you can use as a client:

   kubectl run --namespace nm redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:6.2.5-debian-10-r11 --command -- sleep infinity

   Use the following command to attach to the pod:

   kubectl exec --tty -i redis-client \
   --namespace nm -- bash

2. Connect using the Redis&trade; CLI:
   redis-cli -h redis-master -a $REDIS_PASSWORD
   redis-cli -h redis-replicas -a $REDIS_PASSWORD

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace nm svc/redis-master 6379:6379 &
    redis-cli -h 127.0.0.1 -p 6379 -a $REDIS_PASSWORD
```


- setup minikube docker env internally
```bash
eval $(minikube docker-env)
```
- build application images for minikube
Go to backend, frontend and doc repo, use `scripts/manual_build_minikube_docker.sh` script to build images


- helm install
```bash
helm upgrade --install  uniframe-local nm-chart  --namespace nm        
```
- update hostfile
Get the IP address from ingress information
```
kubectl get ingress -n nm 
NAME                              CLASS    HOSTS                                     ADDRESS        PORTS   AGE
uniframe-local-backend-ingress    <none>   api.uniframe-local.nl                     192.168.64.2   80      9m22s
uniframe-local-doc-ingress        <none>   doc.uniframe-local.nl                     192.168.64.2   80      9m22s
uniframe-local-frontend-ingress   <none>   www.uniframe-local.nl,uniframe-local.nl   192.168.64.2   80      9m22s
```

**N.B.**, if you don't see address ip, run `minikube addons enable ingress` again.


We presume the host domain is uniframe-local.nl, which is built as ENV in our backend and frontend app. Add this record to /etc/hosts to point to 192.168.64.2. e.g.
```
192.168.64.2 uniframe-local.nl
192.168.64.2 www.uniframe-local.nl
192.168.64.2 api.uniframe-local.nl
192.168.64.2 doc.uniframe-local.nl
```

### Build latest uniframe application docker image
If the application code change, use these scripts to build the latest docker images
- backend: `uniframe-backend/scripts/deploy/manual_build_minikube_docker.sh`
- frontend: `name-matching-frontend/scripts/manual_build_minikube_docker.sh`
- doc: `name-matching-docs/scripts/manual_build_local_docker.sh`





### uninstall application chart
Helm uninstall uniframe-local -n nm   


#### debug
kubectl exec -it  -n nm uniframe-local-backend-deployment-8dbc99795-hmrbr   -- /bin/bash 

## Debug with backend or frontend
Go to ECR uniframe-dev-backend-local repostiory, delete the latest tag image. 
~~Use script `./scripts/deploy/manual_build_minikube_docker.sh` to build and push the latest image.~~
It seems that `./scripts/deploy/manual_build_minikube_docker.sh` has problem. For now, please use the script as reference and manual build backend local docker image 

Then reinstall helm chart to load the latest change of backend

**If the backend behavior is suspicous, you can use `kubectl exec -it -n nm POD_NAME -- /bin/bash `** to check the environment variable or code




### connect to postgres via psql
```
PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    postgresql.nm.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace nm postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

To connect to your database run the following command:

    kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace nm --image docker.io/bitnami/postgresql:11.13.0-debian-10-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql -U postgres -d postgres -p 5432



To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace nm svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
```

#### misc
apt-get install postgresql postgresql-contrib

### Diagnose service and pod DNS
install dnsutils inside pod container
```bash
apt update && apt install dnsutils
```

