# Master the Art of Secure DevOps
### Learn How to Configure a Secure CI Pipeline on Ubuntu 22.04 LTS

This guide provides step-by-step instructions for setting up Jenkins, Docker Engine, Trivy, and SonarQube, followed by creating a secure CI pipeline in Jenkins. These tools will help ensure automated vulnerability scans, code quality checks, and secure software delivery.

---

## Table of Contents
1. [Prerequisites](#prerequisites)  
2. [Install Jenkins](#1-install-jenkins)  
    - [Install Java](#installation-of-java)  
    - [Install Jenkins](#install-jenkins)  
    - [Accessing Jenkins](#accessing-jenkins)  
    - [Install Required Plugins](#install-required-jenkins-plugins)  
3. [Install Docker Engine on Ubuntu](#2-install-docker-engine-on-ubuntu)  
    - [Set Up Docker’s apt Repository](#set-up-dockers-apt-repository)  
    - [Install Docker Packages](#install-the-docker-packages)  
    - [Post-installation Steps for Docker Engine](#post-installation-steps-for-docker-engine)  
4. [Install Trivy](#3-install-trivy)  
    - [Download Trivy Database](#download-the-trivy-database)  
5. [SonarQube Installation](#4-sonarqube-installation)  
    - [Access SonarQube](#access-sonarqube)  
    - [Set Up SonarQube in Jenkins](#setup-sonarqube-in-jenkins)  
6. [Node.js Installation](#5-node-js-installation)  
7. [References](#references)  

---

## Prerequisites
Ensure you meet these requirements before proceeding:  
- Operating System: **Ubuntu 22.04 LTS Server or higher**  
- A user account with `sudo` privileges  
- Basic understanding of Linux commands  

---

## 1. Install Jenkins  

<img src="https://upload.wikimedia.org/wikipedia/commons/e/e3/Jenkins_logo_with_title.svg" alt="Jenkins" width="25%" height="100%">

### Installation of Java  
Jenkins requires Java to run. Update the system repositories, install OpenJDK 17, and verify the installation:  

```bash

# Update the package list from all configured repositories
sudo apt update

# Install the required packages:
# - fontconfig: Ensures proper font rendering (used by some tools and Java applications)
# - openjdk-21-jre: Installs the OpenJDK 21 Runtime Environment (needed for running Java applications, like Jenkins)
sudo apt install fontconfig openjdk-21-jre

# Verify the installed Java version to ensure OpenJDK 21 is properly installed
java -version

```

### Install Jenkins
Add the Jenkins repository and install Jenkins:

```bash

# Download and add the Jenkins GPG key to your system's trusted keyring.
# This ensures that the packages you download from the Jenkins repository are authentic and have not been tampered with.
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian/jenkins.io-2023.key

# Add the Jenkins repository to the system's package sources list.
# The "deb" line specifies the repository URL and its components.
# We use the `signed-by` option to associate the GPG key with this repository for secure installation.
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update the package list to include the new Jenkins repository.
# This allows the system to recognize and fetch Jenkins-related packages from the newly added repository.
sudo apt-get update

# Install Jenkins from the official Jenkins repository.
# This will install the latest version of Jenkins along with its dependencies.
sudo apt-get install jenkins

```
Enable, start, and check the Jenkins service status:

```bash

# Enable the Jenkins service to start automatically at system boot.
# This ensures Jenkins will be started every time the server reboots.
sudo systemctl enable jenkins

# Start the Jenkins service immediately.
# This will launch the Jenkins application so you can access it on its default port (8080 or custom port if configured).
sudo systemctl start jenkins

# Check the current status of the Jenkins service.
# This command provides information such as whether Jenkins is running, its PID (process ID), and possible errors if it's not running.
sudo systemctl status jenkins

```

If Jenkins fails to start due to a port conflict, edit the Jenkins configuration file to specify a different port:

```bash

# Open the full configuration file for the Jenkins service in a text editor.
# This allows you to modify the service settings, such as environment variables or startup options.
sudo systemctl edit jenkins --full

# Add the following lines to the opened configuration file:
# These lines change the default port used by Jenkins from 8080 to 8081 (or any custom port you prefer).
# ---------------------------------
[Service]
Environment="JENKINS_PORT=8081"
# ---------------------------------

# Restart the Jenkins service to apply the changes made to the configuration file.
# This ensures Jenkins starts on the new port specified in the configuration.
sudo systemctl restart jenkins

```

### Accessing Jenkins
Visit Jenkins at `http://<your_server_ip>:8080` (or the custom port you specified). Use this command to retrieve the initial admin password:

```bash
# This command retrieves the initial admin password for Jenkins from the specified file.
# The password is required for the first-time setup of Jenkins.
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

```
### Install required Jenkins plugins
- Pipeline Stage View
- Docker
- Generic Webhook Trigger
- OWASP Dependency-Check
- SonarQube Scanner
- HTML Publisher

## 2. Install Docker Engine on Ubuntu

<img src="https://upload.wikimedia.org/wikipedia/commons/4/4e/Docker_%28container_engine%29_logo.svg" alt="Docker" width="25%" height="100%">


Set Up Docker’s apt Repository

```bash

# Update the package list from all configured repositories.
# This ensures that your system has access to the latest version of available software packages.
sudo apt-get update

# Install the required dependencies:
# - ca-certificates: Provides SSL/TLS certificates for secure communication.
# - curl: A command-line tool used to transfer data and download files over HTTP/HTTPS.
sudo apt-get install ca-certificates curl

# Create the directory `/etc/apt/keyrings` with the appropriate permissions (0755).
# This directory is used to securely store GPG keys for apt repositories.
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's official GPG key and save it in the `/etc/apt/keyrings` directory.
# The GPG key ensures that the Docker packages you download are authentic and have not been tampered with.
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Adjust the permissions of the Docker GPG key file so it can be read by the system.
# This step is necessary for apt to verify the authenticity of Docker's repository.
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker apt repository to your system's package sources list.
# The "deb" line specifies where to fetch Docker packages:
# - $(dpkg --print-architecture): Detects the CPU architecture automatically (e.g., amd64, arm64).
# - $(lsb_release -sc): Automatically detects the Ubuntu codename (e.g., jammy for Ubuntu 22.04).
# - signed-by=/etc/apt/keyrings/docker.asc: Ensures that this repository is verified using the previously downloaded Docker GPG key.
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -sc) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update the package list again to include the newly added Docker repository.
# This allows the system to recognize and fetch Docker-related packages from the official Docker repository.
sudo apt-get update

```

### Install the Docker packages
```bash

# Install Docker-related packages:

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```

### Post-installation steps for Docker Engine

```bash

# Create a new user group named "docker" (if it doesn't already exist).
# The "docker" group allows users within this group to run Docker commands without requiring `sudo`.
sudo groupadd docker

# Add the currently logged-in user ($USER) to the "docker" group.
# This grants the user permission to interact with the Docker daemon without using `sudo` for every command.
sudo usermod -aG docker $USER

# Add the Jenkins user to the "docker" group.
# This ensures that Jenkins (which runs under its own user account) can access the Docker daemon for building and running containers.
sudo usermod -aG docker jenkins

# Apply the updated group membership for the current user without needing to log out and back in.
# This step ensures the changes made by `usermod` take effect immediately in the current shell session.
newgrp docker

# Restart the Jenkins service to ensure it picks up the changes and allows Jenkins to use Docker.
# This step is necessary because Jenkins may not have access to the Docker group until it's restarted.
sudo systemctl restart jenkins

# Test the Docker installation by running the official "hello-world" container.
# If successful, this command downloads the "hello-world" image from Docker Hub and runs it in a container.
docker run hello-world

```

## 3. Install Trivy
<img src="https://object-storage.nz-hlz-1.catalystcloud.io/v1/AUTH_52213f2d28354f499d85ec4722164456/catalystcloudnz_django_storage_prod/images/Trivy-OSS-Logo-Color-Horizontal-RGB-2022.width-500.png" alt="Trivy" width="25%" height="100%">
Trivy is a vulnerability scanning tool. Install it as follows:

```bash

# Install required packages:
# - wget: A tool to download files from the web.
# - apt-transport-https: Enables the use of HTTPS for secure communication with package repositories.
# - gnupg: Used for adding and managing GPG keys for repository authentication.
# - lsb-release: Provides information about the Linux distribution, used here to dynamically detect your OS version.
sudo apt-get install wget apt-transport-https gnupg lsb-release

# Download and add the GPG public key for Trivy's repository.
# The GPG key is used to verify that the Trivy packages are authentic and have not been tampered with.
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -

# Add the Trivy repository to your system's list of package sources.
# - $(lsb_release -sc): Dynamically determines your Ubuntu codename (e.g., "jammy" for Ubuntu 22.04).
# - "main": Specifies the main component of the repository.
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list

# Update the system's package index to include metadata from the newly added Trivy repository.
# This ensures that the system recognizes and can fetch Trivy packages from the Aqua Security repository.
sudo apt-get update

# Install Trivy, an open-source vulnerability scanner, from the Aqua Security repository.
# This will install Trivy and its dependencies on your system.
sudo apt-get install trivy

wget https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl

# Copy the template to the Trivy directory
sudo cp html.tpl /usr/local/share/trivy/html.tpl

# Verify the installation by checking the installed Trivy version.
# If successful, this will output the current version of Trivy installed on your system.
trivy --version

```

### Download the Trivy Database
```bash

# Create a temporary directory for Trivy's cache.
# The `mktemp -d` command creates a unique temporary directory and returns its path.
# We store this path in the `TRIVY_TEMP_DIR` variable for later use.
TRIVY_TEMP_DIR=$(mktemp -d)

# Use Trivy to download its vulnerability database into the temporary cache directory.
# - `--cache-dir $TRIVY_TEMP_DIR`: Specifies that the downloaded data should be stored in the temporary directory.
# - `image --download-db-only`: Downloads only the vulnerability database without scanning an actual container image.
trivy --cache-dir $TRIVY_TEMP_DIR image --download-db-only

# Archive the Trivy database files into a compressed `.tar.gz` file called `db.tar.gz`.
# - `-cf ./db.tar.gz`: Creates a tar archive (`.tar.gz`) in the current working directory.
# - `-C $TRIVY_TEMP_DIR/db`: Changes to the directory where the Trivy database is stored.
# - `metadata.json trivy.db`: Includes only the specific database files needed by Trivy.
tar -cf ./db.tar.gz -C $TRIVY_TEMP_DIR/db metadata.json trivy.db

# Clean up by removing the temporary directory created earlier.
# This ensures no leftover, unnecessary files remain on your system.
rm -rf $TRIVY_TEMP_DIR

```

## 4. SonarQube Installation
<img src="https://assets-eu-01.kc-usercontent.com/55017e37-262d-017b-afd6-daa9468cbc30/6c029869-3710-457a-9f48-58b10b871cb6/SQ-Server_Built-in-padding_300px.svg?w=300&h=72&auto=format&fit=crop" alt="Trivy" width="25%" height="100%">
Run SonarQube using a Docker container:

```bash

# Run a SonarQube container in detached mode.
# - `docker run`: Command to create and start a new Docker container.
# - `-d`: Runs the container in "detached" mode, meaning it runs in the background.
# - `--name sonarqube`: Assigns the name "sonarqube" to the container for easier reference.
# - `-e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true`: Disables Elasticsearch bootstrap checks.
#    - This is necessary because SonarQube uses Elasticsearch internally,
#      and the default configuration may fail startup in Docker environments without this setting.
# - `-p 9000:9000`: Maps port 9000 on the host machine to port 9000 in the container.
#    - This allows you to access SonarQube from your browser using `http://<host_ip>:9000`.
# - `sonarqube:latest`: Specifies the SonarQube image to be used (the `latest` tag indicates the most recent version available on Docker Hub).

docker run -d --name sonarqube -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true -p 9000:9000 sonarqube:latest

```

### Access SonarQube
Access SonarQube at `http://<your_server_ip>:9000`. Default credentials:

Username: `admin`
Password: `admin`

### Setup SonarQube in Jenkins
- Go to `Manage Jenkins` > `Global Tool Configuration`
- Scroll down to the `SonarQube Scanner installations` section and click on `Add SonarQube Scanner`.
- Enter the following details:
  - Name: `SonarqubeAll`
  - Install automatically: `Install automatically`
  - Click on `Save` to save the configuration.

- Generate a token for the user `admin` in SonarQube
- Go to `(http://IP:9000/account/security)` Name: `yourmentors-webinar` Type: `User Token` and click on `Generate`
- `squ_daf11725baf5e574b5e32b760c39aa33ac07909a`

- Go to `Manage Jenkins` > `Configure System`
- Scroll down to the `SonarQube servers` section and click on `Add SonarQube`.
- Enter the following details:
  - Name: `SonarQube`
  - Server URL: `http://<your_server_ip>:9000`
  - Server authentication token: `+ Add` > `Jenkins` > `Secret text`
  - ID: `sonarqube-creds`
  - Server authentication token: `squ_daf11725baf5e574b5e32b760c39aa33ac07909a`
  - Click on `Save` to save the configuration.

## 5. Node.js Installation
<img src="https://upload.wikimedia.org/wikipedia/commons/d/d9/Node.js_logo.svg" alt="Trivy" width="25%" height="100%">
Install Node.js to support JavaScript builds:

```bash

# Download and execute the NodeSource setup script to add the Node.js 18.x repository.
# - `curl -fsSL`: Fetches the script securely in a silent mode without progress bars.
# - `https://deb.nodesource.com/setup_18.x`: The URL to the NodeSource setup script for Node.js version 18.x.
# - `sudo -E bash -`: Runs the fetched script as root, preserving your environment (`-E`) and executing it in Bash.
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# Install Node.js (and npm) from the newly added NodeSource repository.
# - `sudo apt install -y nodejs`: Installs Node.js and npm (Node.js Package Manager) with non-interactive confirmation (`-y`).
sudo apt install -y nodejs

# Verify the installed version of Node.js.
# The `node -v` command outputs the installed Node.js version (e.g., v18.x.x if successful).
node -v

# Verify the installed version of npm (Node.js Package Manager).
# The `npm -v` command outputs the installed npm version, which is bundled with Node.js installations.
npm -v

```

## Reference Links
- [Jenkins Documentation](https://www.jenkins.io/doc/book/installing/linux/ "Click to install Jenkins")
- [Docker Documentation](https://docs.docker.com/engine/install/ubuntu/ "Click to install Docker Engine")
- [Trivy Documentation](https://trivy.dev/latest/getting-started/ "Click to install Trivy")
- [SonarQube Documentation](https://www.sonarsource.com/open-source-editions/sonarqube-community-edition/ "Click to install SonarQube Server")

# Jenkins Pipeline Script

```groovy

pipeline {

  agent any

  environment {
    REPO_LINK = ''
    SONAR_HOST_URL = ""
    IMAGE_NAME= "node-goat"
    IMAGE_TAG= "v1.0.${env.BUILD_ID}"
  }
  stages {
    stage('Cloning repository') {
        steps {
            git branch: 'master', url: env.REPO_LINK
      }
    }

    stage('Trivy FS scanning') {
      steps {
        script {
          // HTML Report
          sh "trivy fs . --format template --template \"@/usr/local/share/trivy/templates/html.tpl\" --output Trivy_FS_Scan_${env.BUILD_ID}.html"
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "Trivy_FS_Scan_${env.BUILD_ID}.html", fingerprint: true
          publishHTML (target: [
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: true,
            reportDir: '.',
            reportFiles: 'Trivy_FS_Scan_${env.BUILD_ID}.html',
            reportName: 'Trivy File System Scan',
          ])
        }
      }
    }
    stage('Trivy misconfig scanning') {
      steps {
        script {
          // HTML Report
          sh "trivy fs --scanners license,misconfig . --format template --template \"@/usr/local/share/trivy/templates/html.tpl\" --output Trivy_Misconfig_Scan_${env.BUILD_ID}.html"
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "Trivy_Misconfig_Scan_${env.BUILD_ID}.html", fingerprint: true
          publishHTML (target: [
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: true,
            reportDir: '.',
            reportFiles: 'Trivy_Misconfig_Scan_${env.BUILD_ID}.html',
            reportName: 'Trivy Misconfig Scan',
          ])
        }
      }
    }

    stage('Code Quality Analysis'){
        steps {
            script {
                SCANNER_HOME = tool 'SonarqubeAll'
            }
            withSonarQubeEnv(installationName: 'Sonarqube',credentialsId: 'sonarqube-creds') {
                sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=${env.JOB_BASE_NAME} -Dsonar.projectName=${env.JOB_BASE_NAME}"
            }
        }
    }

    stage("Quality Gate check") {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }

    stage('Building image') {
      steps {
        script{
            sh "docker build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} ."
        }
      }
    }

    stage('Trivy image scanning') {
      steps {
        script {
          // HTML Report
          sh "trivy image --scanners vuln ${env.IMAGE_NAME}:${env.IMAGE_TAG} --format template --template \"@/usr/local/share/trivy/templates/html.tpl\" --output Trivy_Image_Scan_${env.BUILD_ID}.html"
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "Trivy_Image_Scan_${env.BUILD_ID}.html", fingerprint: true
          publishHTML (target: [
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: '.',
            reportFiles: 'Trivy_Image_Scan_${env.BUILD_ID}.html',
            reportName: 'Trivy Image Scan',
          ])
        }
      }
    }
  }
  post {
      always {
          cleanWs()
      }
  }
}

```