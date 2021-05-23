# Kubernetes Course

See:
* [Kubernetes Tutorial for Beginners](https://www.youtube.com/watch?v=X48VuDVv0do) by TechWorld with Nana on YouTube

Contents:
- [Kubernetes Course](#kubernetes-course)
  - [Notes](#notes)
    - [Main K8 Components](#main-k8-components)
    - [K8 Architecture](#k8-architecture)
    - [Basic usage](#basic-usage)
    - [K8 YAML configuration file spec](#k8-yaml-configuration-file-spec)
    - [Accessing ConfigMap & Secrets](#accessing-configmap--secrets)
    - [Internal/External Services](#internalexternal-services)
    - [Namespaces](#namespaces)
    - [Ingress](#ingress)
  - [Demos](#demos)
  - [Commands](#commands)

## Notes

### Main K8 Components

Node:
* A machine running Kubernetes

Pod:
* Smallest unit of K8s
* Abstraction over container
* Usually 1 application per pod
* Each pod gets its own internal IP address

Service:
* Allows for permanent IP address that can be attached to each pod
* External service - For frontend/webservers
* Internal service - for backend/databases

Ingress:
* Forwards to the service for certain cases

Configmap:
* External configuration for your application
* Like database-url
* Or other environment variables

Secret:
* Similar to configmap, but is used to store secret data securely
* Not in plain-text, base64 encoded
* Like passwords, certificates, tokens, etc
* (Note: built-in security mechanism not enabled by default)

Volumes:
* Attaches physical storage to a pod

Deployment:
* Blueprint for pods

StatefulSet:
* Used for databases!!! (instead of deployments)
* Databases read and write to an external storage volume
* StatefulSet is used to ensure that database reads/writes are sycronised, so there are no inconsistencies between the pods.
* (Note: You cannot deploy multiple databases)
* (Note: StatefulSet can be tedious, so it is very common to host the entire database externally)

### K8 Architecture

3 processes must be installed on each node:
* Kubelet
  * Interacts with container and node
  * Starts the pod with a container inside
* Container runtime
  * Used to run the containers
* Kube proxy
  * Used for forwardng requests between services and pods
  * Can intelligently route to pods within the same node

4 processes run on master nodes:
* API server (cluster gateway + auth)
* Scheduler (interacts with kubelets on nodes)
* Controller manager (detects state changes and informs scheduler)
* etcd (cluster brain, key-value store information, updated based on cluster state)

### Basic usage

Create an Nginx deployment (not using a yaml file):
`kubectl create deployment nginx-depl --image=nginx`

Edit the deployment (by default it has just 1 replica and an app:name lavel):
`kubectl edit deployments.apps nginx-depl`

View the logs of the pod:
`kubectl logs nginx-depl-5c8bf76b5b-cjhzc`

Open an interactive terminal in a pod/container (very similar to docker!):
`kubectl exec -it nginx-depl-5c8bf76b5b-wflwj -- bin/bash`

Create or update a deployment (using a yaml file):
`kubectl apply -f nginx-deployment.yaml`

We can also delete a deployment using the configuration file instead of deployment name.

### K8 YAML configuration file spec

Metadata
* Different for each type/kind of deployment
  * deployments: replicas, selector, template
  * services: selector, port

Status
* Automatically injected by Kubernetes

Note that all status information is stored in etcd.

Deployment:
* spec > template:
  * The blueprint for the pod for a deployment
* metadata > labels > key:value
  * The labels for the deployment
* spec > template > metadata > labels > key:value
  * The labels for the pods (? same as deployment label ?)

Service:
* spec > selector > matchLabels:
  * Needs to point to either the pod template label or the deployment label
* spec > ports > -protocol:TCP,port:80,targetPort:8080
  * Target port is the port on the worker pods
  * Port is the port on the service

After deploying a YAML file, we can view the **FULL** status/config, pulled directly from etcd:
```bash
# Get config (with status info added) and pipe to a yaml file
kubectl get deployments.apps nginx-depl -o yaml > nginx-deployment-result.yaml
```

### Accessing ConfigMap & Secrets

```yaml
env:

# SECRET
- name: ME_CONFIG_MONGODB_ADMINPASSWORD # local environment variable
  valueFrom:
    secretKeyRef:
      name: mongodb-secret              # Secret name
      key: mongo-root-password          # item name

# CONFIGMAP
- name: ME_CONFIG_MONGODB_SERVER        # local environment variable
  valueFrom:
    configMapKeyRef:
      name: mongodb-configmap           # ConfigMap name
      key: database_url                 # item name
```

Secrets:
* Use `echo -n 'username' | base64` to base64 encode your secrets before putting them in your YAML

DO NOT COMMIT SECRETS.YML TO GITHUB
* Consider using "Sealed Secrets for Kubernetes" to make it safe to store publicly.
* Or don't store them at all.

### Internal/External Services

Change an internal service (ClusterIP) to an external service (LoadBalancer) by changing:
* `spec: type: LoadBalancer`
* And possibly also: `spec: ports: nodePort: 30000`

In an external service, the packet should be forwarded to:
1. nodePort (port exposed to external traffic)
2. port (port accessible internally to the cluster)
3. targetPort (port running the Mongo Express web application) - not sure why this isn't happening - maybe it is due to Linode?

### Namespaces

A way to organise/seperate your resources. Like a cluster inside of your cluster.
```
kubectl get namespaces
```

4 default namespaces:
* kube-system
  * Not meant for user access
  * System & master processes etc
* kube-public
  * Publicly accessible data even without authentication
  * `kubectl cluster-info`
* kube-node-lease
  * Node heartbeat, each node has an associated lease object
  * Determines availability of a node
* Default
  * Where our resources are created

We can create namespaces using a configuration file. e.g.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-configmap
  namespace: my-namespace
data:
  db_url: mysql-service.database
```

When should use one?
* When our default namespace gets fill and it gets hard to manage/filter through the resources.
  * We can have a namespace for database, monitoring, logging, elastic stack, nginx, etc
* Large projects
* Large/numerous teams
* Staging/development environments
* Blue/green deployment

Cons:
* Limited access
* ConfigMaps and secrets need to be replicated over multiple namespaces

Services can still be accessed between namespaces! `<SERVICE>.<NAMESPACE>` => `mysql-service.database`

We can use a different namespace by using the `-n` flag, or installing kubens to change the default namespace.

### Ingress

The Problem with Load Balancers:
* Only 1 LB per service
* Each LB needs the cloud-provider to create an external machine which routes to the service
* Can get expensive if we have many different services

In this case, we would not make an external service, instead an ingress which directs to the
internal service. `ingress > internal service > pod`

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: myapp-ingress
spec:

  tls: # TLS certificate!
  - hosts:
    - myapp.com
    secretName: myapp-secret-tls

  rules: # Routing rules
  - host: myapp.com
    http:
      paths:
      - backend: # (No path)
        serviceName: myapp-internal-service
        servicePort: 8080
      - path: /analytics
        backend:
        serviceName: analytics-service
        servicePort: 8080
```

We define the exact rules for subdomains/paths and the service that it directs to.

We also need an ingress controller pod which evaluates the routing rules and acts as an entrypoint
to the cluster. (There are lots of 3rd party implementations):
`ingress controller > ingress > internal service > pod`


## Demos

[Demo: MongoDB & MongoExpress](mongo/README.md)
* **Deployment** for MongoDB
* **Secret** for username, password
* **Internal Service** for accessing the database
* **ConfigMap** to obtain central configuration (the DNS name of the database service)
* **Deployment** for MongoExpress
* **External Service** to expose the Mongo Express server publicly

## Commands

Purpose                             | Command/s
------------------------------------|------------------------
List all kubernetes objects         | `kubectl get all`
View pod logs                       | `kubectl logs <POD>`
Open interactive terminal in pod    | `kubectl exec -it <POD> -- bin/bash`
Get full deployment config/status   | `kubectl get deployments.apps nginx-depl -o yaml`
Test connectivity                   | `nc -vz <URL> <PORT>` or `ping <URL>:<PORT>` (shows DNS lookup result)

