# Postgres Persistent Volume Demo

See:
* [Persistent Volumes on Kubernetes for beginners](https://www.youtube.com/watch?v=ZxC6FwEc9WQ&)
* [marcel-dempers/docker-development-youtube-series ](https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/kubernetes)

## Volumes with docker

```bash
# Start Postgres database and enter shell
docker run -d --rm -e POSTGRES_DB=postgresdb -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=admin123 --name=mydb postgres:10.4
docker exec -it mydb bash

# Open database
psql --username=admin postgresdb

# Create a table, then list the tables, then exit the container
CREATE TABLE COMPANY(
ID      INT PRIMARY KEY NOT NULL,
NAME    TEXT            NOT NULL,
AGE     INT             NOT NULL,
ADDRESS CHAR(50),
SALARY  REAL
);
\dt
\q
exit

# Delete
docker top mydb
# OUR DATA IS GONE!
```

Instead, we can use docker volumes
```bash
# Create volume
docker volume create postgres

# Create container (with mounted volume) and open shell
docker run -d --rm \
  -e POSTGRES_DB=postgresdb \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=admin123 \
  --name=mydb \
  -v postgres:/var/lib/postgresql/data \
  postgres:10.4
docker exec -it mydb bash

# Open database
psql --username=admin postgresdb

# Create a table, then list the tables, then exit the container
CREATE TABLE COMPANY_D(
ID      INT PRIMARY KEY NOT NULL,
NAME    TEXT            NOT NULL,
AGE     INT             NOT NULL,
ADDRESS CHAR(50),
SALARY  REAL
);
\dt
\q
exit
docker stop mydb

# Create a new container and see if it persists
docker run -d --rm \
  -e POSTGRES_DB=postgresdb \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=admin123 \
  --name=mydb \
  -v postgres:/var/lib/postgresql/data \
  postgres:10.4
docker exec -it mydb bash
psql --username=admin postgresdb
\dt
\q
exit
docker stop mydb
```

!!! DATA IS NOT SYNCED BETWEEN CONTAINERS! Having multiple database containers will cause overwrites.

To do synced data we might need to create a docker service with for an NFS drive or something.

## Kubernetes persistent volumes

Use `kubectl get storageclasses.storage.k8s.io` to get the available storage classes in your cluster.

```bash
kubectl create ns postgres
kubectl apply -n postgres -f ./postgres-no-pv.yaml
kubectl -n postgres get pods
kubectl -n postgres exec -it postgres-0 bash
```