# Docker Tutorial

See
* [Docker Tutorial for Beginners [2021]](https://www.youtube.com/watch?v=pTFZFxd4hOI) By Programming with Mosh
* [Learn Docker in 7 Easy Steps - Full Beginner's Tutorial](https://www.youtube.com/watch?v=gAkwW2tuIqE) By Fireship

Contents:
- [Docker Tutorial](#docker-tutorial)
  - [Intro](#intro)
  - [Setup an image (dockerfile)](#setup-an-image-dockerfile)
  - [Build an image (docker build)](#build-an-image-docker-build)
  - [Viewing images (docker image ls)](#viewing-images-docker-image-ls)
  - [Running images (docker run)](#running-images-docker-run)
  - [Shared volumes (docker volume)](#shared-volumes-docker-volume)
  - [Debugging (docker exec)](#debugging-docker-exec)
  - [Compose (docker compose)](#compose-docker-compose)
  - [Command List](#command-list)
    - [Image commands:](#image-commands)
    - [Container commands:](#container-commands)
    - [Debugging commands](#debugging-commands)
    - [Composition/other commands](#compositionother-commands)
    - [Notes](#notes)

## Intro

It is good to develop with containers. :D

## Setup an image (dockerfile)

We will create a dockerfile to create the docker image.

```dockerfile
# Using the node image, on the alpine linux distro
FROM node:alpine

# Copy all files into /app in the container
COPY . /app

# Set CWD to /app
WORKDIR /app

# Run the application
CMD node app.js
```

Docker statements:
* `EXPOSE 8080` to expose a port to the outside
* `RUN <command>` used for a build-image step. Can have multiple.
* `CMD <command>` default command which can be overidden by the docker run command. Only 1 available
* `ENTRYPOINT <command>` CMD is fed into the entrypoint. i.e. `/bin/sh -c bash` - default is `/bin/sh -c`
* (commands can be done in `shell` or `["exec"]` form)
* (CMD and ENTRYPOINT is useful when you want to make an image specific to a command or multipurpose)

Notes:
* Ctrl+click on the image name to go to the docker website to view images.
* Create a `.dockerignore` file to make docker ignore certain files from the build.
* Images will automatically be `docker pull`'ed when needed when we `docker run` or `docker build`

## Build an image (docker build)

Then we can run the command:
```
docker build -t hello-docker .
```
* `-t <name>` is the name/path of the image. Consider creating a username on Dockerhub and including your username and version (i.e. `plasmatech8/myapp:1.0`).
* `.` is the directory which contains the dockerfile


## Viewing images (docker image ls)

Command: `docker image ls`

Alternatively you can use a GUI application, or the Docker VSCode extension.

## Running images (docker run)

Command: `docker run hello-docker`

In the Docker VSCode extension, you can use 'run' or 'run interactive'.

Options:
* Port forwarding: use `-p 5000:8080` flag to map a port from localhost:container

## Shared volumes (docker volume)

Create a volume using the command: `docker volume create shared-stuff`

We can mount them when we use `docker run` using: `--mount source=shared-stuff,target=/stuff`

## Debugging (docker exec)

We can see logs of an instance:
* VSCode docker extension - 'view logs'

We can interact with the shell of an instance:
* Docker command: `docker exec <container-name> <command>` or `docker exec -ti <container-name> sh` (`docker attach <container-name>` will only really give stdout)
* VSCode docker extension - 'attach shell'

## Compose (docker compose)

Create a `docker-compose.yml` file.

We can define the whole container infrastructure, including:
* Creating multiple containers at once
  * Database if needed
  * Shared volumes
* Environment variables
  * Such as API keys/passwords

Then we can use `docker compose up` to run the entire compose.

## Command List

### Image commands:

Purpose                 | Command/s
------------------------|------------------------
Build an image          | `docker build`
View images             | `docker image ls`. Consider `-a` to show hidden
Rm dangling images      | `docker image prune`

### Container commands:

Purpose                 | Command/s
------------------------|------------------------
Run a new container     | `docker run <image-name>`.
.                       | Use following args to replace the CMD.
.                       | Mount volumes (`--mount source=shared-stuff,target=/stuff`)
.                       | Detached mode (`-d`)
.                       | Remove on exit (`-r`)
.                       | Port mapping (`-p 5000:8080`)
.                       | Use `-it` if you want to have access to the container shell (like a VM).
Stop a container        | `docker stop <container-name>`
View running containers | `docker ps` or `docker container ls`. Consider `-a` to show hidden/stopped.
Rm all stopped conts.   | `docker container prune`


### Debugging commands

Purpose                 | Command/s
------------------------|------------------------
Execute command once    | `docker exec <container-name> <command>`
Attach a shell          | `docker exec <container-name> <command>` or `docker exec -ti <container-name> sh`. `-it` is for interactive mode and allocate a pseudo-TTY (so a shell works + stays open).
View stdout             | `docker logs <container-name>` or possibly `docker attach <container-name>`

### Composition/other commands

Purpose                 | Command/s
------------------------|------------------------
Create shared volume    | `docker volume create <volume-name>`
Run docker compose      | `docker compose up`

### Notes

Run an ubuntu container: `docker run -it ubuntu`
* Make sure you `apt update` to update the package database so you can install packages
* You will want to install Python3, wget, curl, iputils-ping
* Or use a specialised image like the node or python images