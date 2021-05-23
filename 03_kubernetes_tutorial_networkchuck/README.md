# Kubernetes Tutorial

See:
* [you need to learn Kubernetes RIGHT NOW!!](https://www.youtube.com/watch?v=7bA0gTroJjw) by NetworkChuck on YouTube

Contents:
- [Kubernetes Tutorial](#kubernetes-tutorial)
  - [Kubernetes on Linode Kubernetes Engine](#kubernetes-on-linode-kubernetes-engine)
    - [LKE](#lke)
    - [KubeCTL](#kubectl)
    - [Single Pod](#single-pod)
    - [Deployment (multiple pods)](#deployment-multiple-pods)
    - [Editing deployments](#editing-deployments)
    - [Exposing Pods (network access)](#exposing-pods-network-access)
  - [Kubernetes on Minikube](#kubernetes-on-minikube)
  - [Notes](#notes)
    - [Types of k8 objects](#types-of-k8-objects)
  - [Commands](#commands)

## Kubernetes on Linode Kubernetes Engine

### LKE

We can easily create a LKE cluster:
* Name
* Region
* Kubernetes Version

And we add a fixed number of nodes to our cluster. e.g. 3 (1CPU, 50GM storage, 2GB RAM) instances.

The master node is FREE. It is pointed to by the Kubernetes API Endpoint URL.

### KubeCTL

Install kubectl.

The `kubeconfig.yml` file gives us all the information we need to connect to our cluster.

```bash
export KUBECONFIG=kubeconfig.yaml
kubectl get nodes
# YAY
```

We can also set the default kubeconfig by setting `~/.kube/config` to the contents of `kubeconfig.yaml`

### Single Pod

Containers run inside pods (pods are a single independent unit).

We can create a single pod:
```bash
kubectl run stuff --image=thenetworkchuck/nccoffee:pourover --port=80       # Create pod
kubectl get pods                # View pods
kubectl describe pods           # Shows detailed information including network info, etc
#kubectl delete pod stuff       # Delete pod
```

Every pod is assigned a private IP address. Pods will usually have one container dedicated to a task.

### Deployment (multiple pods)

We might want to deploy multiple pods (replicas=3).

Best way to do this is the yaml file: `networkchuckcoffee_deployment.yaml`
* Kind: Deployment
* Replicas 3

We can execute this using:
```bash
kubectl apply -f networkchuckcoffee_deployment.yaml
kubectl get pods
```

### Editing deployments

We can update this deployment live by editing the deployment:
```bash
kubectl edit deployments.apps networkchuckcoffee-deployment
# Use VIM editor to change number of replicas (e.g. 10)

kubectl get pods
# We should hve 10 pods running!!!
```

We can even change the image being used:
* Terminates all old pods and starts new ones!

### Exposing Pods (network access)

Our load balancer will be described with another YAML file: `networkchuckcoffee_service.yaml`
* Kind: Service
* type: LoadBalancer
* It will load balance between all pods with the label: `nccoffee`

Now lets apply this service:
```
kubectl apply -f networkchuckcoffee_service.yaml
kubectl get services
```

In the services, we can see our load balancer, **and it has an external IP!**

In Linode, we can see our service under the NodeBalancers page (it is a load balancer).

## Kubernetes on Minikube

Installation: https://minikube.sigs.k8s.io/docs/start/

MiniKube will allow us to run a Kubernetes cluster on our local machine.

```
minikube version
minikube start --nodes=3
```

It will start MiniKube using docker containers and we can access our cluster using KubeCTL
with localhost.

## Notes

### Types of k8 objects

Types of k8 objects, the 'kind' in the YAML spec:
* Pod
* DaemonSet
* Deployments
* Service

## Commands

Examination:

Purpose                             | Command/s
------------------------------------|------------------------
Get list                            | `kubectl get [pods|nodes|services|deployments.apps]`
View cluster info                   | `kubectl cluster info`
List all pods + node/IP info        | `kubectl get pods -o wide`
List all pods in a specific node    | `kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<NODE_NAME>`

Creating pods/deployments:

Purpose                             | Command/s
------------------------------------|------------------------
Create a single pod                 | `kubectl run stuff --image=<IMAGE> --port=80`
Create a deployment using YAML      | `kubectl apply -f <FILE>.yaml`
