# Git-SSH

A simple Git-over-SSH server Docker image with UID/GID handling, based on
Alpine Linux.

Clients (users) can interact with the server using Git after adding their
public SSH key to the Git-SSH server for authentication.
Within the Docker container all Git repositries are managed by a single `git`
user, whos UID/GID is specified at container start.
Normal (interactive shell) SSH access to the Docker host machine that runs the
container is needed to add a new user/client (add an SSH public key) and create
or delete Git reposities.
The UML deployment diagram in the figure below gives an overview.

![UML deployment diagram](./uml-dd-deployment-overview.png "UML deployment diagram")
 
> Used terminology in this documentation:
> * Client - client machine that connects to the host
> * Host - Host machine that runs the Docker container

> Used variables in this documentation:
> * BASE_PATH - Path to the directory that contains this project’s files
>   (especially `Dockerfile` + other build-related files and
>   `docker-compose.yml`)
> * GID - GID assigned to the `git` user within the Docker container, e.g.
>   `1000`.
>   The access permissions of the Docker volumes content will be set to this
>   GID.
> * PORT - Network port used for the Git SSH connection, e.g. `2222`
> * REPO_NAME - Name of the Git repository
> * SERVER - Network address (IP/domain name) of the host
> * UID - UID assigned to the `git` user within the Docker container, e.g.
>   `1000`.
>   The acces spermissions of the Docker volumes content will be set to this
>   UID.
> * USER - SSH user used to login to the host
> * VERSION - Version of this project, e.g. `1.0.0`.
>   Adheres to [Semantic Versioning](https://semver.org).

## Applicability

The main use case for this project is to provide a very simple but secure^1
solution to host Git repositories on a network (e.g., LAN/WAN/internet), using
SSH key authentication.

^1: "Secure" here only means access restiction and encryption using SSH key
authentication.

## Makefile

Most of the instructions in this documentation can also be run with the
provided `Makefile` (which uses Docker-Compose).
Run `cd ${BASE_PATH} && make help` to see the list of available targets.

> The Makefile uses Docker-Compose, see the prerequisite in "How to run the
> container with Docker-Compose (on the host)" in the [Run section](#run).

## Build

How to build the Docker image (on the host):

```bash
$ cd ${BASE_PATH}
$ sudo docker build -t git-ssh:${VERSION} .
$ sudo docker image tag git-ssh:${VERSION} git-ssh:latest
```

## Arguments

* Exposed port: 22
* Volumes:
    * /git/keys-host: Volume to store the SSHD host keys
    * /git/keys: Volume to store the users’ public keys
    * /git/repos: Volume to store the Git repositories
* Environment variables:
    * PUID: UID that is assigned to the `git` user inside the Docker container
    * PGID: GID that is assigned to the `git` user inside the Docker container

## Run

How to run the container (on the host):

```bash
$ cd ${BASE_PATH}
$ mkdir -p ./git-ssh/keys-host/ ./git-ssh/keys/ ./git-ssh/repos/
$ sudo docker run \
  -d \
  -p ${PORT}:22 \
  -v ${BASE_PATH}/git-ssh/keys-host/:/git/keys-host/ \
  -v ${BASE_PATH}/git-ssh/keys/:/git/keys/ \
  -v ${BASE_PATH}/git-ssh/repos/:/git/repos/ \
  -e PUID=${UID} \
  -e PGID=${GID} \
  --name="git-ssh" \
  git-ssh:latest
```

How to run the container with Docker-Compose (on the host):

Prerequisite:
Copy `docker-compose.yml.example` to `docker-compose.yml` and adjust it.

```bash
$ cd ${BASE_PATH}
$ mkdir -p ./git-ssh/keys-host/ ./git-ssh/keys/ ./git-ssh/repos/
$ sudo docker-compose up -d
```

## SSH keys

> Based on [this reference](https://www.ssh.com/ssh/keygen/).

How to generate a private/public key pair (on the client; this generates
stronger keys than the default, RSA):

```bash
$ ssh-keygen -t ecdsa -b 521
```

> Or if supported by the client:
> `ssh-keygen -t ed25519`

How to add a client’s public SSH key to the Git-SSH server:

Upload the key to the host’s volume mount point (on the client):

```bash
$ scp ~/.ssh/id_ecdsa.pub ${USER}@${SERVER}:${BASE_PATH}/git-ssh/keys/
```

Restart the Docker container (on the host):

```bash
$ sudo docker restart git-ssh
```

> Or with Docker-Compose:
> `sudo docker-compose down -t 1 && sudo docker-compose up -d`

## Basic usage

How to
* check that the container works and
* list all repositories
(on the client; the client’s SSH public key must have been uploaded to the host
already):

```bash
$ ssh -p ${PORT} git@${SERVER}

~~~ Welcome to Git SSH server! ~~~

You have successfully authenticated but
interactive shell access is not provided.
[...]
```

How to create a new (bare) repository (on the host):

```bash
$ sudo docker exec -u git git-ssh git init --bare ./repos/${REPO_NAME}.git
```

> Or with Docker-Compose:
> `sudo docker-compose exec -u git git-ssh git init --bare ./repos/${REPO_NAME}.git`

How to clone a repository (on the client):

```bash
$ git clone ssh://git@${SERVER}:${PORT}/git/repos/${REPO_NAME}.git
```

How to push a (non-bare) repository that (yet) only exists locally (on the
client):

Prerequisite: Create a new (bare) repository (on the host).

> See "How to create a new (bare) repository (on the host)".

```bash
$ git remote add origin ssh://git@${SERVER}:${PORT}/git/repos/${REPO_NAME}.git
$ git push -u origin master
```

> Replace `git remote add [...]` with `git remote set-url [...]` if `origin`
> already exists.

> Repeat the `git push [...]` command for all tracking branches ...

How to upload an existing bare repository (on the client):

```bash
$ scp -r /path/to/${REPO_NAME}.git ${USER}@${SERVER}:${BASE_PATH}/git-ssh/repos/
```

> Make sure that uploaded bare repositories have the correct access permissions
> set (see "How to fix Git repository access permission issues (on the host)"
> in the [Troubleshooting section](#troubleschooting)).

## Troubleshooting

How to fix Git repository access permission issues (on the host):

```bash
$ sudo docker exec git-ssh ./fix-repos.sh
```

> Or with Docker-Compose:
> `sudo docker-compose exec git-ssh ./fix-repos.sh`
