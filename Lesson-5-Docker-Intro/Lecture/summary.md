# Docker Fundamentals: Containers, Images & Docker Hub

A study guide covering virtualization vs. containerization, Docker architecture, images and registries, essential Docker commands, port mapping, data persistence, and publishing images to Docker Hub.

> 🎥 **Lecture:** [Lecture 5. VMs vs Containers + Docker Intro: Runtimes, Images & Layers](https://www.youtube.com/watch?v=l8Pn8aaZ_hw&list=PLG8Vnz_p0gOU&index=5)

## Table of Contents

1. [Virtualization vs. Containerization](#1-virtualization-vs-containerization)
2. [Docker Architecture](#2-docker-architecture)
3. [Images, Containers & Registries](#3-images-containers--registries)
4. [Essential Docker Commands](#4-essential-docker-commands)
5. [Port Mapping](#5-port-mapping)
6. [Data Persistence & Stateful Workloads](#6-data-persistence--stateful-workloads)
7. [Running Nginx in a Container](#7-running-nginx-in-a-container)
8. [Docker Hub: Public & Private Repositories](#8-docker-hub-public--private-repositories)
9. [Mock Interview Questions](#9-mock-interview-questions)
10. [Hands-on Practice Assignment](#10-hands-on-practice-assignment)
11. [Useful Links](#11-useful-links)

---

## 1. Virtualization vs. Containerization

Both technologies isolate workloads, but at different layers.

- **Virtualization:** A hypervisor runs multiple **virtual machines (VMs)**. Each VM has its own full guest operating system and kernel.
- **Containerization:** Containers are isolated processes that **share the host's kernel**. They bundle only the application and its dependencies, not a whole OS.

```
   Virtual Machines                      Containers
+-------+ +-------+ +-------+      +-------+ +-------+ +-------+
| App A | | App B | | App C |      | App A | | App B | | App C |
| Bins  | | Bins  | | Bins  |      | Bins  | | Bins  | | Bins  |
| Guest | | Guest | | Guest |      +-------+-+-------+-+-------+
|  OS   | |  OS   | |  OS   |      |   Container runtime       |
+-------+-+-------+-+-------+      +---------------------------+
|        Hypervisor         |      |      Host OS (Linux       |
+---------------------------+      |      kernel shared)       |
|      Host OS / Hardware   |      +---------------------------+
+---------------------------+      |         Hardware          |
                                   +---------------------------+
```

### Comparison (from the lecture slide, plus context)

| Aspect | Virtualization (VM) | Containerization (Container) |
|---|---|---|
| **Setup** | Full OS setup, large size | Lightweight, small size |
| **OS support** | Any OS can be provisioned | Must match the host kernel: Linux containers need a Linux kernel |
| **CPU overcommit** | OK | OK |
| **RAM overcommit** | Not okay | Not okay (process gets **OOM killed**) |
| **Storage overcommit** | Not okay | Limited by host disk |
| **Isolation** | **More isolated** (separate kernel) | Isolated (namespaces + cgroups), but shares the kernel |
| **Startup time** | Minutes | Seconds or less |
| **Best for** | Stateful workloads, strong isolation, different OSes | Stateless apps, microservices, fast scaling, CI/CD |

### Why CPU Can Be Overcommitted but RAM Cannot

CPU time is **shareable**: if five workloads want CPU at once, they simply take turns, so everything runs slower but nothing breaks. RAM is **not shareable** in the same way: if the total demand exceeds physical memory, something must be killed. For a container, the Linux kernel's **OOM (Out Of Memory) killer** terminates the process.

```bash
# Give a container a hard memory ceiling; exceeding it gets the process OOM killed
docker run -d --name limited --memory=256m --cpus=0.5 nginx:1.27
```

> **Note:** The slide's "no RAM/storage overcommit" statements are a simplification. Hypervisors can technically overcommit RAM (ballooning, swapping) and storage (thin provisioning), but it is risky and can degrade or crash workloads.

> **Note:** On macOS and Windows, Docker Desktop runs a small Linux VM in the background, so Linux containers still get a Linux kernel.

---

## 2. Docker Architecture

Docker is a stack of components. Each layer delegates work to the one below it.

```
+--------------------+
|     Docker CLI     |   the `docker` command you type
+---------+----------+
          |  REST API (unix socket /var/run/docker.sock)
          v
+--------------------+
|  Docker Daemon     |   dockerd: manages images, networks, volumes
|     (dockerd)      |
+---------+----------+
          v
+--------------------+
|    containerd      |   manages container lifecycle and image transfer
+---------+----------+
          v
+--------------------+
|  runc (cgroups)    |   creates the container using kernel features
+---------+----------+
          v
+--------------------+
|    Linux Kernel    |   namespaces, cgroups, filesystems
+--------------------+
```

| Component | Role |
|---|---|
| **Docker CLI** | The client. Translates commands like `docker run` into API calls. |
| **Docker daemon (`dockerd`)** | The server. Builds images, manages containers, networks, and volumes. |
| **containerd** | High-level runtime. Pulls images, manages container lifecycle. |
| **runc** | Low-level runtime. Actually spawns the container process and applies kernel limits. |
| **Linux kernel** | Provides the isolation primitives (below). |

### Kernel Features Behind Containers

| Feature | Purpose |
|---|---|
| **Namespaces** | Isolate what a process can *see* (process IDs, network, filesystem mounts, hostnames). |
| **cgroups (control groups)** | Limit what a process can *use* (CPU, memory, I/O). This is what causes an OOM kill when the memory limit is exceeded. |

---

## 3. Images, Containers & Registries

### Image

An **image** is a read-only **blueprint** (template) used to create containers. It packages the application code, runtime, libraries, and filesystem. It is built from a **Dockerfile**, a text file of instructions.

### Container

A **container** is a **running instance** of an image. You can start many containers from one image.

> **Analogy:** An image is the **recipe**; a container is the **dish** cooked from it. A class is to an object what an image is to a container.

| | Image | Container |
|---|---|---|
| State | Static, read-only | Running (or stopped) process with a writable layer |
| Created by | `docker build` / `docker pull` | `docker run` |
| Analogy | Blueprint | Building made from the blueprint |

### Container vs. Server

A **server** is a machine (physical or virtual) with its own OS that runs many services. A **container** is not a mini-server. It is an isolated process (typically **one main process**) running on a shared kernel. Containers are meant to be disposable: stop one, delete it, start a fresh one.

### Dockerfile

A Dockerfile is the file of commands that defines how to build an image:

```dockerfile
FROM nginx:1.27                                  # base image
COPY index.html /usr/share/nginx/html/index.html # add our content
EXPOSE 80                                        # document the listening port
```

```bash
docker build -t my-nginx:v1 .
```

### Registry

A **registry** is a service that stores and distributes images. **Docker Hub** is the default public registry. Others include GitHub Container Registry (GHCR), Amazon ECR, Azure Container Registry, and Google Artifact Registry.

Image names follow this pattern:

```
[registry-host/]namespace/repository:tag

nginx:1.27                       -> docker.io/library/nginx:1.27 (official image)
myuser/my-nginx:v1               -> docker.io/myuser/my-nginx:v1 (your repo on Docker Hub)
ghcr.io/myorg/app:2.0            -> GitHub Container Registry
```

- If no registry is given, Docker uses **Docker Hub**.
- If no tag is given, Docker uses **`latest`**. Pin explicit versions in real projects.

---

## 4. Essential Docker Commands

| Command | Purpose |
|---|---|
| `docker images` | List images stored locally. |
| `docker pull <image>` | Download an image from a registry (Docker Hub by default). |
| `docker run <image>` | Create and start a container from an image. |
| `docker ps` | List **running** containers. |
| `docker ps -a` | List **all** containers, including stopped and finished ones. |
| `docker exec` | Run a command inside a running container. |
| `docker logs <container>` | Show container output. |
| `docker stop <container>` | Gracefully stop a running container. |
| `docker rm <container>` | Remove a stopped container. |
| `docker rmi <image>` | Remove a local image. |

### Examples

```bash
# Image management
docker images
docker pull nginx:1.27

# Run a container in the background (-d), named "web", port 3000 on host -> 80 in container
docker run -d --name web -p 3000:80 nginx:1.27

# Inspect
docker ps            # running only
docker ps -a         # running + exited

# Get an interactive shell inside the container
docker exec -it web bash        # use `sh` for minimal images such as Alpine

# View logs, then clean up
docker logs web
docker stop web
docker rm web
docker rmi nginx:1.27
```

### Useful `docker run` Flags

| Flag | Meaning |
|---|---|
| `-d` | Run detached (in the background). |
| `--name <name>` | Give the container a readable name. |
| `-p HOST:CONTAINER` | Publish a port (see next section). |
| `-it` | Interactive terminal (for shells). |
| `-e KEY=value` | Set an environment variable. |
| `-v <src>:<dest>` | Mount a volume or directory. |
| `--rm` | Delete the container automatically when it exits. |
| `--memory`, `--cpus` | Set resource limits (cgroups). |

### `docker pull` vs. `docker run`

`docker run <image>` **automatically pulls** the image from Docker Hub if it isn't present locally, so a separate `docker pull` is optional. Use `docker pull` when you want to download or update an image ahead of time.

---

## 5. Port Mapping

A container has its **own network namespace**, so a service listening on port 80 *inside* a container is not reachable from outside by default. **Port mapping** (publishing) forwards traffic from a port on the **host** to a port in the **container**.

```
  Browser / curl
        |
        |  request to localhost:3000
        v
+------------------------------+
|  Host machine                |
|   port 3000 -----------------+---+
|                              |   |  forwarded
|   +----------------------+   |   |
|   |  Container           |   |   |
|   |   Nginx on port 80 <-+---+---+
|   +----------------------+   |
+------------------------------+
```

Syntax: `-p <host_port>:<container_port>`

```bash
# Request on host port 3000 is forwarded to container port 80
docker run -d --name web -p 3000:80 nginx:1.27
curl http://localhost:3000
```

Variations:

```bash
docker run -d -p 8080:80 nginx:1.27             # different host port
docker run -d -p 127.0.0.1:3000:80 nginx:1.27   # reachable only from the host itself
docker run -d -P nginx:1.27                     # publish exposed ports on random host ports
docker port web                                 # show the mappings of a container
```

Key points:

- Each **host port can be used by only one container**. Run two Nginx containers on different host ports (for example 3000 and 3001), both mapping to container port 80.
- The container port is fixed by the application. The host port is your choice.

---

## 6. Data Persistence & Stateful Workloads

By default, anything written inside a container goes into a **temporary writable layer** that belongs to that specific container. It is **ephemeral**: it is lost when the container is **removed** and recreated from the image.

### Demonstration

```bash
docker run -d --name web -p 3000:80 nginx:1.27
docker exec web sh -c 'echo "hello" > /usr/share/nginx/html/test.txt'
curl http://localhost:3000/test.txt     # -> hello

docker restart web
curl http://localhost:3000/test.txt     # -> still hello (same container, data survives a restart)

docker rm -f web
docker run -d --name web -p 3000:80 nginx:1.27
curl http://localhost:3000/test.txt     # -> 404 (new container, data is gone)
```

> **Note:** A common simplification is "data is erased on restart." Precisely, data survives `docker restart` of the same container but is lost when the container is **deleted and recreated**, which is what happens during redeployments, scaling, and crashes handled by orchestrators.

### Keeping Data: Volumes and Bind Mounts

| Type | Command | Use case |
|---|---|---|
| **Named volume** | `-v pgdata:/var/lib/postgresql/data` | Managed by Docker. Best for databases. |
| **Bind mount** | `-v $(pwd)/site:/usr/share/nginx/html:ro` | Mount a host folder. Best for development and static content. |

```bash
# Database with persistent storage
docker run -d --name db \
  -e POSTGRES_PASSWORD=example \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Serve your own site from a host folder (read-only)
docker run -d --name web -p 3000:80 \
  -v "$(pwd)/site:/usr/share/nginx/html:ro" nginx:1.27
```

### Stateless vs. Stateful

| | Stateless | Stateful |
|---|---|---|
| Examples | Web servers, APIs, workers | Databases, message queues, file stores |
| Fits | Containers (easy to replace and scale) | Containers **with volumes**, or VMs / managed services |
| Rule | Nothing important stored locally | Must save data before a restart or replacement |

Per the lecture, **virtualization is traditionally used for programs that have state** and must save information across restarts. Containers can run stateful software too, but only when data is kept in volumes or external storage.

---

## 7. Running Nginx in a Container

**Nginx** is a web server and reverse proxy. Running it in Docker takes one command, with no installation on the host.

### Quick Start

```bash
docker run -d --name web -p 3000:80 nginx:1.27
curl http://localhost:3000           # shows the "Welcome to nginx!" page
docker ps                            # container is listed as running
```

### Look Inside the Container

```bash
docker exec -it web bash
# inside the container:
cat /etc/nginx/nginx.conf            # main config
ls /usr/share/nginx/html             # default web root
nginx -v                             # version
exit
```

### Custom Page via Bind Mount

```bash
mkdir site && echo "<h1>Hello from Docker</h1>" > site/index.html
docker run -d --name web2 -p 3001:80 \
  -v "$(pwd)/site:/usr/share/nginx/html:ro" nginx:1.27
curl http://localhost:3001           # -> Hello from Docker
```

### Custom Image via Dockerfile

```dockerfile
FROM nginx:1.27
COPY site/index.html /usr/share/nginx/html/index.html
```

```bash
docker build -t my-nginx:v1 .
docker run -d --name web3 -p 3002:80 my-nginx:v1
```

### Lifecycle Cheat Sheet

```bash
docker stop web web2 web3
docker ps -a                         # exited containers still listed
docker rm web web2 web3
```

---

## 8. Docker Hub: Public & Private Repositories

**Docker Hub** (hub.docker.com) is a registry where you can store and share images.

| Repository type | Who can pull | Typical use |
|---|---|---|
| **Public** | Anyone | Open-source projects, demos |
| **Private** | Only you and users you grant access to | Internal or proprietary applications |

### Workflow: Build, Tag, Push, Pull

```
 docker build  ->  docker tag  ->  docker login  ->  docker push
   (image)        (add repo name)    (authenticate)    (upload to Hub)
                                                              |
                         docker pull / docker run  <----------+
```

```bash
# 1. Build an image locally
docker build -t my-nginx:v1 .

# 2. Authenticate with Docker Hub
docker login

# 3. Tag it with your Docker Hub namespace and repository name
docker tag my-nginx:v1 <your-username>/my-nginx:v1

# 4. Push it
docker push <your-username>/my-nginx:v1

# 5. Pull or run it from anywhere
docker pull <your-username>/my-nginx:v1
docker run -d -p 3000:80 <your-username>/my-nginx:v1
```

Key points:

- The image name **must include your Docker Hub username** (the namespace) to be pushed: `username/repository:tag`.
- **Create the repository** on Docker Hub first (or let the first push create it) and choose **public** or **private** visibility.
- Pulling from a **private** repo requires `docker login` first. Without it you get a "pull access denied" or "repository does not exist" error.
- **Never push secrets** (passwords, API keys, private keys) inside images, even to a private repo.

---

## 9. Mock Interview Questions

### Q1. What is the difference between virtualization and containerization? When would you choose each?

**Answer:**
Virtualization uses a hypervisor to run multiple virtual machines, each with its **own full guest OS and kernel**. This gives strong isolation, lets you run any OS, and suits stateful or legacy workloads, but VMs are large and slow to start. Containerization isolates **processes** on a **shared host kernel** using namespaces (what a process can see) and cgroups (what it can use). Containers are small, start in seconds, and are ideal for microservices, CI/CD, and horizontal scaling. The trade-off is weaker isolation than VMs (a kernel vulnerability affects all containers) and the constraint that a Linux container needs a Linux kernel.

On resources, CPU can be overcommitted because it is time-shared, while RAM cannot: a container exceeding its memory limit is killed by the kernel's OOM killer. Choose VMs for strong isolation, different operating systems, or traditional stateful systems. Choose containers for fast, repeatable, disposable application deployment.

---

### Q2. Walk through what happens, from the CLI down to the kernel, when you run `docker run nginx`.

**Answer:**
The **Docker CLI** sends the request over the Docker REST API (a Unix socket) to the **Docker daemon (`dockerd`)**. The daemon checks whether the `nginx` image exists locally and, if not, pulls it from the registry (Docker Hub by default) through **containerd**. containerd prepares the image layers and a writable layer, then hands off to **runc**, the low-level runtime. runc asks the **Linux kernel** to create new **namespaces** (PID, network, mount, and so on) and applies **cgroup** limits, then starts the container's main process (`nginx`). The result is an ordinary Linux process that is isolated and resource-limited, not a separate OS.

---

### Q3. What is the difference between an image, a container, and a registry? Does `docker run` need a prior `docker pull`?

**Answer:**
An **image** is a read-only blueprint built from a Dockerfile: filesystem layers plus metadata. A **container** is a running (or stopped) instance of an image with its own writable layer, process space, and network namespace. A **registry** (Docker Hub, GHCR, ECR, and others) stores and distributes images, and every image name resolves to a registry, with Docker Hub as the default.

`docker pull` is not required first: `docker run` pulls the image automatically if it isn't available locally. Pulling explicitly is useful to pre-download, verify, or update an image. In production, pin versions (`nginx:1.27`) instead of relying on `latest`.

---

### Q4. Explain port mapping. What does `-p 3000:80` do, and why is data inside a container considered ephemeral?

**Answer:**
Each container has its own network namespace, so a service on port 80 inside is not reachable from outside by default. `-p 3000:80` publishes the container's port 80 on the host's port 3000, so requests to `host:3000` are forwarded to the container's port 80. The format is `host:container`, and a host port can be bound by only one container at a time.

Container data lives in a writable layer tied to that specific container. It survives a `docker restart`, but disappears when the container is **removed and recreated**, which happens on redeploys, scaling, and crash recovery. Stateful workloads such as databases must therefore store data in **volumes** (for example `-v pgdata:/var/lib/postgresql/data`) or external storage, so the data outlives any single container.

---

## 10. Hands-on Practice Assignment

**Goal:** Practice the core Docker workflow, then publish one public and one private image to Docker Hub.

### Prerequisites

- Docker installed (Docker Desktop or Docker Engine). Verify with `docker --version`.
- A free Docker Hub account at hub.docker.com.

### Step 1: Verify the Installation

```bash
docker --version
docker run hello-world
```

**Expected:** A "Hello from Docker!" message. Docker pulled the image automatically because it wasn't present locally.

### Step 2: Explore Images and Containers

```bash
docker images
docker ps
docker ps -a
```

**Expected:** `hello-world` appears under images and, in `docker ps -a`, as an **exited** container (not in `docker ps`).

### Step 3: Run Nginx with Port Mapping

```bash
docker run -d --name web -p 3000:80 nginx:1.27
curl http://localhost:3000
```

**Expected:** The "Welcome to nginx!" HTML. Also open `http://localhost:3000` in a browser.

### Step 4: Go Inside the Container

```bash
docker exec -it web bash
echo "<h1>Hello from inside the container</h1>" > /usr/share/nginx/html/index.html
exit
curl http://localhost:3000
```

**Expected:** The page now shows your custom message.

### Step 5: Prove That Data Is Ephemeral

```bash
docker restart web
curl http://localhost:3000        # custom page still there
docker rm -f web
docker run -d --name web -p 3000:80 nginx:1.27
curl http://localhost:3000        # back to the default page
```

**Expected:** Data survives the restart but is lost after the container is deleted and recreated.

### Step 6: Build a Custom Image

Create `index.html` and a `Dockerfile`:

```dockerfile
FROM nginx:1.27
COPY index.html /usr/share/nginx/html/index.html
```

```bash
echo "<h1>My first Docker image</h1>" > index.html
docker build -t my-nginx:v1 .
docker run -d --name custom -p 3001:80 my-nginx:v1
curl http://localhost:3001
```

**Expected:** Your custom page is served on port 3001.

### Step 7: Create Two Docker Hub Repositories (Homework)

On hub.docker.com, create:

1. `docker-demo-public` with **Public** visibility
2. `docker-demo-private` with **Private** visibility

### Step 8: Push the Image to Both

```bash
docker login

docker tag my-nginx:v1 <your-username>/docker-demo-public:v1
docker tag my-nginx:v1 <your-username>/docker-demo-private:v1

docker push <your-username>/docker-demo-public:v1
docker push <your-username>/docker-demo-private:v1
```

**Expected:** Both pushes succeed, and each repository on Docker Hub shows the `v1` tag.

### Step 9: Test Public vs. Private Access

```bash
docker rmi <your-username>/docker-demo-public:v1 <your-username>/docker-demo-private:v1
docker logout

docker run -d -p 3002:80 <your-username>/docker-demo-public:v1    # works: pulled automatically
docker run -d -p 3003:80 <your-username>/docker-demo-private:v1   # fails: needs authentication

docker login
docker run -d -p 3003:80 <your-username>/docker-demo-private:v1   # works after login
```

**Expected:** The public image pulls without login, the private one fails until you authenticate, and `docker run` pulls images automatically when they are missing.

### Step 10: Clean Up

```bash
docker rm -f $(docker ps -aq)
docker rmi $(docker images -q)
```

> This removes **all** containers and images on your machine. Skip it if you have other Docker work.

### Stretch Goals

- Start a container with `--memory=64m` and observe what happens when a process exceeds the limit.
- Run PostgreSQL with a **named volume**, create a table, delete the container, start a new one with the same volume, and confirm the data survived.
- Run two Nginx containers on different host ports and compare them with `docker ps`.
- Add a tag `latest` to your image and explain why pinning versions is safer.

---

## 11. Useful Links

| Resource | Link |
|---|---|
| Lecture 5. VMs vs Containers + Docker Intro: Runtimes, Images & Layers (Picsart Academy) | [YouTube](https://www.youtube.com/watch?v=l8Pn8aaZ_hw&list=PLG8Vnz_p0gOU&index=5) |
| Docker documentation | [docs.docker.com](https://docs.docker.com) |
| Docker CLI reference | [docs.docker.com/reference/cli/docker](https://docs.docker.com/reference/cli/docker/) |
| Docker Hub | [hub.docker.com](https://hub.docker.com) |
| Official Nginx image | [hub.docker.com/_/nginx](https://hub.docker.com/_/nginx) |
| Picsart Academy | [picsartacademy.am](https://picsartacademy.am/) |

---

## License

MIT, free to use, adapt, and share.

---

🎥 **Lecture:** [Watch the full lecture on YouTube](https://www.youtube.com/watch?v=l8Pn8aaZ_hw&list=PLG8Vnz_p0gOU&index=5)