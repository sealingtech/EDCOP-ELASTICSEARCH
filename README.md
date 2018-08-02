# EDCOP Elasticsearch Guide

Table of Contents
-----------------
 
* [Configuration Guide](#configuration-guide)
	* [Image Repository](#image-repository)
	* [Networks](#networks)
	* [Persistent Volume Storage](#persistent-volume-storage)
	* [Node Selector](#node-selector)
	* [Elasticsearch Configuration](#elasticsearch-configuration)
		* [General](#general)
		* [Master Settings](#master-settings)
		* [Environment](#environment)
		* [Resource Limits](#resource-limits)
	* [Snapshot Configuration](#snapshot-configuration)
		* [Schedule](#schedule)
		* [NFS Storage](#nfs-storage)
	* [Curator Configuration](#curator-configuration)
		* [Schedule](#schedule)
		* [Closing Indices](#closing-indices)
		* [Deleting Indices](#deleting-indices)
		
# Deployment Guide

This Elasticsearch chart is designed to be deployed on EDCOP.  By default, this values chart will deploy a single master pod that is suitable for small test environments.  To scale this deploy it will be necessary to properly plan and configure a number of these settings depending on the hardware architecture as well as the amount of data that will be ingested.

More information can be found at https://github.com/sealingtech/EDCOP/blob/master/docs/storage_guide.rst

To prevent data loss, it is important to understand the procedures from this guide.
	
# Configuration Guide

Within this configuration guide, you will find instructions for modifying Elasticsearch's helm chart. All changes should be made in the *values.yaml* file.
Please share any bugs or feature requests via GitHub issues.
 
## Image Repository

By default, Elasticsearch is pulled from Elastic's official repository and the Curator is pulled from a customized image hosted on Docker's hub. If you're changing these values, make sure you include the full repository name.
 
```
images:
  elasticsearch: docker.elastic.co/elasticsearch/elasticsearch:6.3.0
  curator: bobrik/curator
```
 
## Networks

Elasticsearch only uses an overlay network to transfer data between nodes. By default, this interface is named *calico*. 

```
networks:
  overlay: calico
```
 
To find the names of your networks, use the following command:
 
```
# kubectl get networks
NAME		AGE
calico		1d
passive		1d
inline-1	1d
inline-2	1d
```

## Persistent Volume Storage

These values tell Kubernetes where Elasticsearch's index data should be stored on the host for persistent storage. By default, this value is set to */EDCOP/bulk/esdata* but should be changed according to your logical volume setup. 

```
volumes:
  data: /EDCOP/bulk/esdata
```
	  
## Node Selector

This value tells Kubernetes which hosts the statefulsets should be deployed to by using labels given to the hosts. Hosts without the defined label will not receive pods. Client pods will only deploy to nodes labeled 'data=true' while the master pod will only deploy to the node labeled 'infrastructure=true'. 
 
```
nodeSelector:
  client: data
  master: infrastructure
```
 
To find out what labels your hosts have, please use the following:
```
# kubectl get nodes --show-labels
NAME		STATUS		ROLES		AGE		VERSION		LABELS
master 		Ready		master		1d		v1.10.0		...,infrastructure=true
minion-1	Ready		<none>		1d		v1.10.0		...,data=true
minion-2	Ready		<none>		1d		v1.10.0		...,data=true
```

## Elasticsearch Configuration

Elasticsearch is deployed as a statefulset spread across all of the worker nodes in a single cluster. These instances point to the master deployment that should be on your Kubernetes master node. 

### General

In order to prevent permission issues, Elasticsearch is required to run as a different user and that user should own the volume directory you specified above. This user must be created beforehand and should only have access to this directory/subdirectories for security purposes. Enter the UID of this user in the space below:

```
elasticsearchConfig:
  runAsUser: 2000
```

Since Elasticsearch's workers are run as Statefulsets, you need to specify how many instances you want to maintain. By default, this value is 3, but should be scaled to include the number of worker nodes you have. Do not include the master as one instance because it is deployed in a seperate deployment that only runs on the master. 

```
elasticsearchConfig:
  workerNodes: 3
```

### Master Settings

Since the Elasticsearch master nodes are also run as Statefulsets, you will need to specify how many master nodes you want. The default value is 1, but should be scaled in the same manner as the workers. Again, do not include the number of workers in this number

```
elasticsearchConfig:
  master:
    nodes: 1
```

In order to accomodate master nodes that do not have a lot of storage, you can prevent Elasticsearch from storing data on the master server, meaning all of the data will be stored on the workers. Setting this value to *false* will prevent the master from storing data as well as disabling the hostpath setting for master nodes. 

```
elasticsearchConfig:
  master:
    data: true
```

### Environment

Within the environment settings, you can change the cluster name and set the maximum (Xmx) and minimum (Xms) java heap sizes. These two values should be set to the same number as a general rule of thumb. For more information on Elastisearch's configuration, please check out their [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html).

```
elasticsearchConfig:
  env:
    clustername: edcop
    javaopts: -Xms16g -Xmx16g
```

### Resource Limits

The second part of Elasticsearch's configuration allows you to limit the CPU and memory usage. The request values must be smaller than the limit values and are set low by default to accomodate VMs. Elasticsearch recommends memory to be capped at a 32GB maximum per their instructions available [here](https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html).

```
elasticsearchConfig:
  ...
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 12
    memory: 32Gi
```
## Snapshot Configuration

If you want to backup your indices to Kubernetes NFS storage, you can enable periodic Elasticsearch snapshots run as cronjobs within Kubernetes. It is highly recommended that you take snapshots of your indices in case of unexpected failures or errors.

### Schedule

As mentioned before, the Snapshots are run as Kubernetes Cron Jobs, which use the Cron format as described [here](http://www.nncron.ru/help/EN/working/cron-format.htm). The default setting runs once per day, but you might want to increase this in production. 

```
snapshotConfig:
  cronjob_schedule: "1 0 * * *"
```

### NFS Storage

In order to store the snapshots, we use Kubernetes based NFS volumes mounted into the Elasticsearch containers. Luckily, EDCOP already has an automatic NFS provisioner built in, so we just need to define the amount of storage we want to give to snapshots. The default is 10GB to accomodate test VMs.

```
snapshotConfig:
...
pvc:
  storage: 10Gi
```
 

## Curator Configuration

The EDCOP Elasticsearch Helm chart comes bundled with Elasticsearch's Curator and is run periodically as a Kubernetes Cron Job. You can disable the Curator entirely by setting the *enabled* value to false, but it is recommended that you periodically clean up old indices to save disk space.

### Schedule

Just like Snapshots, the Curator is run as a Kubernetes Cron Job, which uses the Cron format as described [here](http://www.nncron.ru/help/EN/working/cron-format.htm). The default setting runs just after midnight each day.

```
curatorConfig:
  cronjob_schedule: "1 0 * * *"
```

### Closing Indices

The first action the Curator will perform closes indices that are older than the specified amount of time. The default time is set to 7 days and you can always disable this action by setting *disable_action* to True. You can find more information on the close action [here](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/close.html).

```
curatorConfig:
  ...
  actions:
    close:
      disable_action: False
      unit: days
      unit_count: 7
```

### Deleting Indices

The second action the Curator will perform deletes indices that are older than the specified amount of time. The default time for deletion is 14 days and you can always disable this action as well by setting *disable_action* to True. For more information on deleting indices, please click [here](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/delete_indices.html). 

```
curatorConfig:
  ...
  actions:
    delete_indices:
      disable_action: False
      unit: days
      unit_count: 14
```
