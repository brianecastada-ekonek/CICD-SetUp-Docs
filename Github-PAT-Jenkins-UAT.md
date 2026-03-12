# Setting Up GitHub PAT on UAT Jenkins Server

This guide walks through generating a GitHub Fine-Grained Personal Access Token (PAT), configuring it on a UAT Jenkins server, and creating a test pipeline to verify access to private repositories.

---

## 1. Prerequisites

- UAT server with Jenkins installed.
- Git installed on Jenkins server (`git --version`).
- Access to GitHub account(s) that own the repositories.
- Admin access in Jenkins to manage credentials.

---

## 2. Generate GitHub Fine-Grained PAT

1. Login to GitHub account (used for CI).
2. Navigate to:
   
```

Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens → Generate new token

```

3. Set **Token Name** (e.g., `jenkins-uat`).
4. Set **Repository Access**:

- Select specific repositories needed for CI.
- Or “All repositories” if allowed.

5. Set **Permissions**:

| Permission             | Access  |
|------------------------|---------|
| Repository contents     | Read    |
| Repository metadata     | Read    |

6. Expiration: choose suitable expiration.
7. Click **Generate token**.
8. Copy the token (you will not see it again).

> ⚠️ **Important:** Do not share the token or commit it to Git.

---

## 3. Configure the PAT in Jenkins

1. Open Jenkins dashboard.
2. Navigate to:

```

Manage Jenkins → Credentials → Global → Add Credentials

````

3. Fill in the form:

| Field          | Value                                  |
|----------------|----------------------------------------|
| Kind           | Username with password                  |
| Username       | GitHub username (used to generate PAT) |
| Password       | PAT token                               |
| ID             | `github-pat` (or custom ID)            |
| Description    | PAT for UAT server pipeline             |

4. Save the credentials.

---

## 4. Test PAT from UAT Server (Optional)

SSH into Jenkins server as the Jenkins user:

```bash
sudo -u jenkins -i
````

Run a quick access test:

```bash
git ls-remote https://<username>:<PAT>@github.com/<owner>/<repo>.git
```

Example:

```bash
git ls-remote https://leonardSalinas-ekonek:ghp_xxxxxxxx@github.com/leonardSalinas-ekonek/todoApp.git
```

You should see commit hashes. If you get an error:

* Check the repository URL is correct (case-sensitive).
* Check the token has proper repository permissions.

---

## 5. Create a Test Pipeline in Jenkins

1. Go to **Jenkins → New Item → Pipeline**.
2. Name it `tokenTestPipeline1`.
3. Select **Pipeline** → OK.
4. In **Pipeline Script**, add:

```groovy
pipeline {
    agent any

    stages {
        stage('Clone Repository Test') {
            steps {
                script {
                    try {
                        git branch: 'main',
                            url: 'https://github.com/leonardSalinas-ekonek/todoApp.git',
                            credentialsId: 'github-pat'

                        echo "SUCCESS: Pipeline test passed. Repository cloned using PAT."

                    } catch (Exception e) {
                        echo "FAILED: Repository clone failed."
                        error("Pipeline test failed.")
                    }
                }
            }
        }
    }
}
```

5. Save and **Build Now**.
6. Verify **Console Output** shows:

```
SUCCESS: Pipeline test passed. Repository cloned using PAT.
```

---

## 6. Optional: Multi-Repo Checkout

For projects with multiple components (mobile, API, web):

```groovy
pipeline {
    agent any

    stages {

        stage('Clone Multiple Repositories') {
            steps {
                script {
                    try {
                        // Mobile App
                        dir('mobile-app') {
                            git branch: 'main',
                                url: 'https://github.com/leonardSalinas-ekonek/todoApp.git',
                                credentialsId: 'nard-github-pat'
                        }

                        // API Service
                        dir('api-service') {
                            git branch: 'main',
                                url: 'https://github.com/leonardSalinas-ekonek/ekonek_sysdev_documentation.git',
                                credentialsId: 'nard-github-pat'
                        }

                        // Web App
                        dir('web-app') {
                            git branch: 'main',
                                url: 'https://github.com/leonardSalinas-ekonek/ekonek-legacy-jars.git',
                                credentialsId: 'nard-github-pat'
                        }

                        echo "SUCCESS: All repositories cloned successfully using PAT."

                    } catch (Exception e) {
                        echo "FAILED: One or more repositories failed to clone."
                        error("Pipeline test failed.")
                    }
                }
            }
        }

        stage('Cleanup Workspace') {
            steps {
                script {
                    echo "Cleaning up cloned repositories..."
                    deleteDir() // Deletes entire workspace including cloned repos
                    echo "Workspace cleaned."
                }
            }
        }

    }
}
```

---

## 7. Security Notes

* Do not print the PAT in logs.
* Store all credentials in **Jenkins Credential Manager**.

