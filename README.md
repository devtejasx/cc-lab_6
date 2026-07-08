# CC Lab 6 — Dockerized Load Balancing with Jenkins CI/CD

![Docker](https://img.shields.io/badge/Docker-required-2496ED?logo=docker&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-pipeline-D24939?logo=jenkins&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-load--balancer-009639?logo=nginx&logoColor=white)
![C++](https://img.shields.io/badge/C%2B%2B-backend-00599C?logo=cplusplus&logoColor=white)

A Cloud Computing lab exercise demonstrating a load-balanced backend architecture: two identical C++ HTTP server containers behind an Nginx reverse proxy, deployed automatically via a Jenkins declarative pipeline.

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Running via Jenkins Pipeline](#running-via-jenkins-pipeline)
- [Running Manually (without Jenkins)](#running-manually-without-jenkins)
- [Usage Example](#usage-example)
- [CI/CD](#cicd)
- [Known Issues / Inconsistencies](#known-issues--inconsistencies)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Features

- A minimal C++ HTTP server (`backend/app.cpp`) using raw BSD sockets — no framework — that responds to any request with plain text identifying which container instance served it (via `gethostname()`), which makes load balancing observable.
- An Nginx reverse proxy configured as a round-robin load balancer across two backend instances (`nginx/default.conf`).
- A Jenkins declarative pipeline (`Jenkinsfile`) that builds the backend Docker image, deploys two backend containers on a shared Docker network, and deploys/configures the Nginx load balancer — fully automating the setup.
- A custom Jenkins agent image (`Dockerfile.jenkins`) with Docker CLI, `make`, `g++`, and `curl` pre-installed so the pipeline can build and run Docker containers from within Jenkins itself.

## Tech Stack

- **C++** (raw POSIX sockets: `socket`, `bind`, `listen`, `accept`) — the backend HTTP server
- **Docker** — containerizes the backend and hosts the Nginx load balancer
- **Nginx** — `upstream` round-robin reverse proxy
- **Jenkins** (declarative pipeline) — CI/CD orchestration, built on the official `jenkins/jenkins:lts` image extended with Docker-in-Docker tooling

## Architecture Overview

```
                     ┌────────────────────┐
   client  ───────▶  │  nginx-lb (:80)     │
                     │  upstream backend_  │
                     │  servers (round     │
                     │  robin)             │
                     └─────────┬──────────┘
                                │
                 ┌──────────────┴───────────────┐
                 ▼                               ▼
        ┌─────────────────┐            ┌─────────────────┐
        │ backend1 (:8080)│            │ backend2 (:8080)│
        │ C++ socket server│            │ C++ socket server│
        └─────────────────┘            └─────────────────┘
   all three containers share a Docker bridge network: app-network
```

The Jenkins pipeline builds one `backend-app` image and runs it twice (`backend1`, `backend2`) on a Docker network named `app-network`, then starts a stock `nginx` container on that same network, copies in `nginx/default.conf` (which proxies to the `backend_servers` upstream — `backend1:8080` and `backend2:8080`), and reloads Nginx to apply it.

## Project Structure

```
cc-lab_6/
├── backend/
│   ├── app.cpp          # minimal raw-socket HTTP server, returns hostname of the responding container
│   └── Dockerfile        # Ubuntu 22.04 + g++, compiles and runs app.cpp on port 8080
├── nginx/
│   └── default.conf       # round-robin upstream + reverse proxy on port 80
├── Jenkinsfile             # declarative pipeline: build image -> run 2 backend containers -> deploy Nginx LB
└── Dockerfile.jenkins       # Jenkins LTS image + docker.io/make/g++/curl, for a Jenkins agent capable of running this pipeline
```

## Prerequisites

- Docker (with the daemon reachable from wherever you run the commands/pipeline)
- Jenkins (only if running via the pipeline) with Docker CLI access from the Jenkins agent (see `Dockerfile.jenkins`)

## Running via Jenkins Pipeline

1. Build/run a Jenkins agent capable of executing Docker commands, e.g. using the provided image:
   ```bash
   docker build -t jenkins-docker -f Dockerfile.jenkins .
   docker run -d --name jenkins -p 8081:8080 -v /var/run/docker.sock:/var/run/docker.sock jenkins-docker
   ```
   (Mounting the host's Docker socket is what lets the Jenkins container run `docker build`/`docker run` itself — note this grants the Jenkins container root-equivalent control over the host's Docker daemon, standard practice for Docker-based CI agents but worth being aware of.)
2. Create a Jenkins pipeline job pointing at this repository's `Jenkinsfile`.
3. Run the pipeline. It executes three stages: **Build Backend Image**, **Deploy Backend Containers**, **Deploy NGINX Load Balancer**.

> **Note:** the `Jenkinsfile` references paths `CC_LAB-6/backend` and `CC_LAB-6/nginx/default.conf` — see [Known Issues](#known-issues--inconsistencies) for what this assumes about the Jenkins workspace layout.

## Running Manually (without Jenkins)

You can reproduce exactly what the pipeline does, directly with Docker:

```bash
# 1. Build the backend image
docker build -t backend-app backend/

# 2. Create a shared network and run two backend instances
docker network create app-network
docker run -d --name backend1 --network app-network backend-app
docker run -d --name backend2 --network app-network backend-app

# 3. Run Nginx on the same network and load the load-balancer config
docker run -d --name nginx-lb --network app-network -p 80:80 nginx
docker cp nginx/default.conf nginx-lb:/etc/nginx/conf.d/default.conf
docker exec nginx-lb nginx -s reload
```

## Usage Example

Hit the load balancer repeatedly and observe the response alternate between backend containers:

```bash
curl http://localhost/
# Served by backend: <container-id-of-backend1>

curl http://localhost/
# Served by backend: <container-id-of-backend2>
```

## CI/CD

The `Jenkinsfile` itself **is** the CI/CD pipeline for this project — there is no separate GitHub Actions workflow. Its three stages:

1. **Build Backend Image** — removes any existing `backend-app` image, rebuilds it from `backend/`
2. **Deploy Backend Containers** — creates `app-network` if missing, removes any existing `backend1`/`backend2` containers, starts two fresh instances
3. **Deploy NGINX Load Balancer** — removes any existing `nginx-lb` container, starts a fresh stock `nginx` container, copies in the load-balancer config, and reloads Nginx

## Known Issues / Inconsistencies

1. **Path mismatch between `Jenkinsfile` and actual repo layout.** The pipeline references `CC_LAB-6/backend` and `CC_LAB-6/nginx/default.conf`, implying the Jenkins job checks out this repository into a workspace subdirectory literally named `CC_LAB-6`. The actual repository (and its folders) are named `cc-lab_6`/`backend`/`nginx` with no `CC_LAB-6` prefix anywhere. If the Jenkins job's working directory doesn't happen to be named `CC_LAB-6`, both `docker build` and `docker cp` steps in the `Jenkinsfile` will fail with a "no such file or directory" error. Rename the paths in `Jenkinsfile` to match the actual repo structure (`backend`, `nginx/default.conf`) or configure the Jenkins job to check out into a `CC_LAB-6` subdirectory to match.
2. **No `.gitignore`, no LICENSE file.**
3. **No automated tests** — this is an infrastructure/ops lab exercise, so manual verification via `curl` (see [Usage Example](#usage-example)) is the intended validation method.
4. **Backend container has no restart policy** and the C++ server has no graceful shutdown handling — acceptable for a lab exercise, but not production-hardened.

## Troubleshooting

- **`docker build` fails with "no such file or directory: CC_LAB-6/backend"** — see [Known Issues](#known-issues--inconsistencies) #1; either adjust the path in `Jenkinsfile` or check out the repo into a `CC_LAB-6`-named folder.
- **`curl http://localhost/` connection refused** — confirm the `nginx-lb` container is running and port 80 isn't already in use on the host.
- **Load balancer always returns the same backend** — confirm both `backend1` and `backend2` containers are running and on `app-network`, and that `nginx -s reload` was executed after copying in `default.conf`.

## License

No `LICENSE` file exists in this repository.
