---
title: "Vertica on Kubernetes development environment"
linkTitle: "Vertica on Kubernetes development environment"
weight: 20
---

## Vertica and Kubernetes

Kubernetes is an open-source container orchestration platform that provides deployment, services, and monitoring task automation at scale. For production environments, Vertica provides a [Helm](https://helm.sh/) chart that deploys an Eon Mode database as a [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/). For details on how to download and install our Helm charts, see our official [GitHub](https://github.com/vertica/vertica-kubernetes). For details about the architecture of the Vertica on Kubernetes Helm chart, see [Containerized Vertica on Kubernetes](https://www.vertica.com/docs/10.1.x/HTML/Content/Authoring/Containers/Kubernetes/ContainerizedVerticaWithK8s.htm) in the official documentation.

## Environment Setup and Description

Download our [Developer
repository](https://github.com/vertica/vertica-kubernetes) and navigate to [DEVELOPER.md](https://github.com/vertica/vertica-kubernetes/blob/main/DEVELOPER.md). After you install the required software and scripts in the "Software Setup" section, complete the instructions in the README to configure an Eon Mode database. You must have [Docker](https://docs.docker.com/get-docker/) installed on your workstation.

[Kind (Kubernetes IN Docker)](https://kind.sigs.k8s.io/docs/user/quick-start/) uses containers to mimic Kubernetes nodes. This enables you to replicate a Vertica on Kubernetes cluster with minimal resources, including on your workstation.

The developer environment deploys a Helm chart with the release name **cluster**. This Helm instance deploys an Eon Mode database that runs a MinIO S3-compatible Eon Mode database, and TLS communication between the pods in the cluster. A Vertica database named **verticadb** is created automatically, with a default password of **password**.

## Suggested Configurations

This section details a few key configurations to help reproduce your production environment in development.

The Helm chart configuration settings in this section are added to a YAML formatted file named `overrides.yaml`. When you add new settings to `overrides.yaml`, add them below any existing settings. Update the Helm chart with the following command:

```bash
$ helm upgrade <release-name> vertica-charts/vertica -f overrides.yaml
```

For a list of configurable parameters and their defaults, see [Helm Chart Parameters](https://www.vertica.com/docs/10.1.x/HTML/Content/Authoring/Containers/Kubernetes/HelmChartParams.htm) in the official documentation.

### Adding Your License Key

The default Helm chart uses the [Vertica Community Edition](https:/www.vertica.com/download/vertica/community-edition/community-edition-10-1-0/) license. This permits youto create a cluster of 3 Vertica hosts. In some circumstances, you might want to add pods to your development cluster to mimic production scenarios. To add more than 3 pods, you must add your Vertica license key to your deployment.

To add a key, you must first create a secret that stores the location of your Vertica license. The following command creates a secret named
*vertica-license*:

```bash
$ kubectl create secet generic vertica-license --from-file=license.dat=*/path/to/license.dat
```

Add the following to your `overrides.yaml` configuration file

```
db:  
 licenseSecret:  
   name: vertica-license
```

### CPU and Memory Resources

By default, the Helm chart's default configuration requests CPU and memory resources that are likely much more than your workstation can make available. If you do not change the defaults, your computer will not have enough resources to restart failed or rescheduled pods.

The following addition to your `overrides.yaml` file requires 1 Gigabyte of RAM and 1 CPU for each pod. It also limits each pod's memory and CPU usage when there are other objects that require the host node's resources.

```
subclusters: 
 defaultsubcluster:  
   resources: 
    requests:  
       memory: "1Gi"  
       cpu: 1  
     limits:  
       memory: "1Gi"  
       cpu: 1
```

## Getting Started

The [Vertica Administration Tools](https://www.vertica.com/docs/10.1.x/HTML/Content/Authoring/AdministratorsGuide/AdminTools/AdministrationToolsReference.htm) is helpful when interacting with your cluster. The [admintool commands](https://www.vertica.com/docs/10.1.x/HTML/Content/Authoring/AdministratorsGuide/AdminTools/WritingAdministrationToolsScripts.htm) provide command line tools to create a database, add and remove Vertica cluster nodes, and [re_ip your cluster](https://www.vertica.com/docs/10.1.x/HTML/Content/Authoring/AdministratorsGuide/ManageNodes/ReMapIPs/RestartNodeNewHostIPs.htm) automatically whether the cluster is UP or DOWN. Using the [kubectl command line tool](https://kubernetes.io/docs/referenc/kubectl/overview/) and [Helm commands](https://helm.sh/docs/) is required knowledge to interact with your cluster, but getting started is esy. Below are a few common commands

### Pod Information
Use the -o wide option on the `kubectl get pods` and othe `get` kubectl commands to get a detailed view of the pods, including their IP addresses

```bash
$ kubectl get pods -o wide  
NAME                                                    READY   STATUS      RESTARTS   AGE    IP            NODE           NOMINATED NODE   READINESS GATES cluster-vertica-defaultsubcluster-0                     1/1     Running     4          3d7h   10.20.30.40   kafka-worker   <none>           <none>  
cluster-vertica-defaultsubcluster-1                     1/1     Running     4          3d7h   10.20.30.41   kafka-worker   <none>           <none>  
cluster-vertica-defaultsubcluster-2                     1/1     Running     4          3d7h   10.20.30.42   kafka-worker   <none>           <none>  
...
```

### Available Services

To view the available services:

```bash
$ kubectl get svc  
NAME                                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE  
cluster-vertica-defaultsubcluster      ClusterIP   10.96.75.102   <none>        5433/TCP,5444/TCP   2d22h  
cluster-vertica-defaultsubcluster-hl   ClusterIP   None           <none>        22/TCP              2d22h  
kubernetes                             ClusterIP   10.96.0.1      <none>        443/TCP             2d22h  
minio                                  ClusterIP   10.96.17.49    <none>        80/TCP              2d22h  
minio-hl                               ClusterIP   None           <none>        9000/TCP            2d22h  
octopus-1619456048                     ClusterIP   10.96.156.21   <none>        443/TCP             2d22h
```

### Current User-Supplied Values

To view the custom configurations currently deployed to your cluster:

```bash
$ helm get values <release-name>  
USER-SUPPLIED VALUES:  
db:  
  licenseSecret:  
    name: vertica-license  
image:  
  server:  
    tag: default-1  
subclusters:  
  defaultsubcluster:  
    resources:  
      limits:  
        cpu: 2  
        memory: 1Gi  
      requests:  
        cpu: 1  
        memory: 1Gi
```

### Access a Shell in a Pod

After you use `kubectl get pods`, use the following command to access a shell within one of the pods:

```bash
$ kubectl exec -it <pod-name> -- /bin/bash
```

## Restarting Your Database After Shutdown
When you shutdown your workstation, the containers in your kind environment stop, and your Vertica database is shut down. When the containers are restarted, new pods are scheduled. Because the new pods are rescheduled when your database is down, Vertica is not aware of their IP addresses.

When your cluster is UP and a pod fails or is terminated, Vertica uses the re-ip script to [re-map the new pod's IP address](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/ManageNodes/ReMapIPs/RestartNodeNwHostIPs.htm) to your Vertica database. In this situation, you must manually re-ip your cluster.

### Restart the Container

Use `docker container ls` the `-a` with flag to list stopped containers:

```bash
$ docker container ls -a  
CONTAINER ID   IMAGE                  COMMAND                  CREATED      STATUS                        PORTS                     NAMES  
c08a876179bd   kindest/node:v1.19.3   "/usr/local/bin/entr…"   3 days ago   Exited (255) 14 minutes ago                             cluster-worker  
7eb369de0fb6   kindest/node:v1.19.3   "/usr/local/bin/entr…"   3 days ago   Exited (255) 14 minutes ago   0.0.0.0:34893->6443/tcp   cluster-control-plane
```

Start the containers:

```bash
$ docker start c08a876 7eb369  
c08a876  
7eb369
```

### Verify the Remappings

Use admintools with the `list_allnodes` option to verify that the nodes were successfully added to the database:

```bash
$ admintools -t list_allnodes  
 Node                 | Host       | State | Version                 | DB  
----------------------+------------+-------+-------------------------+-----------  
 v_verticadb_node0001 | 10.244.1.2 | DOWN  | vertica-10.1.1          |verticadb  
 v_verticadb_node0002 | 10.244.1.4 | DOWN  | vertica-10.1.1          | verticadb  
 v_verticadb_node0003 | 10.244.1.5 | DOWN  | vertica-10.1.1          | verticadb
```

Start your database. After the database starts, verify the Vertica hosts' **State** is **UP**:

```bash
$ admintools -t start_db -d verticadb
$ admintools -t list_allnodes  
 Node                 | Host       | State | Version                | DB  
----------------------+------------+-------+-------------------------+-----------  
 v_verticadb_node0001 | 10.244.1.2 |  UP   | vertica-10.1.1          | verticadb  
 v_verticadb_node0002 | 10.244.1.4 |  UP   | vertica-10.1.1          | verticadb  
 v_verticadb_node0003 | 10.244.1.5 |  UP   | vertica-10.1.1          | verticadb
```

### Sync the Catalog

After your database is up, sync the catalog among the nodes in the cluster using vsql:

```bash
$ vsql  
=> SELECT sync_catalog()
```