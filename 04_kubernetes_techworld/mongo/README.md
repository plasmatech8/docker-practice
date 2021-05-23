# Demo: MongoDB & MongoExpress

Contents:
- [Demo: MongoDB & MongoExpress](#demo-mongodb--mongoexpress)
  - [Overview](#overview)
  - [MongoDB](#mongodb)
  - [MongoDB-Secret](#mongodb-secret)
  - [MongoDB internal service](#mongodb-internal-service)
  - [MongoExpress](#mongoexpress)
  - [MongoExpress external service](#mongoexpress-external-service)

## Overview

We want:
* MongoDB pod
  * Accessed via internal service
* MongoExpress pod
  * Needs
    * Database URL / service DNS name (configmap)
    * Database user (secret)
    * Database password (secret)
    * These will be passed as environment variables in the deployment.yaml
  * Accessed via external service

Browser > External Service > MongoExpress > InternalService > MongoDB

## MongoDB

See [mongo.yaml](mongo/mongo.yaml)

We need to look at the documentation for the docker container for
[mongo](https://hub.docker.com/_/mongo) so we know how to use the container:
* Database port: 27017
* MONGO_INITDB_ROOT_USERNAME
* MONGO_INITDB_ROOT_PASSWORD

## MongoDB-Secret

See [mongo-secret.yaml](mongo/mongo-secret.yaml)

We will write the username and password in the secret YAML files:
* We will base64 encode the text so it is not in plain-text
  * `echo -n 'username' | base64`
  * DO NOT COMMIT SECRETS TO GITHUB
    * Consider using "Sealed Secrets for Kubernetes" to make it safe to store publicly
```yaml
data:
  mongo-root-username: dXNlcm5hbWU=
  mongo-root-password: cGFzc3dvcmQ=
```

Deploy the secrets:
```
kubectl apply -f mongo-secret.yaml
kubectl get secret
```

In the deployment spec we will add the variables to the file:
```yaml
# ...
spec:
  template:
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
```

Now we can deploy our database:
```bash
kubectl apply -f mongo.yaml
```

## MongoDB internal service

We can put multiple documents in one YAML file using `DOC1 \n---\n DOC2`

We will set:
* selector to our database
* port to our service port
* targetPort to our database port
```yaml
# ...
---
# ...
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

Update: `kubectl apply -f mongo.yaml`

Now we have an internal service for the database.

## MongoExpress

See [mongo-express.yaml](mongo/mongo-express.yaml)

Pod for [mongo-express](https://hub.docker.com/_/mongo-express):
* Container starts application on port 8081
* ME_CONFIG_MONGODB_ADMINUSERNAME - username
* ME_CONFIG_MONGODB_ADMINPASSWORD - password
* ME_CONFIG_MONGODB_SERVER - database endpoint

We will store `ME_CONFIG_MONGODB_SERVER` in a configmap, so we avoid using an IP address which
could potentially change.

See [mongo-configmap.yaml](mongo/mongo-configmap.yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service
```
This will provide us a URL which points to the service named 'mongodb-service'.

Then we can reference this as an environment variable in the express app:
```yaml
valueFrom:
  configMapKeyRef:
    name: mongodb-configmap
    key: database_url
```
Then apply:
```bash
kubectl apply -f mongo-configmap.yaml
```

> * Use secrets for tokens, usernames, passwords, etc
> * Use configmaps for getting the URL address of a service
>
> Note: Only services get DNS names.

Here we are using the configmap to get the URL address of a service.

Configmaps aren't super useful if our application is really simple, but if there are multiple
references to the same configuration, then it is very useful.

e.g.
* We change the name of our mongodb-service (which is also used for DNS), and we don't want to change every occurance of it in every file.
* Instead, we only need to update our configmap.

## MongoExpress external service

We will add another document to `mongo-express.yaml`.

To make an external service we add:
* `spec: type: LoadBalancer`
  * (Which is a bad name since internal services are also load balancers)
  * Assigns our service an external IP address and accepts external requests
* `spec: ports: nodePort: 30000`
  * The port exposed/used to access the service from the outside
  * Restricted to 30000-32767

> Internal service is default

Now we can go to the external IP address of the service in our browser!!!
* Make sure you use `http:<EXTERNAL-IP>:8081`

> In a service, the packet should be forwarded to:
> 1. nodePort (port exposed to external traffic)
> 2. port (port accessible internally to the cluster)
> 3. targetPort (port running the Mongo Express web application)