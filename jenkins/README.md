# Jenkins CI/CD Guide

A complete guide to setting up Jenkins on Amazon Linux, configuring pipelines, and deploying the Landmark Technologies web application.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Installing Jenkins on Amazon Linux](#installing-jenkins-on-amazon-linux)
3. [Accessing Jenkins](#accessing-jenkins)
4. [Initial Configuration](#initial-configuration)
5. [Required Tools & Plugins](#required-tools--plugins)
6. [Pipeline Types & Syntax](#pipeline-types--syntax)
7. [Creating a Pipeline for This Application](#creating-a-pipeline-for-this-application)
8. [Advanced Jenkins](#advanced-jenkins)

---

## Introduction

Jenkins is an open-source automation server used to build, test, and deploy software. It supports CI/CD pipelines through a rich plugin ecosystem and can integrate with virtually any tool in the DevOps toolchain.

**Why Jenkins?**
- Free and open-source
- Massive plugin ecosystem (1,800+ plugins)
- Supports declarative and scripted pipelines
- Integrates with Docker, Kubernetes, AWS, GitHub, and more
- Scales with distributed builds via agents/nodes

---

## Installing Jenkins on Amazon Linux

### Prerequisites

- Amazon Linux 2023 or Amazon Linux 2 EC2 instance
- Instance type: `t3.medium` or larger (Jenkins needs at least 2GB RAM)
- Security group allowing inbound on ports **22** (SSH) and **8080** (Jenkins)

### Step 1: Install Java (JDK 17)

```bash
# Amazon Linux 2023
sudo dnf install java-17-amazon-corretto -y

# Amazon Linux 2
sudo amazon-linux-extras install java-openjdk17 -y

# Verify
java -version
```

### Step 2: Add Jenkins Repository & Install

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# Amazon Linux 2023
sudo dnf install jenkins -y

# Amazon Linux 2
sudo yum install jenkins -y
```

### Step 3: Start & Enable Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

### Step 4: Open Firewall (if applicable)

```bash
# If using firewalld
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

---

## Accessing Jenkins

### First-Time Access

1. Open your browser and navigate to:
   ```
   http://<your-ec2-public-ip>:8080
   ```

2. Retrieve the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

3. Paste the password into the "Unlock Jenkins" screen.

4. Choose **Install suggested plugins** (recommended for beginners).

5. Create your first admin user.

6. Set the Jenkins URL (use your EC2 public IP or domain).

### Security Group Configuration

Ensure your EC2 security group has:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | Your IP |
| Custom TCP | TCP | 8080 | Your IP or 0.0.0.0/0 |

> **Production tip:** Restrict port 8080 to your IP or put Jenkins behind a reverse proxy (Nginx/ALB) with HTTPS.

---

## Initial Configuration

### Configure Global Tools

Navigate to **Manage Jenkins → Tools**:

| Tool | Configuration |
|------|--------------|
| JDK | Install JDK 17 automatically or point to `/usr/lib/jvm/java-17` |
| NodeJS | Add NodeJS 18 installation (via NodeJS plugin) |
| Git | Usually auto-detected at `/usr/bin/git` |

### Configure Credentials

Navigate to **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

| Credential ID | Type | Purpose |
|---------------|------|---------|
| `dockerhub-creds` | Username with password | Docker Hub login |
| `aws-creds` | AWS Credentials | AWS access key + secret key |
| `github-token` | Secret text | GitHub personal access token |
| `kubeconfig` | Secret file | Kubernetes config for kubectl |

#### Setting Up Docker Hub Credentials

1. Go to **Manage Jenkins → Credentials → System → Global credentials**
2. Click **Add Credentials**
3. Fill in:
   - **Kind:** Username with password
   - **Scope:** Global
   - **Username:** Your Docker Hub username
   - **Password:** Your Docker Hub access token (generate at https://hub.docker.com/settings/security)
   - **ID:** `dockerhub-creds`
   - **Description:** Docker Hub login
4. Click **Create**

#### Setting Up AWS Credentials

1. Install the **AWS Credentials** plugin first
2. Go to **Manage Jenkins → Credentials → System → Global credentials**
3. Click **Add Credentials**
4. Fill in:
   - **Kind:** AWS Credentials
   - **Scope:** Global
   - **ID:** `aws-creds`
   - **Description:** AWS IAM for EKS
   - **Access Key ID:** Your IAM access key
   - **Secret Access Key:** Your IAM secret key
5. Click **Create**

> **IAM Policy Required:** The IAM user needs permissions for `eks:DescribeCluster`, `eks:ListClusters`, and `sts:GetCallerIdentity`.

#### Setting Up GitHub Token

1. Generate a token at https://github.com/settings/tokens (select `repo` scope)
2. Go to **Manage Jenkins → Credentials → System → Global credentials**
3. Click **Add Credentials**
4. Fill in:
   - **Kind:** Secret text
   - **Scope:** Global
   - **Secret:** Your GitHub personal access token
   - **ID:** `github-token`
   - **Description:** GitHub PAT
5. Click **Create**

#### Setting Up Kubeconfig Credential

1. Generate your kubeconfig locally:
   ```bash
   aws eks update-kubeconfig --name landmark-eks --region us-east-1
   cat ~/.kube/config
   ```
2. Go to **Manage Jenkins → Credentials → System → Global credentials**
3. Click **Add Credentials**
4. Fill in:
   - **Kind:** Secret file
   - **Scope:** Global
   - **File:** Upload your `~/.kube/config` file
   - **ID:** `kubeconfig`
   - **Description:** EKS kubeconfig
5. Click **Create**

### Configure Security

Navigate to **Manage Jenkins → Security**:

- Enable **CSRF Protection**
- Set authorization to **Role-Based Strategy** (with Role Strategy plugin)
- Disable "Allow users to sign up" if not needed

---

## Required Tools & Plugins

### Plugins to Install

Navigate to **Manage Jenkins → Plugins → Available plugins**:

| Plugin | Purpose |
|--------|---------|
| **Pipeline** | Core pipeline support |
| **Multibranch Pipeline** | Auto-discover branches |
| **Git** | Git SCM integration |
| **GitHub Integration** | Webhooks & PR status |
| **NodeJS** | Node.js build environment |
| **Docker Pipeline** | Docker build/push DSL in pipelines |
| **Docker** | Docker agent support |
| **Kubernetes CLI** | Provides kubectl binary + `withKubeConfig` step |
| **Pipeline: AWS Steps** | `withAWS` credential injection |
| **AWS Credentials** | AWS credential type in Jenkins |
| **Credentials Binding** | Use credentials in pipelines |
| **Blue Ocean** | Modern pipeline UI |
| **Workspace Cleanup** | Clean workspace between builds |
| **Slack Notification** | Send build notifications to Slack |

### Tools to Install on Jenkins Server

```bash
# Git (required)
sudo dnf install git -y

# Docker (required - plugin only provides DSL, not the daemon)
sudo dnf install docker -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# AWS CLI v2 (required - plugin only injects creds, not the binary)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

> **Note:** Node.js and kubectl do NOT need to be installed on the server — their plugins handle installation automatically.

---

## Understanding Jenkins Plugins vs Server Installs

One of the most common questions when setting up Jenkins is: "Can I just install a plugin and use the tool, or do I need to install it on the server too?" This section explains exactly what each plugin provides and what still requires a manual server installation.

### The Key Concept

Jenkins plugins fall into two categories:

1. **Plugins that install the tool for you** — These download and manage the binary automatically. You configure them in **Manage Jenkins → Tools** and reference them in your pipeline. No server-side installation needed.

2. **Plugins that only provide pipeline DSL or credential injection** — These give you convenient pipeline syntax (like `docker.build()` or `withAWS()`), but they expect the actual tool binary to already exist on the server.

Understanding this difference saves hours of debugging "command not found" errors.

---

### Tutorial: NodeJS Plugin (Installs Node.js Automatically)

The **NodeJS** plugin downloads and manages Node.js versions for you. You never need to install Node.js on the Jenkins server.

**Step 1:** Install the plugin
- Go to **Manage Jenkins → Plugins → Available plugins**
- Search for "NodeJS" and install it

**Step 2:** Configure a Node.js installation
- Go to **Manage Jenkins → Tools**
- Scroll to **NodeJS installations**
- Click **Add NodeJS**
- Name: `NodeJS-18`
- Check "Install automatically"
- Select version: `18.x`
- Click **Save**

**Step 3:** Use in your pipeline

```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-18'   // Jenkins auto-installs Node.js 18 for this job
    }
    stages {
        stage('Build') {
            steps {
                sh 'node --version'   // Works without server install
                sh 'npm ci'
                sh 'npm run build'
            }
        }
    }
}
```

**How it works:** When the pipeline runs, Jenkins checks if Node.js 18 is already downloaded. If not, it downloads it to the Jenkins tools directory and adds it to the PATH for that job. Each job gets an isolated tool installation.

---

### Tutorial: Kubernetes CLI Plugin (Installs kubectl Automatically)

The **Kubernetes CLI** plugin provides the `kubectl` binary AND the `withKubeConfig` pipeline step. You do NOT need to install kubectl on the server.

**Step 1:** Install the plugin
- Go to **Manage Jenkins → Plugins → Available plugins**
- Search for "Kubernetes CLI" and install it

**Step 2:** Add your kubeconfig as a credential
- Go to **Manage Jenkins → Credentials → System → Global credentials**
- Click **Add Credentials**
- Kind: **Secret file**
- File: Upload your `~/.kube/config`
- ID: `kubeconfig`
- Click **Create**

> To generate the kubeconfig locally:
> ```bash
> aws eks update-kubeconfig --name landmark-eks --region us-east-1
> cat ~/.kube/config
> ```

**Step 3:** Use in your pipeline

```groovy
stage('Deploy to Kubernetes') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh 'kubectl get nodes'              // kubectl is available automatically
            sh 'kubectl apply -f k8s/namespace.yml'
            sh 'kubectl apply -f k8s/'
        }
    }
}
```

**How it works:** The plugin bundles kubectl and makes it available inside the `withKubeConfig` block. It also sets up the `KUBECONFIG` environment variable pointing to your uploaded kubeconfig file, so kubectl knows which cluster to talk to.

**Alternative — Dynamic kubeconfig (no static file):**

If you prefer not to store a kubeconfig file (since EKS tokens expire), you can generate it at runtime using AWS CLI + the Kubernetes CLI plugin:

```groovy
stage('Deploy') {
    steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
            sh 'aws eks update-kubeconfig --name landmark-eks --region us-east-1'
            sh 'kubectl apply -f k8s/'   // kubectl provided by the plugin
        }
    }
}
```

This approach requires AWS CLI on the server (see below) but avoids storing a static kubeconfig.

---

### Tutorial: Docker Pipeline Plugin (Does NOT Install Docker)

The **Docker Pipeline** plugin provides a clean Groovy DSL for building, pushing, and running Docker containers in your pipeline. However, it does **NOT** install Docker itself. The Docker daemon must already be running on the Jenkins server or agent.

**Why?** Docker is not just a CLI tool — it's a daemon (background service) that manages containers, images, networks, and volumes. A plugin cannot install and run a system daemon. It can only provide a nice API on top of an existing Docker installation.

**Step 1:** Install Docker on the Jenkins server

```bash
# Install Docker
sudo dnf install docker -y

# Start and enable the daemon
sudo systemctl start docker
sudo systemctl enable docker

# Allow Jenkins user to run Docker commands
sudo usermod -aG docker jenkins

# Restart Jenkins to pick up the group change
sudo systemctl restart jenkins
```

**Step 2:** Install the plugins
- Go to **Manage Jenkins → Plugins → Available plugins**
- Install both **Docker Pipeline** and **Docker** plugins

**Step 3:** Use in your pipeline

```groovy
// Example 1: Build and push an image to Docker Hub
stage('Docker Build & Push') {
    steps {
        script {
            // docker.withRegistry handles login/logout automatically
            // 'dockerhub-creds' is a Username+Password credential you created earlier
            docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                def app = docker.build("chafah/landmark-web-app:${IMAGE_TAG}")
                app.push()           // Pushes the tag
                app.push('latest')   // Also push as 'latest'
            }
        }
    }
}
```

```groovy
// Example 2: Run your build inside a Docker container (no Node.js needed on server)
pipeline {
    agent {
        docker {
            image 'node:18'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm ci && npm run build'
            }
        }
    }
}
```

**How it works:** Behind the scenes, `docker.build()` runs `docker build`, `docker.withRegistry()` runs `docker login` and `docker logout`, and `agent { docker {} }` runs `docker run` to spin up a container for your build. The plugin translates your Groovy code into Docker CLI commands — but the Docker CLI and daemon must exist.

---

### Tutorial: Pipeline AWS Steps Plugin (Does NOT Install AWS CLI)

The **Pipeline: AWS Steps** plugin provides `withAWS()` which injects AWS credentials (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`) as environment variables into your pipeline. It does **NOT** provide the `aws` command.

**Why?** The AWS CLI is a Python-based tool with many dependencies. The plugin focuses on securely injecting credentials rather than managing a complex tool installation.

**Step 1:** Install AWS CLI on the server

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

**Step 2:** Install the plugins
- Install **Pipeline: AWS Steps** and **AWS Credentials** plugins

**Step 3:** Add AWS credentials
- Go to **Manage Jenkins → Credentials → Add Credentials**
- Kind: **AWS Credentials**
- ID: `aws-creds`
- Fill in Access Key ID and Secret Access Key

**Step 4:** Use in your pipeline

```groovy
stage('Deploy') {
    steps {
        // withAWS sets AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and AWS_DEFAULT_REGION
        // but the 'aws' command must already be installed on the server
        withAWS(credentials: 'aws-creds', region: 'us-east-1') {
            sh 'aws eks update-kubeconfig --name landmark-eks --region us-east-1'
            sh 'kubectl apply -f k8s/'
        }
    }
}
```

**How it works:** `withAWS()` reads the credentials from Jenkins credential store and exports them as environment variables for the duration of that block. Any `sh` commands inside the block can use `aws` CLI, and it will automatically authenticate using those injected credentials. When the block ends, the credentials are removed from the environment.

---

### Summary Table

| Tool | Plugin | Installs Binary? | Server Install Needed? | Pipeline Step |
|------|--------|:----------------:|:---------------------:|---------------|
| Node.js | **NodeJS** | ✅ Yes | ❌ No | `tools { nodejs 'NodeJS-18' }` |
| kubectl | **Kubernetes CLI** | ✅ Yes | ❌ No | `withKubeConfig([credentialsId: '...'])` |
| Docker | **Docker Pipeline** | ❌ No | ✅ Yes (daemon) | `docker.build()`, `docker.withRegistry()` |
| AWS CLI | **Pipeline: AWS Steps** | ❌ No | ✅ Yes (binary) | `withAWS(credentials: '...')` |
| Git | Built-in | ❌ No | ✅ Yes | `checkout scm` |

---

### Minimum Server Installation for This Project

Based on the table above, here's what you need to install on the Jenkins server:

```bash
# 1. Git — required for checking out code
sudo dnf install git -y

# 2. Docker — required because the plugin only provides DSL, not the daemon
sudo dnf install docker -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# 3. AWS CLI — required because the plugin only injects credentials
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**You do NOT need to install:**
- Node.js — the **NodeJS** plugin handles it
- kubectl — the **Kubernetes CLI** plugin handles it

---

## Pipeline Types & Syntax

Jenkins supports two types of pipelines:

### 1. Declarative Pipeline (Recommended)

Structured, opinionated syntax with a predefined structure. Easier to read and write.

```groovy
pipeline {
    agent any

    environment {
        MY_VAR = 'value'
    }

    stages {
        stage('Build') {
            steps {
                sh 'echo Building...'
            }
        }
        stage('Test') {
            steps {
                sh 'echo Testing...'
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'echo Deploying...'
            }
        }
    }

    post {
        success { echo 'Build passed!' }
        failure { echo 'Build failed!' }
        always  { cleanWs() }
    }
}
```

**Key directives:**

| Directive | Purpose |
|-----------|---------|
| `pipeline` | Root block |
| `agent` | Where to run (any, none, docker, label) |
| `environment` | Environment variables |
| `stages` | Contains all stages |
| `stage` | A named phase of the pipeline |
| `steps` | Actual commands to execute |
| `when` | Conditional execution |
| `post` | Actions after pipeline completes |

### 2. Scripted Pipeline

Full Groovy scripting. More flexible but harder to maintain.

```groovy
node {
    stage('Build') {
        checkout scm
        sh 'npm ci'
        sh 'npm run build'
    }

    stage('Test') {
        sh 'npm test'
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh 'echo Deploying to production...'
        }
    }
}
```

### Declarative vs Scripted

| Feature | Declarative | Scripted |
|---------|-------------|----------|
| Syntax | Structured | Free-form Groovy |
| Learning curve | Low | High |
| Error handling | Built-in `post` block | try/catch |
| Conditionals | `when` directive | `if/else` |
| Recommended for | Most use cases | Complex logic |

---

## Creating a Pipeline for This Application

### Option 1: Multibranch Pipeline (Recommended)

This auto-discovers branches and runs the `Jenkinsfile` from the repo.

**Setup steps:**

1. Go to **Jenkins Dashboard → New Item**
2. Enter name: `landmark-web-app`
3. Select **Multibranch Pipeline** → OK
4. Under **Branch Sources**:
   - Add source → **GitHub**
   - Set credentials (GitHub token)
   - Repository URL: `https://github.com/CHAFAH/landmark-web-app.git`
5. Under **Build Configuration**:
   - Mode: by Jenkinsfile
   - Script Path: `Jenkinsfile`
6. Under **Scan Multibranch Pipeline Triggers**:
   - Check "Periodically if not otherwise run" → interval: 1 minute (or use webhooks)
7. Click **Save**

Jenkins will scan all branches and trigger builds based on the `Jenkinsfile`.

### Option 2: Regular Pipeline Job

1. Go to **Jenkins Dashboard → New Item**
2. Enter name: `landmark-deploy`
3. Select **Pipeline** → OK
4. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: Git
   - Repository URL: `https://github.com/CHAFAH/landmark-web-app.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Click **Save**

### The Jenkinsfile

The `Jenkinsfile` at the project root defines the full CI/CD pipeline:

```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-18'
    }
    environment {
        DOCKER_REPO = 'chafah/landmark-web-app'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'landmark-eks'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install & Test') {
            steps {
                sh 'npm ci'
                sh 'npm test'
                sh 'cd server && npm ci && npm test'
            }
        }
        stage('Build Frontend') {
            steps { sh 'npm run build' }
        }
        stage('Generate Image Tag') {
            steps {
                script {
                    def branch = env.BRANCH_NAME.replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}"
                }
            }
        }
        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'develop'
                    branch pattern: 'release*', comparator: 'GLOB'
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                }
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("${DOCKER_REPO}:${IMAGE_TAG}")
                        app.push()
                    }
                }
            }
        }
        stage('Deploy to Dev') {
            when { branch 'develop' }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: develop/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: develop/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
        stage('Deploy to Staging') {
            when { branch pattern: 'release*', comparator: 'GLOB' }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: staging/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: staging/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
        stage('Deploy to Production') {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                }
            }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: production/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: production/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
        always  { cleanWs() }
    }
}
```

### Pipeline Flow

```
┌──────────┐    ┌──────────────┐    ┌───────────┐    ┌─────────────┐    ┌──────────────────┐    ┌────────────┐
│ Checkout │───▶│ Install/Test │───▶│ Build App │───▶│ Docker Push │───▶│ Deploy to K8s    │───▶│  Success   │
└──────────┘    └──────────────┘    └───────────┘    └─────────────┘    └──────────────────┘    └────────────┘
                                                      (deploy branches)  (branch-specific ns)
```

---

## Advanced Jenkins

### Shared Libraries

Shared libraries let you reuse pipeline code across multiple projects.

**Setup:**

1. Create a repo with this structure:
   ```
   vars/
     deployToK8s.groovy
   src/
     org/landmark/Utils.groovy
   ```

2. Go to **Manage Jenkins → System → Global Pipeline Libraries**:
   - Name: `landmark-shared`
   - Default version: `main`
   - SCM: Git → your shared library repo

3. Use in Jenkinsfile:
   ```groovy
   @Library('landmark-shared') _

   pipeline {
       stages {
           stage('Deploy') {
               steps {
                   deployToK8s(namespace: 'production', image: env.IMAGE_TAG)
               }
           }
       }
   }
   ```

### Parallel Stages

Run stages in parallel to speed up the pipeline:

```groovy
stage('Tests') {
    parallel {
        stage('Frontend Tests') {
            steps { sh 'npm test' }
        }
        stage('Server Tests') {
            steps { sh 'cd server && npm test' }
        }
    }
}
```

### Build Agents (Distributed Builds)

Scale Jenkins by adding build agents:

```groovy
pipeline {
    agent { label 'docker-agent' }
    // ...
}
```

**Agent setup:**
1. Go to **Manage Jenkins → Nodes → New Node**
2. Set remote root directory, labels, and launch method
3. Use SSH or JNLP to connect the agent

### Docker Agent

Run stages inside a Docker container:

```groovy
pipeline {
    agent {
        docker {
            image 'node:18'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    stages {
        stage('Build') {
            steps { sh 'npm ci && npm run build' }
        }
    }
}
```

### Kubernetes Agent (Dynamic Pods)

Use the Kubernetes plugin to spin up build agents as pods:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: node
                    image: node:18
                    command: ['sleep', '3600']
                  - name: docker
                    image: docker:dind
                    securityContext:
                      privileged: true
            """
        }
    }
    stages {
        stage('Build') {
            container('node') {
                steps { sh 'npm ci && npm run build' }
            }
        }
    }
}
```

### Webhooks (Auto-trigger on Push)

1. In GitHub repo → **Settings → Webhooks → Add webhook**:
   - Payload URL: `http://<jenkins-url>:8080/github-webhook/`
   - Content type: `application/json`
   - Events: "Just the push event"

2. In Jenkins job → **Build Triggers** → check "GitHub hook trigger for GITScm polling"

### Input / Manual Approval

Gate production deployments with manual approval:

```groovy
stage('Approve Production Deploy') {
    when { branch 'main' }
    steps {
        input message: 'Deploy to production?', ok: 'Deploy'
    }
}
```

### Notifications

#### Slack

```groovy
post {
    success {
        slackSend channel: '#deployments', color: 'good',
            message: "✅ ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded"
    }
    failure {
        slackSend channel: '#deployments', color: 'danger',
            message: "❌ ${env.JOB_NAME} #${env.BUILD_NUMBER} failed"
    }
}
```

#### Email

```groovy
post {
    failure {
        emailext subject: "Build Failed: ${env.JOB_NAME}",
            body: "Check: ${env.BUILD_URL}",
            to: 'team@landmark.com'
    }
}
```

### Pipeline Caching

Speed up builds by caching `node_modules`:

```groovy
stage('Install') {
    steps {
        cache(caches: [
            arbitraryFileCache(path: 'node_modules', cacheValidityDecidingFile: 'package-lock.json')
        ]) {
            sh 'npm ci'
        }
    }
}
```

### Credentials Best Practices

- Never hardcode secrets in Jenkinsfile
- Use `withCredentials` block to inject secrets only where needed
- Scope credentials to specific folders/jobs when possible
- Rotate credentials regularly
- Use IAM roles for EC2 instead of access keys when possible

### Backup Jenkins

```bash
# Backup Jenkins home directory
sudo tar -czf jenkins-backup-$(date +%Y%m%d).tar.gz /var/lib/jenkins/

# Key directories to back up:
# /var/lib/jenkins/config.xml        - Main config
# /var/lib/jenkins/credentials.xml   - Credentials
# /var/lib/jenkins/jobs/             - Job configs & build history
# /var/lib/jenkins/plugins/          - Installed plugins
```

---

## Quick Reference

| Task | Location |
|------|----------|
| Install plugins | Manage Jenkins → Plugins |
| Add credentials | Manage Jenkins → Credentials |
| Configure tools | Manage Jenkins → Tools |
| View build logs | Job → Build # → Console Output |
| Replay pipeline | Job → Build # → Replay |
| Pipeline syntax helper | Job → Pipeline Syntax (sidebar) |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `docker: permission denied` | Add jenkins to docker group: `sudo usermod -aG docker jenkins` then restart Jenkins |
| `npm: command not found` | Install NodeJS plugin and configure in Tools, or install Node on the server |
| `kubectl: command not found` | Install **Kubernetes CLI** plugin — it provides kubectl automatically |
| `aws: command not found` | Install AWS CLI on the server — the AWS plugin only injects credentials |
| Pipeline not triggering | Check webhook delivery in GitHub, verify Jenkins URL is accessible |
| Agent offline | Check agent logs, verify network connectivity and Java version |

---

## References

- [Jenkins Official Docs](https://www.jenkins.io/doc/)
- [Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins Plugin Index](https://plugins.jenkins.io/)
- [Blue Ocean Documentation](https://www.jenkins.io/doc/book/blueocean/)
