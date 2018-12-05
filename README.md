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
		* [Index Settings](#index-settings)
		* [Shared Roles Settings](#shared-roles-settings)
			* [Environment](#environment)
			* [Resource Limits](#resource-limits)
		* [Breakout Roles Settings](#breakout-roles-settings)
			* [Master Nodes](#master-nodes)
			* [Client Nodes](#client-nodes)
			* [Data Nodes](#data-nodes)
	* [Snapshot Configuration](#snapshot-configuration)
		* [Schedule](#schedule)
		* [NFS Storage](#nfs-storage)
	* [Curator Configuration](#curator-configuration)
		* [Schedule](#schedule)
		* [Closing Indices](#closing-indices)
		* [Deleting Indices](#deleting-indices)

# Deployment Guide

This Elasticsearch chart is designed to be deployed on EDCOP.  By default, this values chart will deploy a single master eligible pod that is suitable for small test environments.  To scale this deployment, it will be necessary to properly plan and configure a number of these settings depending on the hardware architecture as well as the amount of data that will be ingested.

More information can be found at https://github.com/sealingtech/EDCOP/blob/master/docs/storage_guide.rst

To prevent data loss, it is important to understand the procedures from this guide.

# Configuration Guide

Within this configuration guide, you will find instructions for modifying Elasticsearch's helm chart. All changes should be made in the *values.yaml* file.
Please share any bugs or feature requests via GitHub issues.

## Image Repository

By default, Elasticsearch is pulled from Elastic's official repository and the Curator is pulled from a customized image hosted on Docker's hub. If you're changing these values, make sure you include the full repository name.

```
images:
  elasticsearch: docker.elastic.co/elasticsearch/elasticsearch:6.4.2
  curator: bobrik/curator
```

## Networks

Elasticsearch only uses an overlay network to transfer data between nodes. By default, this network is named *calico*.

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

This value tells Kubernetes which hosts the statefulsets should be deployed to by using labels given to the hosts. Hosts without the defined label will not receive pods. If you're deploying Elasticsearch in shared roles mode (default), all pods will be deployed to the data label, and the client + master labels are ignored.

```
nodeSelector:
  client: ingest
  data: data
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

Elasticsearch is deployed as a statefulset spread across all of the correctly labeled nodes in a single cluster. In shared roles mode, a master will randomly be choosen from *all* of the available pods, and in breakout mode, a master will be choosen from only the master pods.

### General

By default, the Elasticsearch cluster is named 'edcop' but this is an arbitrary value that you can change. However, if you're goal is to deploy a multitenant Elasticsearch cluster, you should make sure the names are different per cluster, and each cluster should have its own namespace to isolate data. The ```maxLocalNodes``` setting is *especially* important for multitenant deployments because you will need to increase this value to match the number of clusters you plan on having. By default, only one cluster is allowed and adding any additional clusters will result in an error.

```
elasticsearchConfig:
  clustername: edcop
  maxLocalNodes: 1
```

In order to prevent permission issues, Elasticsearch is required to run as a different user + group, and that user should own the volume directory you specified above. This user must be created beforehand and should only have access to this directory/subdirectories for security purposes. Enter the ID of this user + group in the space below:

```
elasticsearchConfig:
  runAsUser: 2000
  fsGroup: 2001
```

### Index Settings

In order to better control sharding and replication of indices, you can set the number of shards each index should be divided into to spread your data across nodes. To prevent loss of those shards, you can set the ```auto_expand_replicas``` setting to create replicas of each shard that are spread across nodes. This value should start at zero and end at a number you're comfortable with. As you add nodes, Elasticsearch will automatically expand the number of shard replicas until you hit the upper limit.

For example, if you have a 2 node cluster and use the default config, Elasticsearch will expand the replicas to 1 since you have an extra node to replicate to. If you added another node with ```auto_expand_replicas``` set to 0-2, then it would expand replicas again to 2. Adding a fourth node would not add any more replicas because you would already be at the defined limit.

*For more information, please consult the [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html).*

```
elasticsearchConfig:
  index:
    number_of_shards: 5
    auto_expand_replicas: 0-2
```


### Shared Roles Settings

Shared roles mode allows all Elasticsearch instances to ingest data, store data, and be master eligible. This means every node is fully functional and there is no separation of roles. Keep in mind, shared roles mode does not allow you to place multiple Elasticsearch pods on the same host within the same cluster, because they all store data.

For small clusters (<5), we recommend you use the shared roles mode to conserve resources by having Elasticsearch perform all roles across every node. Otherwise, you'd need a pod for each role to form a working Elasticsearch cluster.

To enable shared roles mode, set ```breakoutRoles``` to ```false``` and then define how many nodes you'd like to have. Keep in mind, these nodes will be deployed to the data label you defined in the node selector section.

```
elasticsearchConfig:
  breakoutRoles: false
  noRolesSettings:
    nodes: 1
```


#### Environment

Within the environment settings, you can change the javaopts and set the maximum (Xmx) and minimum (Xms) java heap sizes. These two values should be set to the same number as a general rule of thumb. For more information on Elastisearch's configuration, please check out their [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html).

```
elasticsearchConfig:
  breakoutRoles: false
  noRolesSettings:
    env:
      javaopts: -Xms2g -Xmx2g
      jvm:
        xms: 2g
        xmx: 2g
```

#### Resource Limits

The second part of Elasticsearch's configuration allows you to limit the CPU and memory usage. The request values must be smaller than the limit values and are set low by default to accomodate VMs. Elasticsearch recommends memory to be capped at a 32GB maximum per instance, via their instructions available [here](https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html).

```
elasticsearchConfig:
breakoutRoles: false
noRolesSettings:
  ...
  resources:
    requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 2
      memory: 4Gi
```

### Breakout Roles Settings

For larger clusters (>5) with frequent queries, we recommend you breakout Elasticsearch roles into master, client, and data nodes to better control resources and dedicate nodes to certain jobs. The master pods are deployed as statefulsets, the client nodes are deployed as deployments, and the data nodes are deployed as daemonsets. Each node adheres to the labels you defined in the node selector section.

To enable breakout roles mode, set ```breakoutRoles``` to ```true``` and then proceed to define each roles' configuration.

```
elasticsearchConfig:
  breakoutRoles: true
  breakoutRolesSettings:
  ...
```

#### Master Nodes

Master nodes are in charge of managing the cluster and handling lightweight cluster operations such as shard distribution and index management. These nodes will only manage the cluster, so they require the least amount of resources. For high availabliity, there is a 3 node minimum to form a qourum. For more information, please consult the [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#master-node)

```
elasticsearchConfig:
  breakoutRoles: true
  breakoutRolesSettings:
    master:
      nodes: 3
```

#### Client Nodes

Client (ingest) nodes are in charge of ingesting and processing data from Logstash. Depending on how much data you're ingesting, you'd generally want to give these nodes more resources than the master. These nodes are not master eligible, so they do not count towards the 3 master minimum for HA. For more information, please consult the [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#node-ingest-node)

```
elasticsearchConfig:
  breakoutRoles: true
  breakoutRolesSettings:
    client:
      nodes: 2
```

#### Data Nodes

Last but not least are data nodes which are in charge of storing all of your data, as well as performing CRUD, search, and aggregations operations. These nodes are **very** resource intensive, and you should monitor them closely while being mindful of the 32GB heap size limit for a single instance. As with client nodes, these do not count towards the master requirements and you can only have one data node per host per cluster. These nodes are deployed as daemonsets, so you do not need to define you many you want - they'll be scheduled across all data labeled nodes. For more information, please consult the [official documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#data-node)

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
