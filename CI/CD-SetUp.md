# Docker + Jenkins + GitHub — CI/CD Setup & Administration Guide


## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Installing Docker](#3-installing-docker)
4. [Installing Java](#4-installing-java)
5. [Installing Jenkins](#5-installing-jenkins)
6. [Essential Jenkins Plugins](#6-essential-jenkins-plugins)
7. [GitHub Integration](#7-github-integration)
8. [Creating a Jenkins Pipeline](#8-creating-a-jenkins-pipeline)
9. [Docker Volume Persistence](#9-docker-volume-persistence)
10. [Security Hardening](#10-security-hardening)
11. [Troubleshooting Reference](#11-troubleshooting-reference)
12. [Quick Reference](#12-quick-reference)
13. [References](#13-references)

---

## 1. Overview

This guide covers the complete setup of a Docker-based CI/CD pipeline using Jenkins and GitHub. It is intended for system administrators managing Linux-based servers.

The stack integrates three core components:

- **Docker** — containerizes build environments and application workloads
- **Jenkins** — orchestrates builds, tests, and deployments via pipelines
- **GitHub** — stores source code and triggers Jenkins via webhooks

| Component | Role | Version Recommendation |
|---|---|---|
| Docker Engine | Container runtime | 24.x or latest stable |
| Jenkins LTS | CI/CD server | 2.541.2 |
| GitHub | Source control + webhooks | Cloud or GitHub Enterprise |
| Java (JDK) | Jenkins runtime | OpenJDK 17.0.17 LTS (Red Hat) |
| AlmaLinux / RHEL | Host OS | 8.x or 9.x |

---

## 2. Prerequisites

### 2.1 System Requirements

- **OS:** AlmaLinux 8/9, RHEL, CentOS Stream, or Ubuntu 22.04 LTS
- **RAM:** Minimum 4 GB (8 GB recommended for parallel builds)
- **Disk:** At least 30 GB free on the Jenkins home partition
- **CPU:** 2+ cores
- **Network:** Outbound HTTPS access to GitHub, Docker Hub, and Jenkins update center

### 2.2 User & Permissions

- A non-root user with `sudo` privileges (e.g., `devops` or `jenkins`)
- The `jenkins` OS user must be a member of the `docker` group

> **NOTE:** Run all commands as a sudo-capable user unless stated otherwise.

---

## 3. Installing Docker

### 3.1 Add the Docker CE Repository (AlmaLinux / RHEL)

```bash
# Remove any old Docker packages
sudo dnf remove -y docker docker-client docker-client-latest \
     docker-common docker-latest docker-latest-logrotate \
     docker-logrotate docker-engine

# Install prerequisites
sudo dnf install -y dnf-utils

# Add Docker repository
sudo dnf config-manager --add-repo \
  https://download.docker.com/linux/rhel/docker-ce.repo
```

### 3.2 Install Docker Engine

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
     docker-buildx-plugin docker-compose-plugin
```

### 3.3 Enable and Start Docker

```bash
sudo systemctl enable --now docker

# Verify Docker is running
sudo systemctl status docker
docker version
```

### 3.4 Add Jenkins User to Docker Group

This step is critical — without it, Jenkins pipeline steps that invoke `docker` will fail with a permission denied error.

```bash
sudo usermod -aG docker jenkins

# Apply group changes (restart Jenkins service)
sudo systemctl restart jenkins
```

> **WARNING:** A missing docker group membership is the most common cause of pipeline failures with the error: `Got permission denied while trying to connect to the Docker daemon socket`.

---

## 4. Installing Java (Required by Jenkins)

Jenkins requires Java 17. The tested runtime for this guide is the Red Hat build of OpenJDK 17.0.17 LTS.

```bash
# AlmaLinux / RHEL — install Java 17
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Confirm version
java -version
# Expected output:
# openjdk version "17.0.17" 2025-10-21 LTS
# OpenJDK Runtime Environment (Red_Hat-17.0.17.0.10-1) (build 17.0.17+10-LTS)
# OpenJDK 64-Bit Server VM (Red_Hat-17.0.17.0.10-1) (build 17.0.17+10-LTS, mixed mode, sharing)

# If multiple Java versions exist, set default
sudo alternatives --config java
```

---

## 5. Installing Jenkins

### 5.1 Add Jenkins Repository

```bash
# Import Jenkins GPG key
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Add Jenkins LTS repo (HTTPS — do NOT use plain HTTP)
sudo tee /etc/yum.repos.d/jenkins.repo << 'EOF'
[jenkins]
name=Jenkins-stable
baseurl=https://pkg.jenkins.io/redhat-stable
gpgcheck=1
EOF
```

### 5.2 Install and Start Jenkins

```bash
sudo dnf install -y jenkins
sudo systemctl enable --now jenkins

# Verify it started
sudo systemctl status jenkins
```

### 5.3 Initial Admin Setup

1. Open a browser and navigate to `http://<server-ip>:8080`
2. Retrieve the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Paste the password into the Jenkins unlock screen
4. Choose **Install Suggested Plugins** when prompted
5. Create the first admin user account

> **TIP:** After initial setup, change the default Jenkins port if needed by editing `/etc/sysconfig/jenkins` and setting `JENKINS_PORT=<desired_port>`.

---

## 6. Essential Jenkins Plugins

Install via **Manage Jenkins > Plugins > Available**:

| Plugin | Purpose | Required? |
|---|---|---|
| Git Plugin | Clone repos from GitHub | Yes |
| GitHub Plugin | Webhook integration & status reporting | Yes |
| Pipeline | Declarative & scripted Pipelines (Jenkinsfile) | Yes |
| Docker Pipeline | Use Docker inside Jenkinsfile (`docker.build`, `docker.inside`) | Yes |
| Credentials Binding | Inject secrets/tokens into build steps | Yes |
| Role-based Auth Strategy | Fine-grained RBAC for teams | Recommended |
| Blue Ocean | Modern UI for pipeline visualization | Optional |
| Workspace Cleanup | Auto-clean workspaces post-build | Recommended |

---

## 7. GitHub Integration

### 7.1 SSH Deploy Key Setup

For Jenkins to authenticate to GitHub without a password, generate an SSH key pair and configure it as a Deploy Key on the repository.

```bash
# Generate key pair ON the Jenkins server (as the jenkins user)
sudo -u jenkins ssh-keygen -t ed25519 -C 'jenkins@ci' \
  -f ~/.ssh/id_ed25519_github -N ''

# Print the PUBLIC key — add this to GitHub > Repo > Settings > Deploy Keys
sudo -u jenkins cat ~/.ssh/id_ed25519_github.pub
```

#### Adding the Deploy Key in GitHub

1. Open the repository in GitHub
2. Navigate to **Settings > Deploy Keys > Add deploy key**
3. Paste the public key; check **Allow write access** only if Jenkins needs to push
4. Save

#### Adding the Private Key to Jenkins Credentials

1. Go to **Manage Jenkins > Credentials > System > Global > Add Credentials**
2. Select Kind: **SSH Username with private key**
3. Set ID to a memorable name, e.g., `github-deploy-key`
4. Paste the private key content
5. Save

### 7.2 GitHub Webhook Configuration

Webhooks instruct GitHub to notify Jenkins on every push or pull request. Jenkins must be reachable from the internet on a public IP or via a tunnel (e.g., ngrok, Cloudflare Tunnel) if it is on a private network.

#### Setting up the Webhook in GitHub

1. Go to the repository > **Settings > Webhooks > Add webhook**
2. Set **Payload URL** to: `http://<jenkins-public-url>/github-webhook/`
3. Set **Content type** to `application/json`
4. Select **Just the push event** (or customize as needed)
5. Save — GitHub will send a test ping

> **NOTE:** If Jenkins is behind a private IP (e.g., `172.x.x.x`), use ngrok to expose it:
> ```bash
> ngrok http 8080
> ```
> Then set the ngrok HTTPS URL as the webhook payload URL. Update the webhook URL whenever the tunnel restarts.

---

## 8. Creating a Jenkins Pipeline

### 8.1 Branch Strategy

Only pushes to the `uat` or `deployment` branch trigger a full build, image push, and automatic deployment. All other branches (e.g., `dev`, `feature/*`) are ignored by the pipeline.

| Branch | Checkout | Build | Test | Push | Auto-Deploy |
|--------|----------|-------|------|------|-------------|
| `uat` | ✅ | ✅ | ✅ | ✅ | ✅ UAT environment |
| `deployment` | ✅ | ✅ | ✅ | ✅ | ✅ Production environment |
| `dev` / others | — | — | — | — | — |

> **How it works:** When a developer pushes to `uat` or `deployment` on GitHub, the webhook fires instantly, Jenkins picks up the push, runs the pipeline, and deploys — no manual intervention needed.

### 8.2 Create a New Pipeline Job

1. **Dashboard > New Item** → Enter job name → Select **Pipeline** → OK
2. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**

   > This is what enables automatic deployment — every push GitHub sends via webhook triggers this job immediately.

3. Under **Pipeline Definition**, select **Pipeline script from SCM**
4. Set SCM to **Git**, enter the repository URL, and select the SSH credentials
5. Set **Branch to build** to `*/uat` **and** `*/deployment`

   > To watch multiple branches, add each as a separate branch specifier using the **Add** button.

6. Set **Script Path** to `Jenkinsfile`
7. Save

### 8.3 Sample Jenkinsfile (Node.js + Docker)

Place this file at the root of your repository as `Jenkinsfile`:

```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME   = 'my-app'
    IMAGE_TAG    = "${env.BUILD_NUMBER}"
    REGISTRY     = 'registry.example.com'
    APP_PORT     = '3000'
    // Container name differs per branch
    CONTAINER    = "${env.BRANCH_NAME == 'deployment' ? 'my-app-prod' : 'my-app-uat'}"
  }

  // Only run this pipeline on uat or deployment branch pushes
  when {
    beforeAgent true
    anyOf {
      branch 'uat'
      branch 'deployment'
    }
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        echo "Branch: ${env.BRANCH_NAME} — Build #${env.BUILD_NUMBER}"
      }
    }

    stage('Build Image') {
      steps {
        sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
      }
    }

    stage('Run Tests') {
      steps {
        sh 'docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test'
      }
    }

    stage('Push to Registry') {
      steps {
        withCredentials([usernamePassword(
            credentialsId: 'registry-creds',
            usernameVariable: 'REG_USER',
            passwordVariable: 'REG_PASS')]) {
          sh '''
            echo $REG_PASS | docker login ${REGISTRY} -u $REG_USER --password-stdin
            docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
            docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy to UAT') {
      when { branch 'uat' }
      steps {
        sh '''
          docker stop my-app-uat  || true
          docker rm   my-app-uat  || true
          docker run -d --name my-app-uat \
            -p ${APP_PORT}:${APP_PORT} \
            --restart unless-stopped \
            ${IMAGE_NAME}:${IMAGE_TAG}
        '''
        echo 'Deployed to UAT environment.'
      }
    }

    stage('Deploy to Production') {
      when { branch 'deployment' }
      steps {
        sh '''
          docker stop my-app-prod || true
          docker rm   my-app-prod || true
          docker run -d --name my-app-prod \
            -p ${APP_PORT}:${APP_PORT} \
            --restart unless-stopped \
            ${IMAGE_NAME}:${IMAGE_TAG}
        '''
        echo 'Deployed to Production environment.'
      }
    }

  }

  post {
    always  { cleanWs() }
    success { echo "Pipeline succeeded on branch: ${env.BRANCH_NAME}" }
    failure { echo "Pipeline FAILED on branch: ${env.BRANCH_NAME} — check build logs." }
  }

}
```

> **Auto-deploy flow:**
> ```
> Developer pushes to uat or deployment
>         ↓
> GitHub webhook fires → Jenkins receives trigger
>         ↓
> Pipeline runs: Checkout → Build → Test → Push → Deploy
>         ↓
> Container restarted automatically with new image
> ```

---

## 9. Docker Volume Persistence

Containers are ephemeral by default — any data written inside is lost when the container is removed. Use named volumes or bind mounts to persist data across builds and deployments.

### 9.1 Named Volumes

```bash
# Create a named volume
docker volume create app-data

# Mount in a container
docker run -d --name my-app \
  -v app-data:/app/data \
  my-app:latest
```

### 9.2 Bind Mounts (Host Path)

```bash
docker run -d --name my-app \
  -v /srv/app/data:/app/data \
  my-app:latest
```

> **WARNING:** Bind mounts are tied to host paths. If Jenkins builds wipe or recreate the host directory, data will be lost. Prefer named volumes for application databases and state files.

---

## 10. Security Hardening

### 10.1 Jenkins RBAC

- Install the **Role-based Authorization Strategy** plugin
- Go to **Manage Jenkins > Security > Authorization > Role-Based Strategy**
- Define roles (Admin, Developer, Read-Only) and assign users or GitHub OAuth groups

### 10.2 Credentials Management

- Never hard-code secrets in Jenkinsfiles — use the Credentials Store
- Use `withCredentials()` to inject secrets at runtime as environment variables
- Rotate deploy keys and tokens periodically

### 10.3 Docker Socket Security

- Only the `jenkins` OS user should be in the `docker` group
- Do **NOT** expose the Docker socket over TCP without TLS
- Consider using rootless Docker or Podman for stricter isolation

### 10.4 Firewall Rules

```bash
# Open Jenkins port (default 8080) only to internal network
sudo firewall-cmd --permanent --add-rich-rule=\
  'rule family=ipv4 source address=10.0.0.0/8 port port=8080 protocol=tcp accept'

sudo firewall-cmd --reload
```

---

## 11. Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---|---|---|
| `Permission denied` on docker socket | `jenkins` user not in `docker` group | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| GitHub webhook shows 403 / no delivery | Jenkins CSRF or wrong URL | Check `/github-webhook/` trailing slash; ensure Jenkins is publicly reachable |
| ORA-00942 / DB errors in pipeline | Wrong schema context in SQL | Prefix all table names with schema owner (e.g., `SCHEMA.TABLE`) |
| Jenkins repo HTTP error / 404 on `dnf update` | HTTP repo URL (deprecated) | Change `baseurl` to HTTPS in `/etc/yum.repos.d/jenkins.repo` |
| Build takes 15–20 min each run | `--no-cache` or `--pull` flags in Dockerfile | Remove those flags to leverage Docker layer caching |
| `npm build` fails: missing `build` script | `package.json` missing build entry | Add `"build": "tsc"` (or equivalent) to `scripts` in `package.json` |
| Container data lost after Jenkins build | No volume mount for persistent data | Add `-v named-volume:/path` in `docker run` command |

---

## 12. Quick Reference

### 12.1 Useful Docker Commands

```bash
docker ps -a                            # List all containers
docker images                           # List images
docker logs <container>                 # Tail container logs
docker exec -it <container> bash        # Shell into running container
docker volume ls                        # List volumes
docker system prune -f                  # Remove dangling images/containers
docker stats                            # Live resource usage
```

### 12.2 Useful Jenkins CLI Commands

```bash
sudo systemctl restart jenkins          # Restart Jenkins service
sudo systemctl status  jenkins          # Check service status
sudo journalctl -u jenkins -f           # Tail Jenkins system logs
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 13. References

- [Docker Documentation](https://docs.docker.com)
- [Jenkins Documentation](https://www.jenkins.io/doc)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax)
- [GitHub Webhooks Guide](https://docs.github.com/en/webhooks)
- [Docker Security Best Practices](https://docs.docker.com/engine/security)

---


