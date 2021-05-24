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
    - [Helm package manager](#helm-package-manager)
    - [Volumes](#volumes)
    - [StatefulSet](#statefulset)
    - [Services](#services)
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

** Course is weird

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

Ingress Controller pods:
* We also need an ingress controller pod which evaluates the routing rules and acts as an entrypoint to the cluster
* There are lots of 3rd party images which can be used for this pod (e.g. K8s NginX Ingress Controller)
* Once we install an ingress controller, we are ready to create our Ingress

***

Ingress will allow us to:
* Have just one external load balancer
* Which forwards to the ingress controller pod (which determines routing rules)
* Which is then forwarded to our internal service

Some cloud providers provide configurability of load balancers, but ingress is most powerful
for routing.

### Helm package manager

A way to package YAML files and distribute them in public/private repositories.

Helm chart = Bundle of YAML files.
* template YAML config
* values.yaml - used for config variables of the template

```bash
helm search <keyword>
helm install --values=my-override-values.yaml <chartname>
```

Structure:
* `mychart/` name of chart
  * `Chart.yaml` meta info
  * `values.yaml` values for template
  * `charts/` chart dependencies
  * `templates/` template files

### Volumes

You need external storage to ensure data security.

PersistentVolume
* Used to declare storage resource in the cluster
* Can be mapped to storage such as AzureDisk, NFS, iSCSI, etc
* Cluster-wide resource
* We can specify the storage backend (in cluster or in cloud storage)

```yaml
apiversion: v1
kind: PersistentVolume
metadata:
  name: pv-name
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  assessModes:
  - ReadWriteOnce
  storageClassName: slow
  mountOptions:
  - hard
  - nfsvers=4.0
  nfs:
    path: /dir/path/on/nfs/server
    server: nfs-server-ip-address
```

PersistentVolumeClaim
* Used to give (claim) part of a persistent volume to a pod
* Creates a mounted volumes for each pod with specified size
* Namespace-bound resource
* Attempts to find a PV which can satisfy the claim (any PV)

```yaml
apiversion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-name
spec:
  storageClassName: manual
  volumeMode: Filesystem
  assessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Pod that mounts a volume to the pod, then mounts it to the container:
```yaml
apiversion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myfrontend
    image: nginx
    volumeMounts:
    - name: mypd  # volume name
      mountPath: "/var/www/html"
  volumes:
  - name: mypd # volume name
    persistentVolumeClaim:
      claimName: pvc-name
```

Note: there is a 1-1 mapping from PVC to PV, so a 1GB PVC will need to use up an entire 1TB PV. (To confirm)

StorageClass
* Used for dynamic volume creation + allocation in the background
* Provisions PVs dynamically as a PVC claims it (using a provisioner)
* Internal provision has 'kubernetes.io/*'
* -- Probably superior version of a PV (?)

```yaml
apiversion: v1
kind: StorageClass
metadata:
  name: storage-class-name
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

Then we can claim storage using a PVC:
```yaml
apiversion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-name
spec:
  assessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: storage-class-name
```

### StatefulSet

* Each pod in a statefulset has an identifier. When a pod stops, a new pod with the exact same identifier is created to replace it.
* There is also a mechanism that decides which pod to act as the write-database, while the rest are read-replicas.
* The master database needs to syncronise with worker databases.
* When a new database pod is created, it clones from the *previous* database pod.
* It is a best practice to use data persistence for every database pod, even worker pods.
* The identifier ensures that the persistent storage for the pod gets reattached to the same pod.
* Each pod is also given a DNS name, prepended by a service name defined in the StatefulSet.

e.g.
* mysql-0, master, pv-0, mysql-0.servicename
* mysql-1, worker, pv-1, mysql-1.servicename
* mysql-2, worker, pv-2, mysql-2.servicename

Databases in Kubernetes is tricky because you need to configure remote storage, data
syncronisation and backups manually. Containers are great for stateless applications, not stateful.

### Services

* Services allow us to have a persistent IP address to access a pod/s.
* Provides load balancing

ClusterIP (default Service)
* Internal service
* Accessible by `IP:PORT`
* Uses `selectors: <LABELS>` to specify pods
* Uses `targetPort: <PORT>` to specify the port in the pod to send to
* We can also create a multi-port service which forwards port-to-port
  * `ports: - name: ... -name: ...`
  * i.e. If we have 2 containers in one pod, and we want to forward one port to one container

Headless
* Used when we want to communicate with a single specific pod (not random)
* Useful for stateful applications / databases
  * If we need to databases communicate to the master for data syncronisation
  * If we need to communicate with the master database for writing data

NodePort
* Creates a static port, on each worker node of the cluster
* Allows us to make browser requests directly to the worker node
* Allows ports 30000 - 32767
* Not considered efficient or secure
* Normally only used for testing

LoadBalancer
* Creates an external load balancer, managed by your cloud provider
* Forwards traffic to a ClusterIP internal service

For external traffic, you would normally either use:
* Ingress, which forwards to internal services
* LoadBalancer, which forwards an internal services


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



TODO:
* Volumes practice
* Ingress practice
* Finish course