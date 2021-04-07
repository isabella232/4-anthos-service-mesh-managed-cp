# Anthos Service Mesh Managed Control Plane

This guide is a cheat sheet to easily demonstrate the deployment of istio OSS on a GKE Cluster. Once done we'll migrate to Anthos Service Mesh - Managed Control Plane. Most of the steps are subject to change soon because ASM MCP is still in Public Preview. Always refer first to Google Cloud documentation to get up to date instructions : 

- [Managed Control Plane guide](https://cloud.google.com/service-mesh/docs/managed-control-plane)
- [Istio instalation guide](https://istio.io/latest/docs/setup/install/istioctl/)
- [Anthos Service Mesh Installation Guide](https://cloud.google.com/service-mesh/docs/scripted-install/gke-install)

## Pre-requisites

Please refer to official documentation : [Managed Control Plane - Prerequisites](https://cloud.google.com/service-mesh/docs/managed-control-plane#prerequisites)

Your GKE cluster must meet the following requirements:

- A machine type that has at least 4 vCPUs, such as e2-standard-4. If the machine type for your cluster doesn't have at least 4 vCPUs, change the machine type as described in [Migrating workloads to different machine types](https://cloud.google.com/kubernetes-engine/docs/tutorials/migrating-node-pool).

- The minimum number of nodes depends on your machine type. Anthos Service Mesh requires at least 8 vCPUs. If the machine type has 4 vCPUs, your cluster must have at least 2 nodes. If the machine type has 8 vCPUs, the cluster only needs 1 node. If you need to add nodes, see [Resizing a cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/resizing-a-cluster).

- By default, the script enables [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) on your cluster. Workload Identity is the recommended method of calling Google APIs. Enabling Workload Identity changes the way calls from your workloads to Google APIs are secured, as described in [Workload Identity limitations](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#limitations).

- If you are doing a new installation and plan to use Anthos Service Mesh certificate authority (Mesh CA), you can use the [environ workload identity pool](https://cloud.google.com/anthos/multicluster-management/environs#environ-enabled-components) as an alternative to GKE workload identity. To use Mesh CA with environ (Preview), you need to either follow the steps in [Registering a cluster](https://cloud.google.com/anthos/multicluster-management/connect/registering-a-cluster) before running the script or include the --enable-registration flag when you run the script to let the script register the cluster to the project that the cluster is in. For an example of running the script to use the environ workload identity pool, see [Enable Mesh CA with environ](https://cloud.google.com/service-mesh/docs/scripted-install/gke-install#enable_mesh_ca_with_environ).

## Environment setup

```sh
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} \
    --format="value(projectNumber)")
export CLUSTER_NAME=asm-mcp-demo
export CLUSTER_ZONE=us-central1-a
```

## Create a cluster

```sh
gcloud config set compute/zone ${CLUSTER_ZONE}
gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=n1-standard-4 \
    --num-nodes=4 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --release-channel=regular
```

## Run install script

Cf : `https://cloud.google.com/service-mesh/docs/scripted-install/gke-asm-onboard-1-7`

## Enable Sidecar injection

```
kubectl label namespace default istio-injection- istio.io/rev=asm-173-6 --overwrite
```

## Enable default destination rules

```
kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```


## Request routing example

- First we create a virtual service to route all the traffic to the V1

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

- Check subset definitions

```sh
kubectl get destinationrules -o yaml
```
- Restrict users to `jason`for v2

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```
## Fault injection

Before starting : 

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
```

We'll create a fault injection only for Jason

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
```

We injected a delay on rating service of 7s however we do observe that the service is ending in error. Why ? A bug is in the code, there is a hardcoded timeout of 3s

Now we will test to inject an http abort fault

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml
```

As we can see, when connected as Jason the service doesn't answer 

## Traffic Shifting

Reset the env : `kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml`

Now let's migrate 50% of the traffic to v3

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```
If we consider the service stable we can migrate all the traffic to it :

```sh
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
```

We saw here how we can support canary / green blue deployments

## Traffic mirroring

Open a term with four windows

- Create httpbin v1 and v2 deployments, service and a virtual service to route traffic to v1 only. Also create sleep pod :

```sh
kubectl create -f httpbin-v1.yaml
kubectl create -f httpbin-v2.yaml
kubectl create -f httpbin-svc.yaml
kubectl apply -f httpbin-virtualsvc.yaml
kubectl create -f sleep-deployment.yaml
```

- Send some traffic and check logs to see

```sh
#Get Sleep pod and send traffic to app
export SLEEP_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
kubectl exec "${SLEEP_POD}" -c sleep -- curl -s http://httpbin:8000/headers

#Check logs on V1
export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})
kubectl logs "$V1_POD" -c httpbin

#Check logs en V2
export V2_POD=$(kubectl get pod -l app=httpbin,version=v2 -o jsonpath={.items..metadata.name})
kubectl logs "$V2_POD" -c httpbin
```

Apply mirror rule : `kubectl apply -f httpbin-mirror-v2.yaml`

- Cleaning up :

```sh
kubectl delete virtualservice httpbin
kubectl delete destinationrule httpbin

#Shutdown the httpbin service and client:

kubectl delete deploy httpbin-v1 httpbin-v2 sleep
kubectl delete svc httpbin
```
