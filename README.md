# Docker, GitHub & DevSecOps: Hands-On Lab

![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)
![Flask](https://img.shields.io/badge/flask-%23000.svg?style=for-the-badge&logo=flask&logoColor=white)
![Python](https://img.shields.io/badge/python-3.11-blue.svg?style=for-the-badge&logo=python&logoColor=white)

> A hands-on replication of a Docker and DevSecOps lab covering container fundamentals, image building, Docker Hub publishing, and CI/CD automation with GitHub Actions — including Trivy vulnerability scanning as a hard security gate.

---

## Overview

This repository documents my completion of a hands-on DevSecOps lab assigned by my DevSecOps instructor. The lab walks through containerising a Python Flask web application, pushing it to Docker Hub, and then automating the entire build-scan-push workflow using a GitHub Actions pipeline that uses Trivy as a security gate.

The lab is split into three phases:

* **Phase 0** – Pull and inspect an existing public container (Nginx) to understand the basic Docker workflow
* **Phase 1** – Build your own Flask app, write a secure Dockerfile, push the image to Docker Hub, and share it
* **Phase 2** – Automate everything with a DevSecOps CI/CD pipeline that scans for vulnerabilities before any image is pushed

## Project Structure

```text
docker-manual-automated-lab/
├── app.py                          # Flask application (home page + /health endpoint)
├── screenshots/
├── templates/
│   └── index.html                  # Jinja2 HTML template
├── requirements.txt                # Python dependencies
├── Dockerfile                      # Container build instructions
├── .dockerignore                   # Files excluded from the image
└── .github/
    └── workflows/
        └── docker-build-push.yml   # GitHub Actions CI/CD pipeline

```

## Prerequisites

| Tool | Check | Install |
| --- | --- | --- |
| **Git** | `git --version` | [git-scm.com](https://git-scm.com) |
| **Docker** | `docker --version` | [Docker Desktop](https://www.docker.com/products/docker-desktop/) |
| **GitHub account** | — | [github.com](https://github.com) |
| **Docker Hub account** | — | [hub.docker.com](https://hub.docker.com) |
| **VS Code (recommended)** | — | [code.visualstudio.com](https://code.visualstudio.com) |

---

## Phase 0 – Pull and Inspect a Public Container

Before building anything, I started as a consumer — pulling an existing Nginx image, running it, and accessing it in the browser.

```bash
# Pull a lightweight Nginx image from Docker Hub
docker pull nginx:alpine

# Run it in detached mode, mapping host port 8080 to container port 80
docker run -d -p 8080:80 --name my-nginx nginx:alpine

# List running containers
docker ps

# View container logs
docker logs my-nginx

# Force-remove the container
docker rm -f my-nginx

```

Visiting `http://localhost:8080` confirmed that Nginx was serving its default welcome page inside the container.

### Screenshots
![Phase 0](screenshots/Phase%200%20%E2%80%93%20Pull%20and%20Inspect%20a%20Public%20Container.png)
*Figure 1: Pulling and inspecting a public Docker container.*

![Phase 0](screenshots/Phase%200%20%E2%80%93%20Pull%20and%20Inspect%20a%20Public%20Container%20(3).png)
*Figure 2: Inspecting container configuration and metadata.*

---

## Phase 1 – Build, Push, Pull & Share Your Own Image

### The Application

The app is a minimal Flask web server with two endpoints:

* `/` — Returns an HTML home page that greets with a configurable node name (set via environment variable)
* `/health` — Returns a JSON health check response used by Docker's HEALTHCHECK instruction

**app.py**

```python
from flask import Flask, render_template, jsonify
import os, datetime

app = Flask(__name__)
APP_NODE = os.environ.get("APP_NODE", "developer")

@app.route("/")
def home():
    return render_template("index.html", node=APP_NODE)

@app.route("/health")
def health():
    return jsonify({
        "status": "healthy",
        "node": APP_NODE,
        "time": datetime.datetime.utcnow().isoformat() + "Z"
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)

```

### The Dockerfile

The Dockerfile follows production-grade security practices: a slim base image, a non-root user, layer-caching optimisation for dependencies, and a Docker HEALTHCHECK.

```dockerfile
FROM python:3.11-slim

LABEL maintainer="your-name@example.com"
LABEL version="1.0.0"
LABEL description="Simple Flask web app for Docker/GitHub Actions lab"

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

CMD ["python", "app.py"]

```
![Application Dockerfile](screenshots/Phase%201%20dockerfile.png)

*Figure 4: Dockerfile used to containerize the Python application.*


### Build & Test Locally

```bash
# Build the image
docker build -t my-web-app .

# Run a container from it
docker run -d -p 5000:5000 --name webapp-test my-web-app

```
![Docker image build process](screenshots/Phase%201%20Build%20.png)

*Figure 5: Building the Docker image locally.*


![Docker build output](screenshots/Phase%201-%20build.png)

*Figure 6: Successful Docker image build output.*

![Container test](screenshots/Phase%201-%20Test.png)

*Figure 7: Testing the running Docker container locally.*

![Container health check](screenshots/Phase%201-health%20check.png)

*Figure 8: Verifying container health checks.*

### Push to Docker Hub

```bash
# Login using your Docker Hub access token (not your password)
docker login -u your-dockerhub-username

# Tag the image with the remote repository address
docker tag my-web-app your-dockerhub-username/my-web-app:v1.0.0-yaa

# Push to Docker Hub
docker push your-dockerhub-username/my-web-app:v1.0.0-yaa

```
![Docker Hub push](screenshots/Phase%201-push%20to%20dockerhub.png)

*Figure 9: Pushing the Docker image to Docker Hub.*

![Docker Hub public repository](screenshots/Phase%201-%20Docker%20Hub%20repo%20is%20public.png)

*Figure 10: Docker Hub repository configured as public.*


### Pull and Run the Published Image

To verify everything worked, I deleted my local version and pulled it fresh from the registry.

```bash
# Remove local copy
docker rmi your-dockerhub-username/my-web-app:v1.0.0-yaa

# Pull from Docker Hub
docker pull your-dockerhub-username/my-web-app:v1.0.0-yaa

# Run it
docker run -d -p 5000:5000 --name pulled-app your-dockerhub-username/my-web-app:v1.0.0-yaa

```
![Pulling Docker image from registry](screenshots/Phase%201-%20Pulling%20docker%20image%20from%20docker%20registry.png)

*Figure 11: Pulling the Docker image from Docker Hub.*

![Running pulled Docker image](screenshots/Phase%201-%20running%20container%20from%20pulled%20docker%20image.png)

*Figure 12: Running a container from the pulled Docker image.*

![Pulled image health check](screenshots/Phase%201-%20Health%20check(pulled%20docker%20image).png)

*Figure 13: Health check of the container created from the pulled image.*


![Deleting Docker resources](screenshots/Phase%201-delete.png)

*Figure 15: Cleaning up containers or images after testing.*

---

## Phase 2 – Automated DevSecOps Pipeline (GitHub Actions)

### Setting Up Secrets

Before the workflow runs, two repository secrets must be added under **Settings → Secrets and variables → Actions**.

### The Workflow File

The `.github/workflows/docker-build-push.yml` file automates the build, security scan with Trivy, and the final push.

```yaml
name: Build, Scan, Push and Verify Docker Image
# ... (workflow code)

```

### Triggering the Pipeline

```bash
git add .github/workflows/docker-build-push.yml
git commit -m "Add DevSecOps CI pipeline"
git push

```

After pushing, the workflow appears under the Actions tab and runs through each step automatically.

---

## Challenges I Faced (and How I Fixed Them)

This was not a smooth run on the first attempt. Here's exactly what went wrong and how I resolved each issue.

### 1. Docker Hub credentials not set up correctly

The initial pipeline failure was caused by incorrect secret configuration. I had not stored `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` as two separate secrets.

### 2. Trivy action version tag was wrong

The workflow referenced `aquasecurity/trivy-action@0.20.0` (missing the `v` prefix). This caused an "Unable to resolve action" error.

### 3. Trivy found HIGH and CRITICAL vulnerabilities

With `exit-code: '1'` and `severity: 'HIGH,CRITICAL'` configured, Trivy found vulnerabilities in the older base image. This is where the security gate actually worked!

**The Fix:** I upgraded the base image to a more recent patch version of `python:3.11-slim`, which inherited upstream security fixes.

---

## Clean Up

```bash
# Remove all containers
docker rm -f $(docker ps -aq) 2>/dev/null

# Remove lab images
docker rmi my-web-app your-dockerhub-username/my-web-app:v1.0.0-yaa nginx:alpine 2>/dev/null

```

### Screenshots
Phase 2 — GitHub Actions CI/CD
![GitHub repository setup](screenshots/Phase%202-setting%20up%20github%20repo.png)
*Figure 16: Setting up the GitHub repository for CI/CD automation.*


![Failed workflow](screenshots/Phase%202-%20failed%20workflow.png)
*Figure 17: Initial GitHub Actions workflow failure.*


![Multiple failed workflows](screenshots/Phase%202-Failed%20workflows.png)
*Figure 18:Trivy scan detecting HIGH and CRITICAL vulnerabilities.*


![Successful workflow](screenshots/Phase%202-%20successful%20workflow.png)
*Figure 19: Successful GitHub Actions workflow after fixes.*

Project Structure

## Key Concepts Covered

* Docker image layers and the build cache
* The principle of least privilege in containers (non-root user)
* Docker Hub access tokens vs passwords
* GitHub Actions secrets management
* Trivy as a CI security gate
* Image tagging and versioning conventions
* Smoke testing a deployed container

## Instructor's Original Lab

This lab is a replication of the original assignment created by **samuel-nartey**.

## License

MIT
