---
title: "VClusterOps "
linkTitle: "VClusterOps"
weight: 40
---

In version 24.1.0, Vertica introduces a modernized backend that uses various HTTPS endpoints and communicates with REST APIs. The new backend dramatically reduces the time it takes to set up and execute administrative commands.

Prior to this version, Vertica managed all server and containerized deployments with Admintools, a traditional CLI that accepts parameters through STDIN and outputs unstructured results to STDOUT, and requires SSH for passwordless access so that it can communicate with all the nodes in the cluster. These features and requirements are not optimal for cloud-native applications.

This modernized backend replaces admintools with the vcluster package, which abstracts REST API calls into a high-level API. This API maps to administration commands that are available in admintools, such as create database, start vertica, scaling up or down, etc. This package is open source and available on [GitHub](https://github.com/vertica/vcluster).

The vcluster Go package is integrated into our latest version of the K8s operator. This allows us to deploy Vertica in Kubernetes. In addition to the admin backend change, we have also built some new features into the operator to take advantage of this new architecture.

The following sections deploy a local Kubernetes cluster to demonstrate some client interactions with the REST API, including how to scale out a subcluster.

## Set up K8s cluster 

We first need to set up a Kubernetes cluster. There are many ways to set up a Kubernetes cluster on your local machine. For this tutorial, we will use [minikube](https://minikube.sigs.k8s.io/docs/). If you are running Linux, you can copy the commands below to your local machine. For other operating systems, refer to the minikube instructions to understand how to set that up.

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

After the program downloads, you can create the Kubernetes cluster with the following command:

```
minikube start
```

To create an EON database, we need a location to store communal access data. For testing purposes, we will store the communal data in the minikube VM itself. By default, all host paths in minikube are owned by root. We will create a directory in one of these host paths and make it world readable so that Vertica can write to it.

minikube ssh -- 'sudo mkdir -p /data/vertica && sudo chmod a+w -R /data/vertica'

## Deploy the VerticaDB operator 

After setting up minikube, the next step is to deploy the Vertica operator. A key feature in the latest release is that the operator is now cluster-scoped. This means that you can install the operator in a Kubernetes namespace, and it will watch and manage a Vertica database in any other namespace. There are a few ways to deploy the operator: using a Helm chart, operator lifecycle manager, or with a manifest. We will install it with a manifest, with the default settings. If you want more control over the deployment, the Helm chart has a wide range of options that you can configure.

Here are the commands to run. These commands install custom resource definitions (CRD) that the operator depends on:

```
minikube kubectl -- apply --server-side=true --force-conflicts -f https://github.com/vertica/vertica-kubernetes/releases/latest/download/eventtriggers.vertica.com-crd.yaml 
minikube kubectl -- apply --server-side=true --force-conflicts -f https://github.com/vertica/vertica-kubernetes/releases/latest/download/verticaautoscalers.vertica.com-crd.yaml
minikube kubectl -- apply --server-side=true --force-conflicts -f https://github.com/vertica/vertica-kubernetes/releases/latest/download/verticadbs.vertica.com-crd.yaml
```

This command deploys the operator:
```
minikube kubectl -- apply -f https://github.com/vertica/vertica-kubernetes/releases/latest/download/operator.yaml 
```

This command waits for the operator to be deployed:
```
minikube kubectl -- wait pod --namespace verticadb-operator -l control-plane=controller-manager --for=condition=Ready=True
```

## Create a Vertica cluster 

With the operator deployed, we can now create a Vertica cluster. For the sake of testing, I will create a one-node cluster. This will allow me to scale out the operator without violating the terms of the CE license that it runs.

Run this command to create a local manifest for the [VerticaDB](https://docs.vertica.com/latest/en/containerized/) custom resource (CR).

```
cat << EOF > vdb.yaml
apiVersion: vertica.com/v1
kind: VerticaDB
metadata:
  name: v
  annotations:
    vertica.com/include-uid-in-path: "true"
    vertica.com/run-nma-in-sidecar: "false"
    vertica.com/k-safety: "0" # Allows us to create a 1 node cluster for test purposes
    VERTICA_MEMDEBUG: "2"  # Required if running macOS with an arm based chip
spec:
  image: vertica/vertica-k8s:24.1.0-0-minimal
  communal:
    path: "/communal"
  subclusters:
  - name: sc1
    size: 1
    serviceType: NodePort
  volumes:
  - name: server
    hostPath:
      path: /data/vertica
  volumeMounts:
  - name: server
    mountPath: /communal
EOF
```

If you are familiar with the VerticaDB CR, you will notice that we have upgraded the API version to `vertica.com/v1`. The previous version was `vertica.com/v1beta1`. The old version is now deprecated, but you can continue to use it. However, we encourage you to migrate to the new v1 version as soon as possible.

Run this to create the above CR that will deploy Vertica:

```
minikube kubectl -- apply -f vdb.yaml
```

Downloading the image and creating the database will take a few minutes. You can run this command to wait for the process to finish. Depending on your internet connection speed, you may need to repeat this command a few times:

```
minikube kubectl -- wait vdb v --for=condition=DBInitialized=True --timeout=10m
```

That’s all there is to it. You have successfully deployed Vertica. Now, let’s take a closer look at some of the features we have added.

## Client access to the REST API 

First, let us connect to the database. All connections in Vertica are made through Service objects. By default, we create one for each subcluster. You can retrieve the Service object URLs with the following command:

```
minikube service v-sc1

|-----------|-------|-------------------|---------------------------|  
| NAMESPACE | NAME  | TARGET PORT       | URL                       |  
|-----------|-------|-------------------|---------------------------|  
| default   | v-sc1 | vertica/5433      | http://192.168.49.2:30757 | 
|           |       | vertica-http/8443 | http://192.168.49.2:31218 |
|-----------|-------|-------------------|---------------------------|  
[default v-sc1 vertica/5433 vertica-http/8443 http://192.168.49.2:30757 http://192.168.49.2:31218]
```

Note that the URL values for your cluster are different, so replace the URL values in commands for the rest of this tutorial.

A few ports are exposed by us. 5443 is the traditional Vertica client port. This is where you would connect to send SQL to Vertica through a client like vsql. Run this command to connect and show the Vertica version:

```
vsql -h 192.168.49.2 -p 30757 -U dbadmin -c "select version()"
```

The other port is 8443. This is the new HTTPS service that is embedded in Vertica. By default, Vertica generates a production-ready, self-signed certificate. You can replace this with a custom certificate, but we will use the default certificate in this tutorial. As a result, this is an insecure connection and your browser will warn you about it. To use the endpoints, you will need to accept the certificate.

### View the Swagger documentation 

Open a web browser and go to the following URL. This opens the Swagger documentation to see all of the endpoints exposed by this service:

```
https://192.168.49.2:30989/swagger/ui?urls.primaryName=server_docs
```

### Get Prometheus metrics

To see one endpoint in action, I recommend using the /v1/metrics endpoint to view the Prometheus metrics that the server publishes. You can query that endpoint with the following command:

```
curl -k -u dbadmin: https://192.168.49.2:30989/v1/metrics
```

### Scale out a subcluster

To observe REST endpoints in action for admin commands, we will create a new subcluster and then view the operator's log to see the endpoints it called. Run the following command to initiate a scale out operator to a new subcluster named sc2:

```
minikube kubectl -- patch vdb/v --type=json --patch='[{"op": "add", "path": "/spec/subclusters/-", "value": {"name": "sc2", "size": 1, "type": "secondary", "serviceType": "NodePort"}}]' 
minikube kubectl -- wait pod v-sc2-0 --for=condition=Ready=True 
```

To view the HTTP requests that the operator used, you can inspect the logs of the operator.

```
minikube kubectl -- logs -n verticadb-operator -l control-plane=controller-manager --tail=-1 | grep 'controllers.VerticaDB.AddSubcluster' 
```

## Tear down K8s cluster

To clean up the minikube cluster after you are finished with it, run the following command:

minikube delete