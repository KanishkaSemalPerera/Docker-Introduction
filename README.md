# Docker Introduction 🐳

A simple, beginner-friendly guide to Docker — what it is, how it works, and how to use it.
This guide follows the official Docker learning path. For the full official docs, see the reference at the bottom:
👉 https://docs.docker.com/get-started/

---

## Table of Contents

1. [What is Docker?](#1-what-is-docker)
2. [Docker Architecture (Full Picture)](#2-docker-architecture-full-picture)
3. [Docker Concepts](#3-docker-concepts)
   - [3.1 The Basics](#31-the-basics)
   - [3.2 Building Images](#32-building-images)
     - [Understanding Image Layers](#understanding-image-layers)
     - [Writing a Dockerfile](#writing-a-dockerfile)
     - [Build, Tag, and Publish an Image](#build-tag-and-publish-an-image)
     - [Using the Build Cache](#using-the-build-cache)
     - [Multi-Stage Builds](#multi-stage-builds)
   - [3.3 Running Containers](#33-running-containers)
     - [Publishing and Exposing Ports](#publishing-and-exposing-ports)
     - [Overriding Container Defaults](#overriding-container-defaults)
     - [Persisting Container Data](#persisting-container-data)
     - [Sharing Local Files with Containers](#sharing-local-files-with-containers)
     - [Multi-Container Applications](#multi-container-applications)
4. [Reference](#4-reference)

---

## 1. What is Docker?

Imagine you build an app on your laptop and it works perfectly. But when you send it to your friend, or upload it to a server, it breaks — because their computer has a different setup (different OS, missing libraries, different versions of software).

**Docker solves this problem.**

Docker is a tool that packages your application **and everything it needs to run** (code, libraries, settings, dependencies) into one box called a **container**. That container runs the same way on any computer — your laptop, your teammate's laptop, or a cloud server.

> 💡 Think of it like a **shipping container** for software. No matter what ship (computer) carries it, the contents inside stay the same and work the same way.

```mermaid
flowchart LR
    A["Your Code"] --> D["📦 Docker Container<br/>(code + libraries + settings)"]
    D --> B["💻 Your Laptop"]
    D --> C["🖥️ Server"]
    D --> E["☁️ Cloud"]
```

**Why people use Docker:**
- ✅ "It works on my machine" problems disappear.
- ✅ Fast to set up — no manually installing software.
- ✅ Easy to share — send one file (a Dockerfile or image), everyone gets the same environment.
- ✅ Lightweight compared to full virtual machines.

---

## 2. Docker Architecture (Full Picture)

Docker uses a **client-server architecture**. There are three main pieces working together:

| Piece | What it is |
|-------|------------|
| **Docker Client (CLI)** | The tool you type commands into (`docker run`, `docker build`...). It sends your commands to the daemon. |
| **Docker Daemon (`dockerd`)** | The background service that does the real work — building images, running containers, managing networks and volumes. |
| **Docker Registry** | A storage/distribution service for images (e.g. **Docker Hub**). The daemon pulls images from it and can push images to it. |

The Client and Daemon can be on the same machine, or the Client can talk to a Daemon running on a remote machine/server.

```mermaid
flowchart LR
    subgraph ClientSide["Your Computer"]
        CLI["🖥️ Docker Client (CLI)<br/>docker build / run / pull / push"]
    end

    subgraph Host["Docker Host"]
        Daemon["⚙️ Docker Daemon (dockerd)<br/>REST API"]
        Images["📦 Images"]
        Containers["▶️ Containers"]
        Networks["🌐 Networks"]
        Volumes["📁 Volumes"]
        Daemon --> Images
        Daemon --> Containers
        Daemon --> Networks
        Daemon --> Volumes
    end

    subgraph Registry["Docker Registry"]
        Hub["☁️ Docker Hub<br/>(or private registry)"]
    end

    CLI -->|"REST API / CLI commands"| Daemon
    Daemon <-->|"docker pull / docker push"| Hub
```

**How a simple command flows through this architecture:**

```mermaid
sequenceDiagram
    participant You as You (Docker Client)
    participant Daemon as Docker Daemon
    participant Hub as Docker Hub (Registry)

    You->>Daemon: docker run nginx
    Daemon->>Daemon: Is "nginx" image available locally?
    alt Image not found locally
        Daemon->>Hub: Pull "nginx" image
        Hub-->>Daemon: Image downloaded
    end
    Daemon->>Daemon: Create container from image
    Daemon-->>You: Container is running ✅
```

**Underneath the daemon**, Docker relies on operating system features (like Linux namespaces and cgroups) to isolate each container's processes, filesystem, and resources — this is what makes containers lightweight compared to full virtual machines (no separate guest OS per container).

```mermaid
flowchart TB
    subgraph VM["Virtual Machines (heavy)"]
        H1["Host OS"] --> HV["Hypervisor"]
        HV --> G1["Guest OS 1"] --> A1["App A"]
        HV --> G2["Guest OS 2"] --> A2["App B"]
    end
    subgraph DK["Docker Containers (lightweight)"]
        H2["Host OS"] --> DE["Docker Engine"]
        DE --> C1["Container 1: App A"]
        DE --> C2["Container 2: App B"]
    end
```

Containers share the host machine's OS kernel, while each VM needs its own full guest OS — that's why containers start faster and use fewer resources.

---

## 3. Docker Concepts

Docker has a few core building blocks. Understanding these makes everything else easy.

```mermaid
flowchart TB
    subgraph Build["🔨 BUILD"]
        DF["Dockerfile<br/>(instructions)"] --> IMG["Image<br/>(packaged app)"]
    end
    subgraph Run["▶️ RUN"]
        IMG --> CTR["Container<br/>(running app)"]
    end
    subgraph Share["📤 SHARE"]
        IMG --> REG["Registry<br/>(Docker Hub)"]
        REG --> IMG
    end
```

### 3.1 The Basics

There are 3 key terms you must know:

| Term | Simple Meaning |
|------|----------------|
| **Dockerfile** | A text file with step-by-step instructions on how to build your app's image (like a recipe). |
| **Image** | A read-only snapshot/package of your app + everything it needs. Built from a Dockerfile. (like a frozen meal, ready to cook) |
| **Container** | A running instance of an image. This is your app actually running. (the cooked meal, ready to eat) |

**Relationship:**

```
Dockerfile  --(docker build)-->  Image  --(docker run)-->  Container
 (recipe)                       (package)                  (running app)
```

Other basic commands you'll use a lot:

```bash
docker --version        # Check Docker is installed
docker ps                # List running containers
docker images            # List downloaded/built images
docker pull <image>       # Download an image from Docker Hub
docker run <image>        # Run a container from an image
docker stop <container>   # Stop a running container
docker rm <container>     # Remove a stopped container
docker rmi <image>        # Remove an image
```

---

### 3.2 Building Images

An **image** is what you build once and run anywhere. Let's break down how images are made.

#### Understanding Image Layers

A Docker image is not one big file — it's built from **layers stacked on top of each other**. Each instruction in a Dockerfile (like `FROM`, `RUN`, `COPY`) creates a new layer.

```mermaid
flowchart BT
    L1["Layer 1: Base OS (e.g. ubuntu)"] --> L2["Layer 2: Install dependencies"]
    L2 --> L3["Layer 3: Copy app code"]
    L3 --> L4["Layer 4: Set startup command"]
    L4 --> IMG["📦 Final Image"]
```

**Why layers matter:**
- Layers are **cached** — if a layer hasn't changed, Docker reuses it instead of rebuilding it. This makes builds much faster.
- Layers are **shared** between images — if two images use the same base layer, Docker only stores it once, saving disk space.

#### Writing a Dockerfile

A `Dockerfile` is a plain text file containing instructions Docker follows to build your image. Example for a simple Node.js app:

```dockerfile
# 1. Start from a base image
FROM node:20-alpine

# 2. Set the working directory inside the container
WORKDIR /app

# 3. Copy dependency files first (for caching benefits)
COPY package*.json ./

# 4. Install dependencies
RUN npm install

# 5. Copy the rest of the app code
COPY . .

# 6. Tell Docker which port the app uses
EXPOSE 3000

# 7. Command to run when the container starts
CMD ["node", "app.js"]
```

**Common Dockerfile instructions:**

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Choose the base image to start from |
| `WORKDIR` | Set the working folder inside the container |
| `COPY` | Copy files from your computer into the image |
| `RUN` | Execute a command while building (e.g. install packages) |
| `EXPOSE` | Document which port the app listens on |
| `CMD` | The default command that runs when the container starts |

#### Build, Tag, and Publish an Image

**Build** an image from your Dockerfile:

```bash
docker build -t my-app:1.0 .
```
- `-t my-app:1.0` → tags (names) the image as `my-app` with version `1.0`
- `.` → tells Docker to look for the Dockerfile in the current folder

**Tag** an image (give it another name, e.g. for a registry):

```bash
docker tag my-app:1.0 myusername/my-app:1.0
```

**Publish (push)** it to Docker Hub so others can pull it:

```bash
docker login
docker push myusername/my-app:1.0
```

```mermaid
sequenceDiagram
    participant You as Your Computer
    participant Hub as Docker Hub (Registry)
    participant Other as Teammate's Computer

    You->>You: docker build -t my-app .
    You->>Hub: docker push my-app
    Other->>Hub: docker pull my-app
    Other->>Other: docker run my-app
```

#### Using the Build Cache

Docker rebuilds only the layers that changed, and reuses ("caches") the rest. This is why the **order of instructions in a Dockerfile matters**.

✅ **Good practice:** Copy files that change *less* often (like `package.json`) before files that change *often* (your source code).

```dockerfile
COPY package*.json ./     # changes rarely -> cached most of the time
RUN npm install            # only reruns if package.json changed
COPY . .                   # your code changes often -> put this last
```

```mermaid
flowchart LR
    A["COPY package.json"] -->|unchanged, reused from cache| B["RUN npm install"]
    B -->|unchanged, reused from cache| C["COPY . . (source code)"]
    C -->|changed, rebuilt| D["New Image Layer"]
```

If you put `COPY . .` first, then any tiny code change would invalidate the cache for everything after it — slowing every build down.

#### Multi-Stage Builds

Sometimes you need extra tools to **build** your app (like compilers) but don't want them in your **final** image (to keep it small and secure). Multi-stage builds solve this by using multiple `FROM` steps in one Dockerfile.

```dockerfile
# Stage 1: Build the app
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Stage 2: Run the app (small, clean final image)
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
CMD ["node", "dist/server.js"]
```

```mermaid
flowchart LR
    subgraph Stage1["Stage 1: Build (large, has build tools)"]
        S1["node:20 + source code + build tools"] --> OUT["Compiled output"]
    end
    subgraph Stage2["Stage 2: Final Image (small, clean)"]
        OUT --> S2["node:20-alpine + only compiled output"]
    end
```

**Result:** a much smaller final image, because build tools and source files are left behind in Stage 1.

---

### 3.3 Running Containers

Once you have an image, you run it as a **container** — a live, working instance of your app.

```bash
docker run my-app:1.0
```

#### Publishing and Exposing Ports

By default, a container is isolated — the outside world can't reach it. To access an app running inside a container (like a web server), you must **publish** a port, mapping a port on your machine to a port inside the container.

```bash
docker run -p 8080:80 my-app
#              ^^^^ ^^
#        host port : container port
```

```mermaid
flowchart LR
    Browser["🌐 Browser<br/>localhost:8080"] --> Host["Host Machine<br/>Port 8080"]
    Host -->|"-p 8080:80"| Container["Container<br/>Port 80 (app inside)"]
```

- `EXPOSE` in a Dockerfile is just documentation of which port the app uses.
- `-p` (publish) is what actually makes the port reachable from your computer.

#### Overriding Container Defaults

A Dockerfile's `CMD` sets a default command, but you can override behavior at run time:

```bash
docker run -e APP_ENV=production my-app     # set environment variable
docker run my-app npm run test              # override the default command
docker run --name my-container my-app       # give the container a custom name
```

| Flag | What it Overrides |
|------|--------------------|
| `-e KEY=value` | Environment variables |
| `<image> <command>` | The default `CMD` |
| `--name` | Container's name |
| `--entrypoint` | The default `ENTRYPOINT` |

#### Persisting Container Data

Containers are **temporary** by default — when a container is removed, any data written inside it is lost. To keep data (like a database's files), use **volumes**.

```bash
docker run -v my-data:/var/lib/mysql mysql
#             ^^^^^^^ ^^^^^^^^^^^^^^
#             volume   path inside container
```

```mermaid
flowchart LR
    Container["Container<br/>(can be deleted anytime)"] <--> Volume["📁 Volume<br/>'my-data' (persists on disk)"]
```

Even if you delete and recreate the container, the volume `my-data` keeps your data safe.

#### Sharing Local Files with Containers

During development, it's useful to let a container see files directly from your computer, using a **bind mount**:

```bash
docker run -v $(pwd):/app my-app
#             ^^^^^^ ^^^^
#          your folder  path inside container
```

```mermaid
flowchart LR
    Local["💻 Your Local Folder<br/>(source code)"] <--> Container["Container<br/>/app"]
```

This is great for live-reloading during development — edit code on your machine, and the container sees the change instantly.

**Volume vs Bind Mount (quick comparison):**

| | Volume | Bind Mount |
|---|--------|------------|
| Managed by | Docker | You (a specific folder path) |
| Best for | Persisting data (databases) | Local development (live code editing) |

#### Multi-Container Applications

Real apps often need more than one container — e.g., a web app + a database + a cache. Running and connecting them by hand is tedious, so we use **Docker Compose**, which defines all containers ("services") in one YAML file.

```yaml
# docker-compose.yml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Run everything with one command:

```bash
docker compose up
```

```mermaid
flowchart TB
    subgraph Compose["docker-compose.yml"]
        Web["🌐 web container<br/>(your app)"]
        DB["🗄️ db container<br/>(postgres)"]
        Web -->|talks to| DB
        DB --- Vol["📁 db-data volume"]
    end
```

Docker Compose automatically creates a private network so containers can talk to each other by service name (e.g., `web` connects to `db` using the hostname `db`).

---

## 4. Reference

This guide is a simplified summary based on Docker's official documentation. For deeper detail, hands-on tutorials, and up-to-date instructions, always check the official source:

📖 **Docker Official Get Started Guide:** https://docs.docker.com/get-started/

Additional official resources:
- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Docker Compose docs: https://docs.docker.com/compose/
- Docker Hub (image registry): https://hub.docker.com/
