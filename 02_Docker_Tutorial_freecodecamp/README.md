# Docker Tutorial

See:
* [Docker Tutorial for Beginners](https://www.youtube.com/watch?v=fqMOX6JJhGo) by FreeCodeCamp on YouTube
* [Docker labs](https://kodekloud.com/p/docker-labs)

Contents:
- [Docker Tutorial](#docker-tutorial)
  - [Notes](#notes)
    - [Image tags](#image-tags)
    - [Container inspection](#container-inspection)
    - [Networks](#networks)
    - [Storage / Volume mapping](#storage--volume-mapping)
    - [Docker compose](#docker-compose)
  - [Dockerize a Flask App (!)](#dockerize-a-flask-app-)


## Notes

### Image tags

`:<version>` is called a 'tag' in the image name. Supported tags are described on the Docker website.

### Container inspection

Use `docker inspect <container-name>` to view important details.

We can see:
* Environment variables
* IP address (accessible by docker network or host machine) (`grep "IPAddress"`)
* Mac address
* Port bindings (from containerhost:port to localhost:port)
* Exposed ports (from containerhost:port to containerIP:port)
* Docker Network
* ...

### Networks

By default containers are in the `bridge` network.

Networks are important so that we can isolate or group multiple running instances with a host.

If you set `--network host`:
* The **container localhost** will be the same as the **host localhost**.
* Can access a web application **without** needing to expose any ports or do port-mappings

If you set `--network none`:
* All containers will be isolated

A container can access another container on the same network using IP address, or container-name.

Container-name is best-practice. Docker has a built-in DNS for container names.

 e.g.
```python
import docker
import mysql.connector

client = docker.DockerClient()
container = client.containers.get("magical_meitner")
ip_addr = container.attrs['NetworkSettings']['IPAddress']

mydb = mysql.connector.connect(ip_addr)

```

* Container-name is the right way (i.e.)

### Storage / Volume mapping

Use `-v` or `--mount` flag to mount local folder to the container. (`-v` is old-style)

Commands:
* `docker run -v <host-directory>:<container-directory> <container-name>`
  * Autocreates the volume or directory.
* `docker run --mount type=bind,source=<host-directory>,target=<container-directory> <container-name>`
  * Must create the volume or directory first!!!
* (can also mount docker-volumes)

e.g.
* `docker run -itv $(pwd)/stuff:/stuff ubuntu`
* `docker run -it --mount type=bind,source=$(pwd)/stuff,target=/stuff ubuntu`

The `--mount` command is more modern.

### Docker compose

https://youtu.be/fqMOX6JJhGo?t=4651

## Dockerize a Flask App (!)

Summary:
* Flask app with on `0.0.0.0:8080`
* Port mapped from `(container):8080` to `localhost:5000`
* Container port also exposed on `containerIP:8080`

`flask_app/` contains a basic flask application which relies on an `APP_COLOR` environment variable.
* We need to ensure that we have exposed `0.0.0.0` in our container
* We can access the website via `<CONTAINER-IP-ADDRESS>:<CONTAINER-PORT>` (shown in the stdout via flask)
* Or `localhost:<LOCAL-PORT>` if we using port mapping`-p <LOCAL-PORT>:<CONTAINER-PORT>`

```
docker build . -t my-flask-app
docker run -itp 5000:8080 -e APP_COLOR=blue my-flask-app
```