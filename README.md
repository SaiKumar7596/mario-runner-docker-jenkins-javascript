
# 🚀 Mario Runner Game — Complete CI/CD Pipeline Using AWS EC2, Jenkins, Docker, SonarQube & Nexus

This project demonstrates a **full CI/CD pipeline** for deploying a **JavaScript Mario Runner Game** using:

* AWS EC2
* Docker Containers (Jenkins, SonarQube, Nexus)
* GitHub Webhooks
* Docker Hub
* SSH Deployment to EC2
* Nginx Web Server

The pipeline automatically builds, tests, packages, publishes, and deploys your application.

---

# 🏗️ 1. Launch AWS EC2 Instance

### 1. Select OS

* Choose **Ubuntu Server 22.04 LTS**

### 2. Select Instance Type

* Use **t3.large** (required for SonarQube + Jenkins to run smoothly)

### 3. Create Security Group

Allow inbound rules:

| Port        | Service          | Purpose          |
| ----------- | ---------------- | ---------------- |
| 22          | SSH              | Access server    |
| 8080        | Jenkins          | CI/CD            |
| 8081        | Nexus            | Repository       |
| 9000        | SonarQube        | Code Analysis    |
| 80          | Web Application  | Nginx            |
| ALL Traffic | optional for lab | Simplifies setup |

### 4. Storage

* Minimum **16GB** (lab purpose)

---

# 🖥️ 2. Connect to EC2 & Install Docker

SSH into the instance:

```bash
ssh -i "yourkey.pem" ubuntu@<EC2-IP>
```

Update and install Docker:

```bash
sudo apt update -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
```

Add ubuntu to Docker group:

```bash
sudo usermod -aG docker ubuntu
```

Reconnect SSH after this.

---

# 🟦 3. Install & Run SonarQube (Code Quality Analysis)

```bash
docker pull sonarqube:lts
```

Run container:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts
```

Open:

```
http://<EC2-IP>:9000
```

* Default username: **admin**
* Default password: **admin**

Create new password.

Generate Sonar Token:

```
My Account → Security → Generate Token
```

Save token for Jenkins.

---

# 🟥 4. Install & Run Nexus Repository Manager

Pull image:

```bash
docker pull sonatype/nexus3
```

Run container:

```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

Open Nexus:

```
http://<EC2-IP>:8081
```

Get admin password:

```bash
docker exec -it nexus cat /nexus-data/admin.password
```

Login using the password → create a new password.

---

# 🟩 5. Install & Run Jenkins (CI/CD Engine)

Pull Jenkins image:

```bash
docker pull jenkins/jenkins:lts
```

Create persistent volume:

```bash
mkdir /home/ubuntu/jenkins_home
chmod 777 /home/ubuntu/jenkins_home
```

Run Jenkins container:

```bash
docker run -d \
  --name jenkins \
  --user root \
  -p 8080:8080 -p 50000:50000 \
  -v /home/ubuntu/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

Get Jenkins unlock key:

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open:

```
http://<EC2-IP>:8080
```

Install **Suggested Plugins**
Create **first admin user**

---

# 🔌 6. Install Required Jenkins Plugins

Go to:

```
Manage Jenkins → Plugins → Available
```

Install:

1. Docker Pipeline
2. Docker Commons
3. Docker API
4. SSH Agent Plugin
5. SonarQube Scanner (for Jenkins)
6. NodeJS Plugin
7. Pipeline: Stage View (optional)

---

# ⚙️ 7. Configure Jenkins Tools

Go to:

```
Manage Jenkins → Tools
```

### Configure NodeJS:

* Name: **Node16**
* Install Automatically: **Yes**

### Configure Sonar Scanner (IMPORTANT):

```
Manage Jenkins → Configure System → SonarQube Servers
```

Add:

* Name: `Sonar`
* Server URL: `http://<EC2-IP>:9000`
* Authentication Token: use `sonar-token`

---

# 🔐 8. Add Jenkins Credentials

Go to:

```
Manage Jenkins → Credentials → Global Credentials
```

Add the following:

### ✔ Sonar Token

* **ID:** sonar-token
* **Kind:** Secret Text

### ✔ Nexus Credentials

* **ID:** nexus-creds
* Username: admin
* Password: your nexus password

### ✔ DockerHub Credentials

* **ID:** dockerhub-creds
* Username: your DockerHub username
* Password: your DockerHub password

### ✔ SSH Key for Deployment to EC2

* **ID:** ssh-deploy
* Username: ubuntu
* Private Key: paste your `.pem` file

---

# 🐳 9. Create DockerHub Repository

Create a repo named:

```
mario-runner
```

This will store your Docker images.

---

# 📦 10. Add Dockerfile & Jenkinsfile to GitHub Repo

### Dockerfile

```dockerfile
# Build Stage
FROM node:16 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Run Stage
FROM nginx:latest
RUN rm -rf /usr/share/nginx/html/*
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

### Jenkinsfile

```groovy
pipeline {
    agent any

    tools {
        nodejs "Node16"
    }

    environment {
        DOCKER_REPO = "saikumar7596/mario-runner"
    }

    stages {
        stage('Checkout Code') { steps { checkout scm } }

        stage('Install Dependencies') { steps { sh "npm install" } }

        stage('Build JavaScript App') { steps { sh "npm run build" } }

        stage('Build Docker Image') {
            steps {
                script {
                    def tag = GIT_COMMIT.take(7)
                    sh """
                        docker build -t ${DOCKER_REPO}:${tag} .
                        docker tag ${DOCKER_REPO}:${tag} ${DOCKER_REPO}:latest
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${DOCKER_REPO}:latest
                    """
                }
            }
        }

        stage('Deploy on EC2 via SSH') {
            steps {
                sshagent(['ssh-deploy']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@54.215.43.123 "
                          docker pull saikumar7596/mario-runner:latest &&
                          docker stop mario-nginx || true &&
                          docker rm mario-nginx || true &&
                          docker run -d --name mario-nginx -p 80:80 saikumar7596/mario-runner:latest
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Deployment Successful"
        }
    }
}
```

---

# 🌐 11. Configure GitHub Webhook

Go to:

```
GitHub Repo → Settings → Webhooks → Add Webhook
```

Payload URL:

```
http://<JENKINS-IP>:8080/github-webhook/
```

* Content-Type: `application/json`
* Trigger: **Just the push event**
* Disable SSL verification
* Click **Add Webhook**

If successful → GitHub shows **200 OK**.

---

# 🚀 12. Create Jenkins Pipeline

```
New Item → Pipeline → Pipeline from SCM
```

Enter:

* SCM: Git
* Repo URL:

  ```
  https://github.com/SaiKumar7596/mario-runner-game.git
  ```
* Branch: `main`
* Script Path: `Jenkinsfile`

Save → Build Now
OR Git push will auto-trigger via webhook.

---

# 🧪 13. Verify Deployment

### On EC2:

```bash
docker ps
```

Container should be running:

```
mario-nginx   Up   80->80/tcp
```

### Open browser:

```
http://<EC2-IP>
```

Game should load successfully.

---

# 🎉 What You Achieved

✔ Fully automated CI/CD pipeline
✔ NodeJS application build
✔ Docker multistage build
✔ Push to Docker Hub
✔ Automated deployment to EC2
✔ Nginx production server hosting the game
✔ Webhook-based auto-trigger

This lab demonstrates **complete DevOps CI/CD automation**.

---


If you want this as a **PDF**, **DOCX**, or **architecture diagram**, I can generate that too.
