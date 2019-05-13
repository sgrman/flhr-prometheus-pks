# prometheus-operator on pks

## Prom-Operator Reference
https://github.com/helm/charts/tree/master/stable/prometheus-operator

## Pre-reqs

* K8s Cluster (tested with PKS 1.4)
* Default StorageClass (Optional, only if using Persitent Storage)
* Helm client/server version =>2.13.1
* Assumes BOSH knowledge

## Install the Operator

### Record the Master's IP(s) for later use
```
bosh vms
```

### Get ETCD certs from Master(s)
```
bosh scp -d service-instance_<blah> master/0:/var/vcap/jobs/etcd/config/etcd-ca.crt .
bosh scp -d service-instance_<blah> master/0:/var/vcap/jobs/etcd/config/etcdctl.* .
```

### Create a `monitoring` namespace for the operator to live in

```
kubectl create ns monitoring
```

### Set your context to that namespace
```
kubens monitoring
```
OR
```
kubectl config set-context $(kubectl config current-context) --namespace=monitoring
```

### Create ETCD certs secret in K8s using the certs copied from the Master

```
kubectl create secret generic etcd-client \
--from-file=etcd-ca.crt \
--from-file=etcdctl.crt \
--from-file=etcdctl.key
```

### Install the Operator with `helm install`

#### Without Persistent Storage

**`override-no-persist.yaml`**
```
grafana:
  adminPassword: "VMware1!"
  ingress:
    enabled: true
    hosts:
      - grafana.ing.flhrnet.local  ## Change to your ingress DNS for the cluster
    path: "/*"
prometheus:
  ingress:
    enabled: true
    hosts: 
      - prometheus.ing.flhrnet.local  ## Change to your ingress DNS for the cluster
    paths:
      - "/*"
  prometheusSpec:
    secrets:
      - etcd-client
alertmanager:
  ingress:
    enabled: true
    hosts: 
      - alertmanager.ing.flhrnet.local  ## Change to your ingress DNS for the cluster
    paths:
      - "/*"
kubelet:
  serviceMonitor:
    https: true
kubeControllerManager:
  endpoints:
    - 172.16.0.2     ## Change to the Master IP(s) recorded from "bosh vms"
kubeScheduler:
  endpoints:
    - 172.16.0.2     ## Change to the Master IP(s) recorded from "bosh vms"
kubeEtcd:
  endpoints:
    - 172.16.0.2     ## Change to the Master IP(s) recorded from "bosh vms"
  serviceMonitor:
    insecureSkipVerify: true
    scheme: https
    caFile: "/etc/prometheus/secrets/etcd-client/etcd-ca.crt"
    certFile: "/etc/prometheus/secrets/etcd-client/etcdctl.crt"
    keyFile: "/etc/prometheus/secrets/etcd-client/etcdctl.key"
```

```
helm install -f override-no-persist.yaml --name prom-operator stable/prometheus-operator
```

#### With Persistent Storage

**`override-persist.yaml`**
```    
grafana:
  adminPassword: "VMware1!"
  ingress:
    enabled: true
    hosts:
      - grafana.ing.flhrnet.local  ## Change to your ingress DNS for the cluster
    path: "/*"
## Persistence
  persistence:
    enabled: true
    accessModes: ["ReadWriteOnce"]
    size: 20Gi
prometheus:
  ingress:
    enabled: true
    hosts: 
      - prometheus.ing.flhrnet.local  ## Change to your ingress DNS for the cluster
    paths:
      - "/*"
  prometheusSpec:
    secrets:
      - etcd-client
## Persistence
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 30Gi
alertmanager:
  ingress:
    enabled: true
    hosts: 
      - alertmanager.ing.flhrnet.local  ## Change to your ingress DNS for the cluster
    paths:
      - "/*"
## Persistence
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
kubelet:
  serviceMonitor:
    https: true
kubeControllerManager:
  endpoints:
    - 172.16.0.2     ## Change to the Master IP(s) recorded from "bosh vms"
kubeScheduler:
  endpoints:
    - 172.16.0.2     ## Change to the Master IP(s) recorded from "bosh vms"
kubeEtcd:
  endpoints:
    - 172.16.0.2     ## Change to the Master IP(s) recorded from "bosh vms"
  serviceMonitor:
    insecureSkipVerify: true
    scheme: https
    caFile: "/etc/prometheus/secrets/etcd-client/etcd-ca.crt"
    certFile: "/etc/prometheus/secrets/etcd-client/etcdctl.crt"
    keyFile: "/etc/prometheus/secrets/etcd-client/etcdctl.key"
```

```
helm install -f override.yaml --name prom-operator stable/prometheus-operator
```

## Delete the Operator and CRDs

```
helm delete prom-operator --purge
```

```
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
```
