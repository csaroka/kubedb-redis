# kubedb-redis

Using KubeDB, create a Redis database cluster

## Clone the repository
`$ git clone https://github.com/csaroka/kubedb-redis.git` \
`$ cd kubedb-redis`

## Install the KubeDB operator
Reference: https://kubedb.com/docs/0.11.0/setup/install/#using-helm \
Note: Requires Helm client installed and Tiller server initialized on the target Kubernetes cluster

Add the KubeDB repo to Helm \
`$ helm repo add appscode https://charts.appscode.com/stable/` \
`$ helm repo update` \
`$ helm search appscode/kubedb` 

Instal the kubedb-operator chart to the kube-system namespace\
`$ helm install appscode/kubedb --name kubedb-operator --version 0.11.0 --namespace kube-system`

Monitor the kubedb-operator deployment status with the following command until the AVAILABLE number of pods = 1 \
`$ kubectl --namespace=kube-system get deployments -l "release=kubedb-operator, app=kubedb"`

List the KubeDB CRDs \
`$ kubectl get crds -l app=kubedb`

Instal the kubedb-catalog chart to the kube-system namespace \
`$ helm install appscode/kubedb-catalog --name kubedb-catalog --version 0.11.0 --namespace kube-system`

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
    storageClassName: "kubedb-sc"
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
  If necessary, see reference for spec definitions and options. For instance, the `terminationPolicy: WipeOut` option is used for testing convienance; other safety options such as `Pause` (default) or `DoNotTerminate` should be considered for production. 

  Apply the spec to the target cluster \
  `$ kubectl apply -f kubedb-redis-crd.yaml` 
  
  Monitor the Redis database cluster status until 5 masters exist, each with 2 replicas; total of 3 pods per statefulset \
  `$ kubectl get statefulsets -n kubedb-test`

## View the Redis database cluster details and supporitng Kubernetes infrastructure components

List Redis clusters in the kubedb-test namespace \
`$ kubedb get rd -n kubedb-test` \
View Redis cluster details \
`$ kubedb describe rd redis-cluster -n kubedb-test` \
View the Kubernetes StatefulSets  \
`$ kubectl get statefulsets -n kubedb-test` \
View the Kubernetes  Pesistant Volume Claims \
`$ kubectl get pvc -n kubedb-test` \
View the Kubernetes  Persistant Volumes \
`$ kubectl get pv -n kubedb-test` \
View the Kubernetes Service \
`$ kubectl get service -n kubedb-test`

## Destroy the Redis database cluster, Kubernetes namespace, and Helm KubeDB charts

#### Delete the Redis database cluster
Note: For crd with *terminationPolicy: WipeOut* only \
`$ kubectl delete -n kubedb-test rd/redis-cluster` \
`$ kubectl delete -n kubedb-test service/kubedb`

#### Delete the Kubernetes namespace
`$ kubectl delete ns kubedb-test`

#### Delete the Helm KubeDB charts
`$ helm delete kubedb-catalog --purge` \
`$ helm delete kubedb-operator --purge`