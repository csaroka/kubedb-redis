# kubedb-redis

Using KubeDB, create a Redis database cluster

## Clone the repository
`$ git clone https://github.com/csaroka/kubedb-redis.git` \
`$ cd kubedb-redis`

## Install the KubeDB operator
Reference: https://kubedb.com/docs/0.11.0/setup/install/#using-helm \
Note: Requires Helm client installed and Tiller server initialized on the target Kubernetes cluster

`$ helm repo add appscode https://charts.appscode.com/stable/` \
`$ helm repo update` \
`$ helm search appscode/kubedb` \
`$ helm install appscode/kubedb --name kubedb-operator --version 0.11.0 -n kube-system` \
`$ kubectl get crds -l app=kubedb -w` \
`$ helm install appscode/kubedb-catalog --name kubedb-catalog --version 0.11.0 -n kube-system` \
`$ kubectl get pods --all-namespaces -l app=kubedb --watch` \
`$ kubectl get crd -l app=kubedb`

## Install the KubeDB CLI
Reference: https://kubedb.com/docs/0.11.0/setup/install/#install-kubedb-cli

### Linux amd 64-bit
`$ wget -O kubedb https://github.com/kubedb/cli/releases/download/0.11.0/kubedb-linux-amd64 && chmod +x kubedb && sudo mv kubedb /usr/local/bin/`

### Mac 64-bit
`$ wget -O kubedb https://github.com/kubedb/cli/releases/download/0.11.0/kubedb-darwin-amd64 && chmod +x kubedb && sudo mv kubedb /usr/local/bin/`

## Create a Redis database cluster
Reference: https://kubedb.com/docs/0.11.0/guides/redis/quickstart/quickstart/

### Create a storage class for persistant (durable) volumes
`$ cat kubedb-sc.yaml`
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: kubedb-sc
provisioner: kubernetes.io/vsphere-volume
parameters:
    diskformat: thin
```

`$ kubectl apply -f kubedb-sc.yaml`

### Create a namespace and update context
`$ kubectl create ns kubedb-test` 

### Apply the spec for creating a Redis 5.0.3-v1 database cluster
Reference: https://kubedb.com/docs/0.11.0/concepts/databases/redis/#redis-spec

`$ cat kubedb-redis-crd.yaml`
```
apiVersion: kubedb.com/v1alpha1
kind: Redis
metadata:
  name: redis-cluster
  namespace: kubedb-test
spec:
  name: redis-cluster
  version: "5.0.3-v1"
  storageType: Durable
  storage:
    storageClassName: "thin"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 2Gi
  mode: cluster
  cluster:
    master: 5
    replicas: 2
  terminationPolicy: WipeOut
  ```
  If necessary, see reference for spec definitions and options. For instance, the `terminationPolicy: WipeOut` option is used for testing convienance; other safety options such as `Pause` (default) or `DoNotTerminate` should be considered for production.  Otherwise apply the spec to the target cluster

  `$ kubectl apply -f kubedb-redis-crd.yaml`

### View the Redis database cluster details and supporitng Kubernetes infrastructure components

List Redis clusters \
`$ kubedb get rd -n kubedb-test` \
View Redis cluster details \
`$ kubedb describe rd redis-cluster -n kubedb-test` \
View Stateful Set \
`$ kubectl get statefulset -n kubedb-test` \
View Pesistant Volume Claims \
`$ kubectl get pvc -n kubedb-test` \
View Persistant Volumes \
`$ kubectl get pv -n kubedb-test` \
View Service w/ clusterIP \
`$ kubectl get service -n kubedb-test`

### Destroy the Redis database cluster and Kubernetes namespace

For crd with `terminationPolicy: WipeOut`
```
$ kubectl delete -n kubedb-test rd/redis-cluster`
```

