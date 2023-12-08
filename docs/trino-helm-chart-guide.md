# Getting Started with Pulsar and Trino

This repository contains a collection of instructions the guide you through the process of installing Trino inside
your K8s environment and how to leverage Trino's connector for Pulsar to query data inside Pulsar using SQL.

--------------------
Prerequisites
--------------------
- kubectl (v1.16 or higher), compatible with your cluster (+/- 1 minor release from your cluster).
- Helm (v3.0.2 or higher).
- A running Kubernetes cluster at version >= 1.24 with access configured to it using kubectl and helm.

--------------------
Trino Installation Guide - Helm Chart
--------------------
The fastest way to run Trino on Kubernetes is to use the Trino Helm chart. Helm is a package manager for Kubernetes 
applications that allows for simpler installation and versioning by templating Kubernetes configuration files.

This section covers the process of installing Trino using Helm

## Step 1: Install the Trino Helm Charts
Add the Trino Helm chart repository to Helm using the following command:

```
helm repo add trino https://trinodb.github.io/charts
```

## Step 2: Create a custom Docker image that includes the Pulsar connector for Trino

Normally, the next step would be to use the helm charts to install Trino, but unfortunately the base Trino image does NOT
include the Pulsar connector. Therefore, we will have to first build an image that does.


## Step 3: Configuration
The Helm chart uses the Trino container image by default. This Docker image contains a default configuration to get 
started, and since we built on image off of this base image, we have inherited this configuration as well.

The Trino helm chart allows you to mimic a traditional deployment by supplying configuration in YAML files. When you use
your own YAML Kubernetes configuration, you only override the values you specify. The remaining properties use their 
default values.

There are two groups of configuration properties in the yaml file that must be overridden. The first group of properties
are used to specify which docker repository and image to use when launching the Trino instance. As you might have 
guessed, we will need to update these to use the Docker image we built in the previous step. So we will create a file
named `pulsar-on-trino.yaml` and add the following stanza:

```
image:
  repository: "localhost:32000"
  tag: "pulsar-on-trino:434"
```

Next, we will need to configure Trino to use the Pulsar connector by adding the following values to the 
`additionalCatalogs` property in the configuration file as shown here.

```
pulsar: |-
    connector.name=pulsar
    pulsar.web-service-url=http://<PULSAR-CLUSTER-IP>:8080
    pulsar.broker-binary-service-url=pulsar://<PULSAR-CLUSTER-IP>:6650
    pulsar.zookeeper-uri=<PULSAR-ZOOKEEPER-IP>:2181
```

These are the bare minimum properties that must be set in order for the plugin to be able to connect to a Pulsar cluster
and query data. For a complete list of the Pulsar connector specific properties, please refer to the documentation on how to 
[Configure the Pulsar Trino plugin](https://pulsar.apache.org/docs/next/sql-deployment-configurations/#configure-pulsar-trino-plugin).

Last but not least, we want to allocate additional computing resources to the pods inside the Trino cluster, including the ability to 
dynamically increase the number of worker pods based on the CPU utilization. Therefore, we will add the following stanzas
to the `pulsar-on-trino.yaml` file as well, to allow Trino to use more memory and run more demanding queries.

```
server:
  workers: 3 
  autoscaling
    enabled: true
    maxReplicas: 6
    targetCPUUtilizationPercentage: 80
coordinator:
  jvm:
    maxHeapSize: "8G"
worker:
  jvm:
    maxHeapSize: "8G"
```

After all three stanzas have been added, we will save the configuration file before we use it to deploy the Trino cluster.

## Step 4: Install Trino on the Kubernetes cluster using the Helm chart 

Now that the configuration file has been updated, the next step is to deploy Trino using the Helm charts using the 
following commands:

```
export RELEASE_NAME=trino-cluster
export TRINO_K8S_NS=trino

helm install -f pulsar-on-trino.yaml $RELEASE_NAME trino/trino --namespace $TRINO_K8S_NS --create-namespace
```

The arguments used in the helm install command are as follows:

- `-f pulsar-on-trino`: This specifies the configuration file to apply to the chart before deploying the K8s resources.

- `$RELEASE_NAME`: This is the name given to the release of the Helm chart. It can be any name you choose and is used
  to identify the specific instance of the application being deployed.

- `trino/trino`: This specifies the chart repository and the name of the chart to be installed. In
  this case, we are installing the `trino` chart from the `trino` repository.

- `--namespace`: This specifies the namespace where the Flink Operator will be deployed. A namespace provides a logical
  boundary for resources and helps isolate different components and applications within a Kubernetes cluster.

## Step 5: Verify the Installation
In order to confirm that the helm chart was deployed properly, you can run the following command. If everything is
installed correctly, you should see output similar to that shown below:

```
helm status --namespace $TRINO_K8S_NS $RELEASE_NAME

NAME: trino-cluster
LAST DEPLOYED: Fri Dec  8 11:34:09 2023
NAMESPACE: trino
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace trino -l "app=trino,release=trino-cluster,component=coordinator" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:8080
```

You can also confirm that the K8s components were deployed by checking the status of the components in the K8s
namespace using the following command: `kubectl -n $TRINO_K8S_NS get all`

The output should look similar to this:
```
kubectl -n $TRINO_K8S_NS get all
NAME                                             READY   STATUS    RESTARTS   AGE
pod/trino-cluster-worker-6f8fb5bfbc-bgbs6        1/1     Running   0          62s
pod/trino-cluster-worker-6f8fb5bfbc-rht5v        1/1     Running   0          62s
pod/trino-cluster-worker-6f8fb5bfbc-x8v5f        1/1     Running   0          62s
pod/trino-cluster-coordinator-864c5bb67f-l78fl   1/1     Running   0          62s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/trino-cluster   ClusterIP   10.152.183.210   <none>        8080/TCP   62s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/trino-cluster-coordinator   1/1     1            0           62s
deployment.apps/trino-cluster-worker        3/3     3            0           62s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/trino-cluster-coordinator-864c5bb67f   1         1         0       62s
replicaset.apps/trino-cluster-worker-6f8fb5bfbc        3         3         0       62s
```

-------------------
References
-------------------
1. [Trino on Kubernetes with Helm](https://trino.io/docs/current/installation/kubernetes.html#)
2. [Trino Helm Chart Documentation](https://github.com/trinodb/charts/blob/main/charts/trino/README.md)