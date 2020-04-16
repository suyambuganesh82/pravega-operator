# Pravega Operator

 [![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![GoDoc](https://godoc.org/github.com/pravega/pravega-operator?status.svg)](https://godoc.org/github.com/pravega/pravega-operator) [![Build Status](https://travis-ci.org/pravega/pravega-operator.svg?branch=master)](https://travis-ci.org/pravega/pravega-operator) [![Go Report](https://goreportcard.com/badge/github.com/pravega/pravega-operator)](https://goreportcard.com/report/github.com/pravega/pravega-operator) [![Version](https://img.shields.io/github/release/pravega/pravega-operator.svg)](https://github.com/pravega/pravega-operator/releases)

### Project status: alpha

The project is currently alpha. While no breaking API changes are currently planned, we reserve the right to address bugs and change the API before the project is declared stable.

## Table of Contents

 * [Overview](#overview)
 * [Requirements](#requirements)
 * [Quickstart](#quickstart)    
 * [Install the Operator](#install-the-operator)
 * [Deploying in Test Mode](#deploying-in-test-mode)
 * [Upgrade the Operator](#upgrade-the-operator)
 * [Install a sample Pravega Cluster](#install-a-sample-pravega-cluster)
 * [Scale a Pravega Cluster](#scale-a-pravega-cluster)
 * [Upgrade a Pravega Cluster](#upgrade-a-pravega-cluster)
 * [Uninstall the Pravega Cluster](#uninstall-the-pravega-cluster)
 * [Uninstall the Operator](#uninstall-the-operator)
 * [Upgrade the Operator](#upgrade-the-operator)
 * [Manual installation](#manual-installation)
 * [Configuration](#configuration)
 * [Development](#development)
 * [Releases](#releases)
 * [Troubleshooting](#troubleshooting)

## Overview

[Pravega](http://pravega.io) is an open source distributed storage service implementing Streams. It offers Stream as the main primitive for the foundation of reliable storage systems: *a high-performance, durable, elastic, and unlimited append-only byte stream with strict ordering and consistency*.

The Pravega Operator manages Pravega clusters deployed to Kubernetes and automates tasks related to operating a Pravega cluster.

- [x] Create and destroy a Pravega cluster
- [x] Resize cluster
- [x] Rolling upgrades

## Requirements

- Kubernetes 1.9+
- Helm 2.10+
- An existing Apache Zookeeper 3.5 cluster. This can be easily deployed using our [Zookeeper operator](https://github.com/pravega/zookeeper-operator)
- An existing Apache Bookkeeper 4.9.2 cluster. This can be easily deployed using our [BookKeeper Operator](https://github.com/pravega/bookkeeper-operator)

## Quickstart

### Install the Operator

> Note: If you are running on Google Kubernetes Engine (GKE), please [check this first](doc/development.md#installation-on-google-kubernetes-engine).

Use Helm to quickly deploy a Pravega operator with the release name `foo`.

```
$ helm install charts/pravega-operator --name foo
```

Verify that the Pravega Operator is running.

```
$ kubectl get deploy
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
foo-pravega-operator     1         1         1            1           17s
```

#### Deploying in Test Mode
 The Operator can be run in a "test" mode if we want to create pravega on minikube or on a cluster with very limited resources by  enabling `testmode: true` in `values.yaml` file. Operator running in test mode skips minimum replica requirement checks on Pravega components. "Test" mode ensures a bare minimum setup of pravega and is not recommended to be used in production environments.

### Upgrade the Operator
Pravega operator can be upgraded by modifying the image tag using
```
$ kubectl edit <operator deployment name>
```
This approach will work for upgrades upto 0.4.0

Starting from operator version 0.5 onwards, pravega operator is not handling bookies, and that will be handled by bookkeeper-operator.So while upgrading from 0.4 to 0.5, we have to tansfer the ownership of bookkeeper objects to bookkeeper operator. For this we are maintaining 2 versions namely, v1alpha1 and v1beta1 inside the crd. v1alpha1 is for older versions and v1beta1 is for newer versions. And during upgrade we are triggering conversion webhook that will change the current version to v1beta1
More details on upgrade can be  found at  https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/

For upgrading operator version from 0.4.0 to 0.5.0, we have to execute a script and can be found at this  location https://github.com/pravega/pravega-operator/blob/test-relinquishing-ownership/tools/operatorUpgrade.sh

Script will do the following operations

- Installing cert manager and secrets
- Installing webhook
- Installing version map of operator, that will mention the supported operator  versions
- Updating crd
- Update the operator Roles
- Update the operator image

#### Executing the script
git clone pravega-operator repo

Change the path to `tools` folder

Execute the script as follows
```
./operatorUpgrade.sh  <pravega-operator deployment name> <pravega-operator deployment namespace> <pravega-operator new image-repo/image-tag> <pravegacluster name>

```
Once the script is ran successfully, ensure that operator upgrade is completed and new operator pod is in running state.

Once upgrade is completed, if the pravega version installed is 0.7 and higher, segment store pods will come up with new name, as the ss pod names are changed with 0.5 operator. After the upgrade to 0.5 , bk objects will be managed by bookkeeper operator and only controller and segment stores are managed by pravega operator

#### Known Issues
 - Once the upgrade is completed, deletion of pravega cluster is hanging
 - Bookkeeper cluster deletion is hanging post upgrade.


### Install a sample Pravega cluster

#### Set up Tier 2 Storage

Pravega requires a long term storage provider known as Tier 2 storage.

Check out the available [options for Tier 2](doc/tier2.md) and how to configure it.

For demo purposes, you can quickly install a toy NFS server.

```
$ helm install stable/nfs-server-provisioner
```

And create a PVC for Tier 2 that utilizes it.

```
$ kubectl create -f ./example/pvc-tier2.yaml
```

#### Install a Pravega cluster

Use Helm to install a sample Pravega cluster with release name `bar`.

```
$ helm install charts/pravega --name bar --set zookeeperUri=[ZOOKEEPER_HOST] --set bookkeeperUri=[BOOKKEEPER_SVC] --set pravega.tier2=[TIER2_NAME]
```

where:

- **[ZOOKEEPER_HOST]** is the host or IP address of your Zookeeper deployment (e.g. `zk-client:2181`). Multiple Zookeeper URIs can be specified, use a comma-separated list and DO NOT leave any spaces in between (e.g. `zk-0:2181,zk-1:2181,zk-2:2181`).
- **[BOOKKEEPER_SVC]** is the is the name of the headless service of your Bookkeeper deployment (e.g. `pravega-bk-bookie-0.pravega-bk-bookie-headless.default.svc.cluster.local:3181,pravega-bk-bookie-1.pravega-bk-bookie-headless.default.svc.cluster.local:3181,pravega-bk-bookie-2.pravega-bk-bookie-headless.default.svc.cluster.local:3181`).
- **[TIER2_NAME]** is the Tier 2 `PersistentVolumeClaim` name. `pravega-tier2` if you created the PVC above.


Check out the [Pravega Helm Chart](charts/pravega) for more a complete list of installation parameters.

Verify that the cluster instances and its components are being created.

```
$ kubectl get PravegaCluster
NAME          VERSION   DESIRED MEMBERS   READY MEMBERS   AGE
bar-pravega   0.4.0     7                 0               25s
```

After a couple of minutes, all cluster members should become ready.

```
$ kubectl get PravegaCluster
NAME         VERSION   DESIRED MEMBERS   READY MEMBERS   AGE
bar-pravega  0.4.0     7                 7               2m
```

```
$ kubectl get all -l pravega_cluster=bar-pravega
NAME                                          READY   STATUS    RESTARTS   AGE
pod/bar-pravega-controller-64ff87fc49-kqp9k   1/1     Running   0          2m
pod/bar-pravega-segmentstore-0                1/1     Running   0          2m
pod/bar-pravega-segmentstore-1                1/1     Running   0          1m
pod/bar-pravega-segmentstore-2                1/1     Running   0          30s

NAME                                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
service/bar-pravega-controller              ClusterIP   10.23.244.3   <none>        10080/TCP,9090/TCP   2m
service/bar-pravega-segmentstore-headless   ClusterIP   None          <none>        12345/TCP            2m

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/bar-pravega-controller-64ff87fc49       1         1         1       2m

NAME                                            DESIRED   CURRENT   AGE
statefulset.apps/bar-pravega-segmentstore       3         3         2m
```

By default, a `PravegaCluster` instance is only accessible within the cluster through the Controller `ClusterIP` service. From within the Kubernetes cluster, a client can connect to Pravega at:

```
tcp://<pravega-name>-pravega-controller.<namespace>:9090
```

And the `REST` management interface is available at:

```
http://<pravega-name>-pravega-controller.<namespace>:10080/
```

Check out the [external access documentation](doc/external-access.md) if your clients need to connect to Pravega from outside Kubernetes.

### Scale a Pravega cluster

You can scale Pravega components independently by modifying their corresponding field in the Pravega resource spec. You can either `kubectl edit` the cluster or `kubectl patch` it. If you edit it, update the number of replicas for BookKeeper, Controller, and/or Segment Store and save the updated spec.

Example of patching the Pravega resource to scale the Segment Store instances to 4.

```
kubectl patch PravegaCluster <pravega-name> --type='json' -p='[{"op": "replace", "path": "/spec/pravega/segmentStoreReplicas", "value": 4}]'
```

### Upgrade a Pravega cluster

Check out the [upgrade guide](doc/upgrade-cluster.md).

### Uninstall the Pravega cluster

```
$ helm delete bar --purge
$ kubectl delete -f ./example/pvc-tier2.yaml
```

### Uninstall the Operator

> Note that the Pravega clusters managed by the Pravega operator will NOT be deleted even if the operator is uninstalled.

```
$ helm delete foo --purge
```

If you want to delete the Pravega clusters, make sure to do it before uninstalling the operator.

### Manual installation

You can also manually install/uninstall the operator and Pravega with `kubectl` commands. Check out the [manual installation](doc/manual-installation.md) document for instructions.

## Configuration

Check out the [configuration document](doc/configuration.md).

## Development

Check out the [development guide](doc/development.md).

## Releases  

The latest Pravega releases can be found on the [Github Release](https://github.com/pravega/pravega-operator/releases) project page.

## Troubleshooting

Check out the [troubleshooting document](doc/troubleshooting.md).
