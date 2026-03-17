# Jenkins CI/CD Pipeline Setup Guide
### Java (JDK 8) + Maven + Tomcat 8 + Docker Deployment

---

## Table of Contents

1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Step 1 — Create a Pipeline Branch](#step-1--create-a-pipeline-branch)
4. [Step 2 — Generate and Configure SSH Key](#step-2--generate-and-configure-ssh-key)
5. [Step 3 — Fix Jenkins .m2 Permissions](#step-3--fix-jenkins-m2-permissions)
6. [Step 4 — Store Git-Ignored Config Files on Jenkins Host](#step-4--store-git-ignored-config-files-on-jenkins-host)
7. [Step 5 — Configure Jenkins Pipeline Job](#step-5--configure-jenkins-pipeline-job)
8. [Step 6 — Verify Nexus is Running](#step-6--verify-nexus-is-running)
9. [Step 7 — Prepare the Dockerfile](#step-7--prepare-the-dockerfile)
10. [Step 8 — The Jenkinsfile](#step-8--the-jenkinsfile)
11. [Pipeline Stage Breakdown](#pipeline-stage-breakdown)
12. [Naming Conventions](#naming-conventions)
13. [Troubleshooting](#troubleshooting)

---

## Overview

This document describes the step-by-step process of setting up a Jenkins CI/CD pipeline for a Java web application built with Maven (JDK 8), packaged as a WAR file, and deployed via Docker using a Tomcat 8 base image.

```
GitHub Repo → Jenkins Pull → Copy Config Files → Maven Build → Docker Image → Deploy Container
```

---

## Requirements

### Server / Infrastructure

| Requirement | Details |
|---|---|
| Jenkins installed | Running as `jenkins` user |
| Docker installed | Jenkins user must be in `docker` group |
| Maven installed | Compatible with JDK 8 |
| JDK 8 installed | Set in Jenkins Global Tool Configuration |
| Nexus Repository | Must be running and accessible (pom.xml points to it) |
| Git installed | Available on the Jenkins server |

### Jenkins Plugins Required

| Plugin | Purpose |
|---|---|
| Git Plugin | Clone repositories |
| Pipeline Plugin | Run Jenkinsfile pipelines |
| Workspace Cleanup Plugin | `cleanWs()` support |
| Credentials Plugin | Store SSH keys and secrets |

### Repository Structure Expected

```
project-root/
├── src/
│   ├── main/
│   │   ├── resources/       ← global.properties goes here
│   │   └── webapp/
│   │       ├── META-INF/    ← context.xml goes here
│   │       └── WEB-INF/     ← web.xml goes here
├── Dockerfile
├── pom.xml
└── Jenkinsfile
```

---

## Step 1 — Create a Pipeline Branch

Create a dedicated branch for the Jenkins pipeline. This keeps pipeline-related files separate from the main codebase.

```bash
# from your local machine
git checkout -b jenkins-pipeline
git push origin jenkins-pipeline
```

Add the `Jenkinsfile` and `Dockerfile` to this branch before pushing.

> **Note:** Jenkins will be pointed to this branch when configuring the pipeline job.

---

## Step 2 — Generate and Configure SSH Key

Jenkins needs an SSH key to authenticate with GitHub and pull the repository.

### 2.1 — Switch to Jenkins User

```bash
sudo su - jenkins
# or
sudo -u jenkins -s /bin/bash
```

### 2.2 — Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "jenkins@your-server" -f ~/.ssh/jenkins_github
```

This creates two files:
- `~/.ssh/jenkins_github` — private key
- `~/.ssh/jenkins_github.pub` — public key

### 2.3 — Configure SSH to Use the Key for GitHub

```bash
nano ~/.ssh/config
```

Add the following:

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/jenkins_github
    IdentitiesOnly yes
```

### 2.4 — Set Correct Permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/jenkins_github
chmod 644 ~/.ssh/jenkins_github.pub
```

### 2.5 — Add Public Key to GitHub

Copy the public key:

```bash
cat ~/.ssh/jenkins_github.pub
```

Then go to:
> **GitHub Repo → Settings → Deploy Keys → Add Deploy Key**

Paste the public key. Check **Allow write access** only if needed (read-only is safer for CI).

### 2.6 — Add Private Key to Jenkins Credentials

Go to:
> **Manage Jenkins → Credentials → Global → Add Credentials**

- Kind: `SSH Username with private key`
- ID: `github-ssh-key` *(use this ID in your Jenkinsfile)*
- Username: `git`
- Private Key: paste contents of `~/.ssh/jenkins_github`

### 2.7 — Test SSH Connection

```bash
sudo -u jenkins ssh -T git@github.com
```

Expected output:
```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

### 2.8 — Add GitHub to Known Hosts (if needed)

If you get `Host key verification failed`:

```bash
sudo -u jenkins ssh-keyscan github.com >> /var/lib/jenkins/.ssh/known_hosts
```

---

## Step 3 — Fix Jenkins .m2 Permissions

Maven downloads dependencies into `.m2/repository`. Jenkins must have write access to this directory.

### Check Current Ownership

```bash
ls -la /var/lib/jenkins/
```

### Fix Permissions

```bash
sudo mkdir -p /var/lib/jenkins/.m2/repository
sudo chown -R jenkins:jenkins /var/lib/jenkins/.m2
sudo chmod -R 755 /var/lib/jenkins/.m2
```

### Verify Write Access

```bash
sudo -u jenkins touch /var/lib/jenkins/.m2/test.txt
ls -la /var/lib/jenkins/.m2/
sudo -u jenkins rm /var/lib/jenkins/.m2/test.txt
```

No error means Jenkins can write to `.m2` successfully.

> **Root Cause:** This issue typically occurs when someone ran `mvn` as `root` first, causing `.m2` to be owned by root instead of jenkins.

---

## Step 4 — Store Git-Ignored Config Files on Jenkins Host

Config files that are excluded from the repository (via `.gitignore`) must be stored on the Jenkins host and copied into the workspace during the build.

### Standard Directory Structure

```
/var/lib/jenkins/env/
└── elrs/
    ├── global.properties    ← DB config, app settings
    ├── context.xml          ← Tomcat datasource config
    └── web.xml              ← Servlet configuration
```

### Create the Directory

```bash
sudo mkdir -p /var/lib/jenkins/env/elrs
sudo chown -R jenkins:jenkins /var/lib/jenkins/env/elrs
sudo chmod -R 750 /var/lib/jenkins/env/elrs
```

### Secure Sensitive Files

```bash
sudo chmod 600 /var/lib/jenkins/env/elrs/global.properties
sudo chmod 600 /var/lib/jenkins/env/elrs/context.xml
```

### Place the Files

```bash
# copy or create files manually as jenkins user
sudo -u jenkins nano /var/lib/jenkins/env/elrs/global.properties
sudo -u jenkins nano /var/lib/jenkins/env/elrs/context.xml
sudo -u jenkins nano /var/lib/jenkins/env/elrs/web.xml
```

> **Important:** These files are never committed to the repository. They contain environment-specific or sensitive configuration that should only live on the server.

---

## Step 5 — Configure Jenkins Pipeline Job

### 5.1 — Create a New Pipeline Job

1. Go to **Jenkins Dashboard → New Item**
2. Enter the job name (e.g., `sbma-elrs`)
3. Select **Pipeline** → Click **OK**

### 5.2 — Configure the Job

Under **Pipeline** section:

- Definition: `Pipeline script from SCM`
- SCM: `Git`
- Repository URL: `git@github.com:<org>/<repo>.git`
- Credentials: select `github-ssh-key` (created in Step 2.6)
- Branch: `*/jenkins-pipeline`
- Script Path: `Jenkinsfile`

### 5.3 — Add Jenkins User to Docker Group

Jenkins must be able to run Docker commands without sudo:

```bash
sudo usermod -aG docker jenkins
# restart jenkins for group change to take effect
sudo systemctl restart jenkins
```

Verify:

```bash
sudo -u jenkins docker ps
```

---

## Step 6 — Verify Nexus is Running

The `pom.xml` points to a local Nexus repository for downloading dependencies. If Nexus is down, the Maven build will fail.

### Check if Nexus is Running

```bash
curl -s http://172.21.79.32:9000/repository/maven-public/ | head -20
# or
systemctl status nexus
```

### If Nexus is Down

```bash
sudo systemctl start nexus
```

### Verify pom.xml Repository URL

Check that your `pom.xml` has the correct Nexus URL under `<repositories>` and `<pluginRepositories>`:

```xml
<repository>
    <id>dev-server-nexus-plugins</id>
    <url>http://172.21.79.32:9000/repository/maven-public/</url>
</repository>
```

> **Note:** If Nexus is unavailable, Maven will attempt to download from Maven Central directly. This may work but will be slower and may not have all required internal artifacts.

---

## Step 7 — Prepare the Dockerfile

The Dockerfile should already be in the repository. It uses Tomcat 8 with JDK 8 as the base image and copies the built WAR file into the container.

```dockerfile
FROM tomcat:8.5-jdk8

# Remove default ROOT app
RUN rm -rf /usr/local/tomcat/webapps/ROOT

# Copy the built WAR as the ROOT application
COPY target/*.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

> **Note:** The `COPY target/*.war` copies directly from the build output — there is no separate copy step in the Jenkinsfile for this. The Dockerfile handles it during `docker build`.

---

## Step 8 — The Jenkinsfile

Place this file at the root of the `jenkins-pipeline` branch as `Jenkinsfile`.

```groovy
pipeline {
    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    agent any
    environment {
        APP_NAME       = "sbma-elrs"          // Container name — must match docker run --name
        IMAGE_NAME     = "sbma-elrs:latest"   // Repository:Tag — must match APP_NAME prefix
        HOST_PORT      = "7070"
        CONTAINER_PORT = "8080"
        ENV_DIR        = "/var/lib/jenkins/env/elrs"
    }
    stages {

        stage('Checkout Source Code') {
            steps {
                echo "===> [1/6] Pulling source code from repository..."
                checkout scm
                echo "===> Checkout complete. Listing workspace files:"
                sh 'ls -al'
                echo "===> Current branch and last commit:"
                sh 'git log --oneline -3'
            }
        }

        stage('Prepare Config Files') {
            steps {
                echo "===> [2/6] Copying gitignored config files from Jenkins host..."
                sh '''
                    echo "--- Source dir contents:"
                    ls -al ${ENV_DIR}/

                    echo "--- Copying config files to workspace..."
                    cp ${ENV_DIR}/global.properties src/main/resources/ || { echo "ERROR: global.properties missing"; exit 1; }
                    cp ${ENV_DIR}/context.xml src/main/webapp/META-INF/ || { echo "ERROR: context.xml missing"; exit 1; }
                    cp ${ENV_DIR}/web.xml src/main/webapp/WEB-INF/      || { echo "ERROR: web.xml missing"; exit 1; }

                    echo "--- Config files copied successfully:"
                    ls -al src/main/resources/
                    ls -al src/main/webapp/META-INF/
                    ls -al src/main/webapp/WEB-INF/
                '''
            }
        }

        stage('Build WAR') {
            steps {
                echo "===> [3/6] Building WAR file using Maven with JDK 8..."
                sh '''
                    echo "--- Maven version:"
                    mvn -version

                    echo "--- Starting Maven build..."
                    mvn clean package -DskipTests

                    echo "--- Build complete. WAR file:"
                    ls -alh target/*.war
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "===> [4/6] Building Docker image with Tomcat 8 + WAR..."
                sh '''
                    echo "--- Building Docker image: ${IMAGE_NAME}"
                    docker build -t ${IMAGE_NAME} -f Dockerfile .

                    echo "--- Docker image built successfully:"
                    docker images | grep ${APP_NAME}
                '''
            }# from your local machine
git checkout -b jenkins-pipeline
        }

        stage('Stop Old Container') {
            steps {
                echo "===> [5/6] Stopping and removing old container if exists..."
                sh '''
                    if [ $(docker ps -aq -f name=${APP_NAME}) ]; then
                        echo "--- Found existing container. Stopping and removing..."
                        docker stop ${APP_NAME} || true
                        docker rm   ${APP_NAME} || true
                        echo "--- Old container removed."
                    else
                        echo "--- No existing container found. Skipping."
                    fi
                '''
            }
        }

        stage('Deploy Container') {
            steps {
                echo "===> [6/6] Deploying new container..."
                sh '''
                    docker run -d \
                        --name ${APP_NAME} \
                        --restart unless-stopped \
                        -p ${HOST_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_NAME}

                    echo "--- Container started. Verifying..."
                    docker ps | grep ${APP_NAME}
                    echo "===> Deployment complete! App running on port ${HOST_PORT}"
                '''
            }
        }

    }

    post {
        success {
            echo "===> ✅ Pipeline completed successfully! App deployed on port ${HOST_PORT}"
            echo "===> Cleaning up workspace..."
            cleanWs()
            echo "===> Workspace cleaned."
        }
        failure {
            echo "===> ❌ Pipeline failed. Keeping workspace for debugging."
        }
        always {
            echo "===> Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
```

---

## Pipeline Stage Breakdown

| Stage | Step | What it Does |
|---|---|---|
| `Checkout Source Code` | 1/6 | Pulls the latest code from GitHub via SSH |
| `Prepare Config Files` | 2/6 | Copies gitignored files from Jenkins host into workspace |
| `Build WAR` | 3/6 | Runs `mvn clean package` to compile and package the WAR |
| `Build Docker Image` | 4/6 | Runs `docker build` using the Dockerfile in the repo |
| `Stop Old Container` | 5/6 | Removes the currently running container if it exists |
| `Deploy Container` | 6/6 | Runs a new Docker container from the freshly built image |
| `post (success)` | — | Cleans up the Jenkins workspace |
| `post (failure)` | — | Keeps workspace intact for debugging |

---

## Naming Conventions

Understanding the Docker naming in the pipeline:

| Variable | Value | Role |
|---|---|---|
| `APP_NAME` | `sbma-elrs` | Used as the **container name** in `docker run --name` and `docker ps` grep |
| `IMAGE_NAME` | `sbma-elrs:latest` | The **repository:tag** used in `docker build -t` and `docker run` |

```
docker build -t sbma-elrs:latest .
             └─ IMAGE_NAME (repository = sbma-elrs, tag = latest)

docker run --name sbma-elrs sbma-elrs:latest
           └─ APP_NAME      └─ IMAGE_NAME
           (container name)  (image to run)
```

> **Rule:** Keep `APP_NAME` and the repository part of `IMAGE_NAME` the same to avoid grep mismatches in `docker images`.

---

## Troubleshooting

### SSH / Checkout Issues

| Error | Fix |
|---|---|
| `Permission denied (publickey)` | Public key not added to GitHub Deploy Keys |
| `Host key verification failed` | Run `ssh-keyscan github.com >> ~/.ssh/known_hosts` as jenkins user |
| `Could not read from remote repository` | Check credentials ID in Jenkinsfile matches Jenkins credentials |

### Maven Build Issues

| Error | Fix |
|---|---|
| `Failed to create parent directories for .m2` | Run `sudo chown -R jenkins:jenkins /var/lib/jenkins/.m2` |
| `Could not transfer artifact from Nexus` | Check Nexus is running: `curl http://172.21.79.32:9000` |
| `Cannot find global.properties` | Verify file exists in `/var/lib/jenkins/env/elrs/` |
| `BUILD FAILURE - plugin not resolved` | Nexus is down or `.m2` permissions are wrong |

### Docker Issues

| Error | Fix |
|---|---|
| `permission denied while trying to connect to Docker` | Add jenkins to docker group: `sudo usermod -aG docker jenkins` |
| `docker: command not found` | Install Docker or check PATH for jenkins user |
| `port already in use` | Another process is using `HOST_PORT` — change or stop it |
| `image not found` | Build stage failed — check logs from `Build Docker Image` stage |

### Config File Issues

| Error | Fix |
|---|---|
| `ERROR: global.properties missing` | File not in `/var/lib/jenkins/env/elrs/` |
| `cp: cannot create regular file` | Target directory doesn't exist in workspace — check repo structure |
| `Permission denied` on env files | Run `sudo chown -R jenkins:jenkins /var/lib/jenkins/env/elrs` |

---

*Last updated: March 2026*
