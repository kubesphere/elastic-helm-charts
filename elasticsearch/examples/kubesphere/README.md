## Install Guide

Add repo:

```
$ helm repo add elastic https://helm.elastic.co
$ helm repo update
```

Clone codes from github:

```
$ git clone https://github.com/kubesphere/elastic-helm-charts
$ cd /elastic-helm-charts/elasticsearch/examples/kubespheretre
$ tree .
|-- Makefile
|-- README.md
|-- client.yaml
|-- data.yaml
|-- es-patch-torlence.json
|-- master.yaml
`-- test
    `-- goss.yaml
```

### The reference configuration

master node

```
clusterName: "elasticsearch"
nodeGroup: "master"

roles:
  master: "true"
  ingest: "false"
  data: "false"
  ml: "false"
  remote_cluster_client: "false"

# tolerations for master node
tolerations:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  value: ''

replicas: 3
minimumMasterNodes: 2

esJavaOpts: "-Xmx1g -Xms1g"

resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"

```

data node

```
clusterName: "elasticsearch"
nodeGroup: "data"

roles:
  master: "false"
  ingest: "true"
  data: "true"
  ml: "false"
  remote_cluster_client: "false"


# tolerations for data node
tolerations:
- effect: NoSchedule
  key: node-role.kubernetes.io/master
  value: ''

# replicas 
replicas: 3
minimumMasterNodes: 2

# Starting from version 7.11, Elasticsearch automatically sizes JVM heap based on a node's roles and total memory.
# This configuration should not be modified.
esJavaOpts: null

# Adjust suitable limits  for resources
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"

# Pv storage should be adjusted to 20Gi+
volumeClaimTemplate:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 30Gi
```

### Use make 

Usage of `make cmdline`

```
default: test

include ../../../helpers/examples.mk

PREFIX := elasticsearch
NAMESPACE ?= default
TIMEOUT := 1200s
PATCH := $(shell cat es-patch-torlence.json)

# Install
# eg.  make install -e NAMESPACE=test
install:
        helm upgrade --install  $(PREFIX)-master ../../ --values master.yaml -n $(NAMESPACE) --set imageTag="7.11.1"
        helm upgrade --install  $(PREFIX)-data ../../  --values data.yaml -n  $(NAMESPACE) --set imageTag="7.11.1"

# Patch custom tolerences 
patch:
        kubectl patch sts elasticsearch-master --patch  '$(PATCH)'
        kubectl patch sts elasticsearch-data --patch  '$(PATCH)'

# Uninstall
# eg: make uninstall -e NAMESPACE=test
uninstall:
        helm del $(PREFIX)-master -n  $(NAMESPACE)
        helm del $(PREFIX)-data -n  $(NAMESPACE)

```

#### install

```
make install [-e NAMESPACE=xxx]
```

#### patch

Skip this step if status of the pods is healthy

In my way, a pending status occured like below:

```
$ kubectl get pods
NAME                     READY   STATUS     RESTARTS   AGE
busybox-sleep            1/1     Running    1          23h
elasticsearch-data-0     0/1     Init:0/1   0          14s
elasticsearch-data-1     0/1     Init:0/1   0          14s
elasticsearch-data-2     0/1     Pending    0          14s
elasticsearch-master-0   0/1     Init:0/1   0          15s
elasticsearch-master-1   0/1     Init:0/1   0          15s
elasticsearch-master-2   0/1     Pending    0          15s
```

Check out what happend:

```
$ kubectl describe pod -l  app=elasticsearch-master
.. 1 node(s) had taints that the pod didn't tolerate, 2 node(s) didn't match pod affinity/anti-affinity...
```

Modify your own tolerences in the file `es-patch-torlence.json`:

```
{"spec":{"template":{"spec":{"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master","value":""}]}}}}
```

Run `make patch`:

```
make patch [-e NAMESPACE=xxx]
```

Run this below if needed:

```
kubectl delete pod -l  app=elasticsearch-master [-n xxx]
kubectl delete pod -l  app=elasticsearch-data [-n xxx]
```

#### uninstall


```
make uninstall [-e NAMESPACE=xxx]
```

### Access to the cluster

Convert the svc type to `NodePort`:

```
$ kubectl edit svc elasticsearch-master # type:NodePort
$ kubectl get svc elasticsearch-master
NAME                   TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)                         AGE
elasticsearch-master   NodePort   10.96.4.59   <none>        9200:31778/TCP,9300:31113/TCP   81m
```

Access to the cluster by curl

```
$ curl http://<your node ip>:31778/_cluster/health?pretty
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 6,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
 
