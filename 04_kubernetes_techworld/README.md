# Kubernetes Course

See:
* [Kubernetes Tutorial for Beginners](https://www.youtube.com/watch?v=X48VuDVv0do) by TechWorld with Nana on YouTube

Contents:
- [Kubernetes Course](#kubernetes-course)
  - [Main K8 Components](#main-k8-components)

## Main K8 Components

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

