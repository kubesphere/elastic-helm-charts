## Install Guide

### Table of contents
- [Preparation](#Preparation)
- [Usage](#Usage)
  - [Install](#install)
  - [Patch](#patch)
  - [Uninstall](#uninstall)
- [Access to the cluster](#access-to-the-cluster)
- [The Reference Configuration](#the-reference-configuration) 
  - [Master node](#master-node)
  - [Data node](#data-node)

### Preparation

Install helm 3.0+ and add repo:

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

### Usage

We integrated the useful `make command line`, the description is as follows:

```
default: test

PREFIX := elasticsearch
NAMESPACE ?= default
TIMEOUT := 1200s
PATCH := $(shell cat es-patch-torlence.json)
Tag := 7.11.1

# Install
# eg.  make install -e NAMESPACE=test
install:
    helm upgrade --install  $(PREFIX)-master ../../ --values master.yaml -n $(NAMESPACE) --set imageTag="$(Tag)"
	helm upgrade --install  $(PREFIX)-data ../../  --values data.yaml -n  $(NAMESPACE) --set imageTag="$(Tag)"

# Patch custom tolerences 
patch:
        kubectl patch sts elasticsearch-master --patch  '$(PATCH)' -n  $(NAMESPACE)
        kubectl patch sts elasticsearch-data --patch  '$(PATCH)' -n  $(NAMESPACE)

# Uninstall
# eg: make uninstall -e NAMESPACE=test
uninstall:
        helm del $(PREFIX)-master -n  $(NAMESPACE)
        helm del $(PREFIX)-data -n  $(NAMESPACE)
```

#### Install

```
make install [-e NAMESPACE=xxx]
```

#### Patch

Skip this step if the states of all the pods are normal.

In my case, pending status looks like below:

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

#### Uninstall

```
make uninstall [-e NAMESPACE=xxx]
```

### Access to the cluster

Convert the svc type to `NodePort`:

```
$ kubectl edit svc
$ kubectl get svc 
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
elasticsearch-data              NodePort    10.96.125.133   <none>        9200:30057/TCP,9300:31075/TCP   165m
```

Access to the cluster by curl

```
$ curl http://<your node ip>:30057/_cluster/health?pretty
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

### The reference configuration

#### Master node

```
clusterName: "elasticsearch"
nodeGroup: "master"

roles:
  master: "true"
  ingest: "false"
  data: "false"
  ml: "false"
  remote_cluster_client: "false"

replicas: 3
minimumMasterNodes: 2

esJavaOpts: "-Xmx512m -Xms512m"

resources:
  requests:
    cpu: "512m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"

```

#### Data node

```
clusterName: "elasticsearch"
nodeGroup: "data"

roles:
  master: "false"
  ingest: "true"
  data: "true"
  ml: "false"
  remote_cluster_client: "false"

# replicas 
replicas: 3
minimumMasterNodes: 2

# Starting from version 7.11, Elasticsearch automatically sizes JVM heap based on a node's roles and total memory.
# This configuration should not be modified.
esJavaOpts: null

# Adjust the limit of the Data node according to the data size, and the memory can be up to 32GI
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"

# PV storage should be adjusted to 20Gi+
volumeClaimTemplate:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 30Gi
```
