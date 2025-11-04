# Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

### **Project Overview**
This document outlines the step-by-step process for deploying a 2-tier web application (Flask + MySQL) on an AWS EC2 instance. The deployment is containerized using Docker and Docker Compose.  
A full CI/CD pipeline is established using Jenkins to automate the build and deployment process whenever new code is pushed to a GitHub repository.

---

### **Architecture Diagram**

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same AWS EC2)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

---

### **Step 1: AWS EC2 Instance Preparation**

1. **Launch EC2 Instance:**
   - Navigate to the AWS EC2 console.  
   - Launch a new instance using **Ubuntu 22.04 LTS** AMI.  
   - Choose **t2.micro** (Free-tier eligible).  
   - Create and assign a new key pair for SSH access.

2. **Configure Security Group:**
   - Add inbound rules:
     - **SSH:** TCP 22 (Your IP)  
     - **HTTP:** TCP 80 (Anywhere)  
     - **Flask App:** TCP 5000 (Anywhere)  
     - **Jenkins:** TCP 8080 (Anywhere)

3. **Connect to EC2 Instance:**
   ```bash
   ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>
   ```

---

### **Step 2: Install Dependencies on EC2**

1. **Update System Packages:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Install Git, Docker, and Docker Compose:**
   ```bash
   sudo apt install git docker.io docker-compose-v2 -y
   ```

3. **Start and Enable Docker:**
   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

4. **Add User to Docker Group:**
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```

---

### **Step 3: Jenkins Installation and Setup**

1. **Install Java (OpenJDK 17):**
   ```bash
   sudo apt install openjdk-17-jdk -y
   ```

2. **Add Jenkins Repository and Install:**
   ```bash
   curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt update
   sudo apt install jenkins -y
   ```

3. **Start and Enable Jenkins:**
   ```bash
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```

4. **Initial Jenkins Setup:**
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
   - Access Jenkins: `http://<ec2-public-ip>:8080`
   - Enter password, install suggested plugins, and create admin user.

5. **Grant Docker Permissions to Jenkins:**
   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```

---

### **Step 4: GitHub Repository Configuration**

Ensure your GitHub repo includes these files:

#### **Dockerfile**
```dockerfile
FROM python:3.9-slim
WORKDIR /app
RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config &&     rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

#### **docker-compose.yml**
```yaml
version: "3.8"

services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_DATABASE: "devops"
      MYSQL_ROOT_PASSWORD: "root"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - two-tier
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask:
    build:
      context: .
    container_name: two-tier-app
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=devops
    networks:
      - two-tier
    depends_on:
      - mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

volumes:
  mysql-data:

networks:
  two-tier:
```

#### **Jenkinsfile**
```groovy
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-repo.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

---

### **Step 5: Jenkins Pipeline Creation and Execution**

1. **Create a New Pipeline:**
   - Dashboard → **New Item** → Name it → Select **Pipeline** → Click **OK**.

2. **Configure Pipeline:**
   - Set **Definition:** Pipeline script from SCM  
   - Choose **Git**  
   - Enter your GitHub repo URL  
   - Script Path: `Jenkinsfile`  
   - Click **Save**

3. **Run the Pipeline:**
   - Click **Build Now**
   - Monitor via **Stage View** or **Console Output**

4. **Verify Deployment:**
   - Access Flask app: `http://<ec2-public-ip>:5000`
   - Check containers:
     ```bash
     docker ps
     ```

---

### **Conclusion**
Your CI/CD pipeline is now **fully automated**.  
Any new `git push` to the `main` branch will trigger Jenkins to:
- Build the Docker image  
- Deploy the updated app via Docker Compose  

This ensures continuous integration and continuous deployment with minimal manual effort.

---

### **Infrastructure Diagram**
![Infrastructure](diagrams/Infrastructure.png)

### **Workflow Diagram**
![Workflow](diagrams/project_workflow.png)

### **Website Preview**
![Website](diagrams/website.png)
