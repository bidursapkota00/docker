# Jenkins Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Jenkins](#introducing-jenkins)
2. [Installation & Setup](#installation--setup)
3. [Core Concepts](#core-concepts)
4. [Pipeline Basics](#pipeline-basics)
5. [Declarative Pipeline Syntax](#declarative-pipeline-syntax)
6. [Scripted Pipeline Syntax](#scripted-pipeline-syntax)
7. [Environment Variables](#environment-variables)
8. [Credentials Management](#credentials-management)
9. [Jenkins with Docker](#jenkins-with-docker)
10. [Webhooks & Triggers](#webhooks--triggers)
11. [Shared Libraries](#shared-libraries)
12. [Administration & Maintenance](#administration--maintenance)
13. [Best Practices](#best-practices)

---

## Introducing Jenkins

Jenkins is a free, open-source automation server that helps automate the parts of software development related to building, testing, and deploying, facilitating Continuous Integration (CI) and Continuous Delivery (CD). Written in Java, Jenkins has a massive ecosystem of plugins that allow it to integrate with virtually every tool in the CI/CD toolchain.

Jenkins is commonly used to automatically build code when it is pushed to a repository, run unit and integration tests, perform static code analysis, and deploy the application to various environments (development, staging, production) if all checks pass.

Core concepts of Jenkins are:

- **Controller (Master)**: The central Jenkins server that handles scheduling build jobs, dispatching builds to agents, and monitoring the agents.
- **Agent (Node/Slave)**: A machine that connects to the Jenkins controller and executes the tasks instructed by the controller.
- **Job/Project**: A user-configured description of work which Jenkins should perform, such as building a software project.
- **Pipeline**: A suite of plugins that supports implementing and integrating continuous delivery pipelines into Jenkins. It's defined via code (Pipeline as Code).
- **Jenkinsfile**: A text file that contains the definition of a Jenkins Pipeline and is checked into source control.
- **Plugin**: An extension to Jenkins functionality. Jenkins relies heavily on plugins for integrations with Git, Docker, AWS, and thousands of other tools.

---

## Installation & Setup

### Install Jenkins

**Using Docker** (recommended for local use):

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

`-p 8080:8080` maps the web interface. `-p 50000:50000` is used by inbound Jenkins agents. `-v jenkins_home:/var/jenkins_home` persists Jenkins data (plugins, jobs, configuration) across container restarts.

**Linux (Ubuntu/Debian)**:

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

### Initial Setup

1. Open `http://localhost:8080` in your browser.
2. Retrieve the initial admin password:
   - Docker: `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`
   - Linux: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
3. Enter the password in the UI.
4. Select **Install suggested plugins** (best for beginners).
5. Create your first admin user.

### Managing Plugins

Navigate to **Manage Jenkins → Plugins**.
- **Available plugins**: Search and install new plugins (e.g., Docker Pipeline, Blue Ocean, GitHub Integration).
- **Installed plugins**: View, update, or disable current plugins.

Jenkins will usually require a restart after installing plugins. Check "Restart Jenkins when installation is complete and no jobs are running".

---

## Core Concepts

### Jobs and Projects

- **Freestyle Project**: The classic way to configure Jenkins jobs through the UI. Good for simple scripts but hard to version control.
- **Pipeline**: Defines the build process as code (Jenkinsfile). Highly recommended for all modern Jenkins usage.
- **Multibranch Pipeline**: Automatically creates a Pipeline for each branch in your source control repository that contains a Jenkinsfile.

### Distributed Builds (Controller and Agents)

Jenkins uses a distributed architecture to handle heavy workloads. The Controller orchestrates, while Agents execute the builds.

To add an Agent:
1. Go to **Manage Jenkins → Nodes → New Node**.
2. Give it a name and select **Permanent Agent**.
3. Configure `# of executors` (how many concurrent builds it can run).
4. Set the **Remote root directory** (where the agent stores workspaces).
5. Specify **Labels** (used to route jobs to specific agents, e.g., `linux`, `docker`, `windows`).
6. Select the **Launch method** (usually SSH).

---

## Pipeline Basics

A Jenkins Pipeline is a suite of plugins that supports implementing and integrating continuous delivery pipelines. The definition of a Jenkins Pipeline is written into a text file (called a `Jenkinsfile`), which can be committed to a project's source control repository.

There are two syntaxes for defining a Pipeline: **Declarative** and **Scripted**.

### Declarative vs. Scripted

| Feature | Declarative Pipeline | Scripted Pipeline |
| :--- | :--- | :--- |
| **Syntax** | Stricter, predefined structure | Groovy-based, very flexible |
| **Learning Curve** | Easier, readable | Steeper, requires Groovy knowledge |
| **Error Checking** | Validated at syntax check phase | Validated at runtime |
| **Use Case** | Recommended for most modern pipelines | Complex logic, legacy pipelines |

---

## Declarative Pipeline Syntax

Declarative pipelines start with the `pipeline` block and define stages and steps in a structured manner.

### Basic Structure

```groovy
// Jenkinsfile
pipeline {
    agent any // Run on any available agent

    stages {
        stage('Build') {
            steps {
                echo 'Building the application...'
                sh 'make build' // Run a shell command
            }
        }
        stage('Test') {
            steps {
                echo 'Running unit tests...'
                sh 'make test'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying to staging...'
                sh 'make deploy'
            }
        }
    }
}
```

`agent any` tells Jenkins to allocate an executor and workspace on any available node. `stages` contains a sequence of one or more `stage` directives, which represent phases of the pipeline. `steps` defines the actual tasks to perform. `sh` executes a Unix shell command (`bat` for Windows).

### Agent Directives

You can restrict where a pipeline or stage runs using the `agent` directive.

```groovy
pipeline {
    agent none // Must define agent at the stage level

    stages {
        stage('Linux Build') {
            agent { label 'linux' } // Run on agent labeled 'linux'
            steps {
                sh 'uname -a'
            }
        }
        stage('Docker Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    label 'docker-ready-node'
                }
            }
            steps {
                sh 'node --version' // Runs inside the Node container
            }
        }
    }
}
```

Using `docker` as an agent runs the specified steps inside a spun-up Docker container, automatically mounting the workspace. This isolates build environments perfectly.

### Post Conditions

The `post` section defines actions to take at the end of a pipeline or a specific stage, based on the outcome.

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
    post {
        always {
            echo 'This will always run.'
            cleanWs() // Clean the workspace
        }
        success {
            echo 'This runs if the pipeline succeeds.'
        }
        failure {
            echo 'This runs if the pipeline fails.'
            mail to: 'team@example.com',
                 subject: "Failed Pipeline: ${env.JOB_NAME}",
                 body: "Something is wrong with ${env.BUILD_URL}"
        }
        unstable {
            echo 'This runs if the pipeline is marked unstable (e.g., test failures).'
        }
    }
}
```

### When Directives

The `when` directive allows the pipeline to determine whether the stage should be executed depending on the given condition.

```groovy
stage('Deploy to Prod') {
    when {
        branch 'main' // Only run if the branch is 'main'
        // Other conditions:
        // environment name: 'DEPLOY_TO', value: 'production'
        // tag "release-*"
        // expression { return params.DO_DEPLOY == true }
    }
    steps {
        sh 'deploy-to-prod.sh'
    }
}
```

### Parallel Execution

You can run stages concurrently to speed up the pipeline.

```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'npm run test:unit'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'npm run test:integration'
            }
        }
        stage('E2E Tests') {
            steps {
                sh 'npm run test:e2e'
            }
        }
    }
}
```

---

## Scripted Pipeline Syntax

Scripted Pipelines are executed sequentially from top to bottom, using Groovy control structures like `if/else`, `try/catch`, and `for` loops. They start with a `node` block.

```groovy
node {
    stage('Checkout') {
        checkout scm
    }
    
    stage('Build & Test') {
        try {
            sh 'make build'
            sh 'make test'
        } catch (Exception e) {
            currentBuild.result = 'FAILURE'
            echo "Failed: ${e.message}"
            throw e
        }
    }
    
    stage('Deploy') {
        if (env.BRANCH_NAME == 'main') {
            sh 'make deploy'
        } else {
            echo "Skipping deploy for branch ${env.BRANCH_NAME}"
        }
    }
}
```

`node` allocates a workspace and executor. Scripted pipelines offer more programmatic control but lack the built-in declarative features like `post` or `when` blocks, requiring manual implementation via `try/catch` or `if` statements.

---

## Environment Variables

### Built-in Variables

Jenkins provides several built-in variables accessible via the `env` object.

```groovy
pipeline {
    agent any
    steps {
        echo "Job Name: ${env.JOB_NAME}"
        echo "Build Number: ${env.BUILD_NUMBER}"
        echo "Build URL: ${env.BUILD_URL}"
        echo "Workspace: ${env.WORKSPACE}"
        echo "Git Commit: ${env.GIT_COMMIT}"
        echo "Branch Name: ${env.BRANCH_NAME}" // In Multibranch pipelines
    }
}
```

### Custom Variables

You can define custom environment variables in the `environment` block.

```groovy
pipeline {
    agent any
    environment {
        APP_NAME = 'my-awesome-app'
        PORT = '8080'
        // Evaluate dynamic values
        BUILD_TIME = sh(script: 'date -Iseconds', returnStdout: true).trim()
    }
    stages {
        stage('Build') {
            steps {
                sh "echo Building ${APP_NAME} on port ${PORT} at ${BUILD_TIME}"
                // Variables are also available to shell scripts directly
                sh 'echo Building $APP_NAME' 
            }
        }
    }
}
```

The `environment` block can be placed at the `pipeline` level (applies to all stages) or within a specific `stage` (applies only to that stage).

---

## Credentials Management

Jenkins securely stores credentials (passwords, SSH keys, API tokens) and allows you to inject them into your pipeline without exposing them in logs or source code.

### Adding Credentials

1. Go to **Manage Jenkins → Credentials → System → Global credentials (unrestricted)**.
2. Click **Add Credentials**.
3. Select the **Kind** (e.g., Secret text, Username with password, SSH Username with private key).
4. Provide the value and assign an **ID** (this ID is used in the Jenkinsfile).

### Using Credentials in Pipeline

Use the `credentials()` helper method within the `environment` block.

```groovy
pipeline {
    agent any
    environment {
        // Binds the username and password to variables
        DOCKER_CREDS = credentials('docker-hub-credentials-id')
        // Binds a secret text (like an API token)
        AWS_ACCESS_KEY = credentials('aws-access-key-id')
    }
    stages {
        stage('Docker Login') {
            steps {
                // Jenkins creates DOCKER_CREDS_USR and DOCKER_CREDS_PSW automatically
                sh 'docker login -u $DOCKER_CREDS_USR -p $DOCKER_CREDS_PSW'
            }
        }
        stage('Deploy') {
            steps {
                // Use the secret text
                sh 'aws s3 sync ./build s3://my-bucket/ --access-key $AWS_ACCESS_KEY'
            }
        }
    }
}
```

For older or specific use cases, you can also use the `withCredentials` step:

```groovy
steps {
    withCredentials([usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')]) {
        sh 'mysql -u $DB_USER -p$DB_PASS -h db.example.com mydb < schema.sql'
    }
}
```

---

## Jenkins with Docker

Using Docker with Jenkins solves the "it works on my machine" problem for CI/CD environments.

### Docker in the Agent Directive

You can define the build environment completely via Docker.

```groovy
pipeline {
    agent {
        docker {
            image 'maven:3.8.1-jdk-11'
            args '-v $HOME/.m2:/root/.m2' // Cache maven dependencies
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package' // Runs inside the maven container
            }
        }
    }
}
```

### Building Docker Images

You can build and publish Docker images from within the pipeline using the `docker` global variable (provided by the Docker Pipeline plugin).

```groovy
pipeline {
    agent any
    environment {
        IMAGE_NAME = 'myorg/myapp'
        DOCKER_CREDS = credentials('dockerhub')
    }
    stages {
        stage('Build Image') {
            steps {
                script {
                    // Build the image
                    customImage = docker.build("${IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }
        }
        stage('Push Image') {
            steps {
                script {
                    // Login and push
                    sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
                    customImage.push()
                    customImage.push('latest')
                }
            }
        }
    }
    post {
        always {
            // Clean up local images
            sh "docker rmi ${IMAGE_NAME}:${env.BUILD_NUMBER} || true"
        }
    }
}
```

Note: Running Docker commands inside a Jenkins agent often requires the Jenkins user to be in the `docker` group on the host, or using "Docker out of Docker" (mounting `/var/run/docker.sock`).

---

## Webhooks & Triggers

To make Jenkins build automatically when code changes, configure webhooks in your SCM (GitHub, GitLab, Bitbucket).

### Configuring GitHub Webhooks

1. Install the **GitHub Integration Plugin** in Jenkins.
2. In your Jenkins job configuration, check **GitHub hook trigger for GITScm polling**.
3. In your GitHub repository, go to **Settings → Webhooks → Add webhook**.
4. Payload URL: `http://<jenkins-url>/github-webhook/` (Ensure the trailing slash is present).
5. Content type: `application/json`.
6. Select the events you want to trigger builds (usually `push` and `pull request`).

### Pipeline Triggers Directive

You can also define triggers in the Jenkinsfile.

```groovy
pipeline {
    agent any
    triggers {
        // Run at 02:00 AM every day
        cron('0 2 * * *')
        // Poll SCM every 15 minutes (less efficient than webhooks)
        pollSCM('H/15 * * * *')
        // Trigger if an upstream project builds successfully
        upstream(upstreamProjects: 'base-image-job', threshold: hudson.model.Result.SUCCESS)
    }
    stages {
        // ...
    }
}
```

---

## Shared Libraries

As you adopt Jenkins across many projects, you'll end up copying and pasting pipeline logic. Shared Libraries allow you to write Groovy code once and use it across multiple Jenkinsfiles.

### Library Structure

Create a separate Git repository for your library.

```
(root)
+- src                     # Groovy source files
|   +- org
|       +- foo
|           +- Utilities.groovy
+- vars                    # Global variables (custom steps)
|   +- buildDockerImage.groovy
|   +- standardPipeline.groovy
+- resources               # Non-Groovy files (JSON, scripts, etc.)
```

### Defining a Custom Step

```groovy
// vars/buildDockerImage.groovy
def call(String imageName) {
    echo "Building Docker image: ${imageName}"
    sh "docker build -t ${imageName} ."
}
```

### Using the Library

Configure the library in **Manage Jenkins → System → Global Pipeline Libraries**.

Then, use it in your Jenkinsfile:

```groovy
// Import the library using its configured name
@Library('my-shared-library') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                // Call the custom step defined in vars/buildDockerImage.groovy
                buildDockerImage('myorg/myapp')
            }
        }
    }
}
```

---

## Administration & Maintenance

### Jenkins CLI

Jenkins provides a command-line interface via a JAR file.

```bash
# Download the CLI jar
wget http://localhost:8080/jnlpJars/jenkins-cli.jar

# Run a command (e.g., list jobs)
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth admin:apitoken list-jobs
```

### Backup and Restore

Always back up the `JENKINS_HOME` directory. It contains everything: configurations, job definitions, plugins, and build history.

- If using Docker, back up the mounted volume.
- Use the **ThinBackup** plugin to schedule backups of configurations and jobs without stopping Jenkins.

### Viewing Logs

- System Logs: **Manage Jenkins → System Log**.
- Job Logs: Click the build number, then **Console Output**.
- Docker logs (if running in container): `docker logs -f jenkins`

---

## Best Practices

- **Pipeline as Code**: Always use Jenkinsfiles checked into source control. Avoid Freestyle jobs.
- **Multibranch Pipelines**: Use them to automatically manage CI/CD for feature branches and PRs.
- **Use Docker Agents**: Define your build environments in Docker containers via the `agent { docker { ... } }` directive to avoid installing tools directly on Jenkins nodes.
- **Do not run builds on the Controller**: The controller should only orchestrate. Allocate agents to run actual jobs. Set the controller's `# of executors` to 0.
- **Keep Jenkinsfiles Clean**: If your Jenkinsfile is getting too long or complex, move the logic into a shell script or a Shared Library. Jenkinsfiles should coordinate, not script.
- **Secure Credentials**: Never hardcode passwords or API keys. Always use the Jenkins Credentials plugin.
- **Clean Workspace**: Use `cleanWs()` in the `post { always { ... } }` block to ensure subsequent builds aren't affected by leftover files.
- **Limit Build History**: Configure the job to discard old builds to prevent `JENKINS_HOME` from filling up the disk.
- **Monitor Jenkins**: Monitor disk space, CPU, and memory of the Jenkins controller and agents. Out-of-disk-space errors are common.
