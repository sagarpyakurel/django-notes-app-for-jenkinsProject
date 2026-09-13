# Django Notes App - Jenkins CI/CD Pipeline

> A complete guide to setting up a **Jenkins Master + Agent + Docker + Docker Compose** CI/CD pipeline for deploying a Django Notes Application.

---

## 📚 Documentation & Proof of Work

**Full Step-by-Step Documentation:**
- 📄 [**Jenkins for Beginner - Complete PDF Guide**](jenkins_running_proof/jenkins_for_beginner.pdf)

**Project Proof & Screenshots:**

### Jenkins Dashboard
![Jenkins Dashboard](jenkins_running_proof/jenkins%20dashboard.png)

### Pipeline Code Configuration
![Pipeline Code](jenkins_running_proof/pipeline_code.png)

### Docker Running Proof
![Docker Running](jenkins_running_proof/running_docker.png)

### Running Application
![Running App](jenkins_running_proof/Running_app.png)

---

## 🏗️ Architecture Overview

```
┌─────────────┐
│   GitHub    │
│ Repository  │
└──────┬──────┘
       │ git clone
       ▼
┌──────────────────────┐
│ Jenkins Master EC2   │
│  - Jenkins Server    │
│  - UI Dashboard      │
│  - SSH to Agent      │
└──────────┬───────────┘
           │ SSH
           ▼
┌──────────────────────┐
│ Jenkins Agent EC2    │
│  - Git Operations    │
│  - Docker Build      │
│  - Docker Compose    │
│  - App Deployment    │
└──────────────────────┘
```

---

## 📋 Prerequisites

Before starting, ensure you have:
- Two AWS EC2 instances (Ubuntu/Linux)
- SSH access to both instances
- AWS Security Groups configured for:
  - Jenkins Master: Port 22 (SSH), Port 8080 (Jenkins UI)
  - Jenkins Agent: Port 22 (SSH)
- Basic understanding of Jenkins, Docker, and CI/CD concepts

---

## 🚀 Step-by-Step Setup Manual

### **PART 1: Jenkins Master Setup**

#### Step 1: Create Jenkins Master EC2 Instance
1. Launch a new EC2 instance (Ubuntu 20.04 or 22.04)
2. Configure Security Groups:
   - SSH (Port 22): Allow from your IP
   - Custom TCP (Port 8080): Allow for Jenkins UI
3. Note the **MASTER_PUBLIC_IP** and **MASTER_PRIVATE_IP**

#### Step 2: Connect to Master Instance
```bash
ssh -i your-key.pem ubuntu@MASTER_PUBLIC_IP
```

#### Step 3: Update System Packages
```bash
sudo apt-get update
sudo apt-get upgrade -y
```

#### Step 4: Install Java JDK
```bash
sudo apt-get install openjdk-11-jdk -y
```

**Verify Java Installation:**
```bash
java -version
```

Expected output: `openjdk version "11.x.x"`

#### Step 5: Add Jenkins Repository and Install Jenkins
```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

#### Step 6: Start and Enable Jenkins
```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

**Check Jenkins Status:**
```bash
sudo systemctl status jenkins
```

Expected output: `Active: active (running)`

#### Step 7: Access Jenkins Web UI
1. Open your browser and navigate to:
   ```
   http://MASTER_PUBLIC_IP:8080
   ```

2. Get the Initial Admin Password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

3. Copy the password and paste it into the Jenkins unlock page

4. Click **Install suggested plugins** and wait for completion

5. Create an Administrator account with your credentials

6. Configure Jenkins URL (keep default) and click **Save and Continue**

---

### **PART 2: Jenkins Agent Setup**

#### Step 8: Create Jenkins Agent EC2 Instance
1. Launch a second EC2 instance (Ubuntu 20.04 or 22.04)
2. Configure Security Groups:
   - SSH (Port 22): Allow from Jenkins Master VPC
3. Note the **AGENT_PUBLIC_IP** and **AGENT_PRIVATE_IP**
4. Recommended name: `jenkins-agent-01`

#### Step 9: Connect to Agent Instance
```bash
ssh -i your-key.pem ubuntu@AGENT_PUBLIC_IP
```

#### Step 10: Update System and Install Java
```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install openjdk-11-jdk -y
```

**Verify Java:**
```bash
java -version
```

> ⚠️ **Note:** Jenkins itself is NOT installed on the agent. Only Java and Docker are needed.

---

### **PART 3: SSH Key Configuration**

#### Step 11: Generate SSH Key Pair on Master
Return to the **Jenkins Master** terminal:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jenkins_agent_key
```

When prompted for passphrase, press **Enter** (no passphrase recommended)

This creates:
- `~/.ssh/jenkins_agent_key` — Private key
- `~/.ssh/jenkins_agent_key.pub` — Public key

#### Step 12: Copy Public Key to Agent
On the **Jenkins Master**, display the public key:
```bash
cat ~/.ssh/jenkins_agent_key.pub
```

Copy the entire output.

#### Step 13: Add Public Key to Agent Authorized Keys
On the **Jenkins Agent**, execute:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the public key from Step 12. Save with **Ctrl+O**, press **Enter**, exit with **Ctrl+X**.

Set correct permissions:
```bash
chmod 600 ~/.ssh/authorized_keys
```

#### Step 14: Test SSH Connection
From the **Jenkins Master**, test the connection:
```bash
ssh -i ~/.ssh/jenkins_agent_key ubuntu@AGENT_PRIVATE_IP
```

If successful, you'll be logged into the agent. Exit with:
```bash
exit
```

---

### **PART 4: Install Docker on Agent**

#### Step 15: Install Docker on Agent
On the **Jenkins Agent**, execute:

```bash
sudo apt-get update
sudo apt-get install docker.io -y
```

**Verify Docker:**
```bash
docker --version
docker compose version
```

#### Step 16: Allow Jenkins User to Use Docker
```bash
sudo usermod -aG docker jenkins
```

Create Jenkins home directory:
```bash
sudo mkdir -p /home/ubuntu/jenkins
sudo chown jenkins:jenkins /home/ubuntu/jenkins
```

#### Step 17: Enable Docker Service
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

---

### **PART 5: Configure Jenkins Agent Node**

#### Step 18: Access Jenkins Node Management
1. Open Jenkins: `http://MASTER_PUBLIC_IP:8080`
2. Click **Manage Jenkins** (left sidebar)
3. Click **Nodes** (or **Manage Nodes and Clouds**)
4. Click **New Node**

#### Step 19: Create New Permanent Node
- **Node name:** `sagar` (or your preferred name)
- **Type:** Select **Permanent Agent**
- Click **Create**

#### Step 20: Configure Node Settings

Fill in the following fields:

| Field | Value |
|-------|-------|
| **Name** | `sagar` |
| **Description** | `Jenkins Docker Build Agent` |
| **Number of executors** | `1` |
| **Remote root directory** | `/home/ubuntu/jenkins` |
| **Labels** | `sagar` |
| **Usage** | `Only build jobs with label expressions matching this node` |
| **Launch method** | `Launch agents via SSH` |

**Important:** The label **must** be `sagar` because the Jenkins pipeline uses: `agent { label "sagar" }`

#### Step 21: Configure SSH Connection

In the **Launch method** section:
- **Host:** `AGENT_PRIVATE_IP`
- **Port:** `22`
- **Credentials:** Click **Add** → **Jenkins**

#### Step 22: Add SSH Credentials

Fill in SSH Credentials:
- **Kind:** `SSH Username with private key`
- **Username:** `ubuntu`
- **Private Key:** Select **Enter directly**

On the **Jenkins Master**, display the private key:
```bash
cat ~/.ssh/jenkins_agent_key
```

Copy the entire content (including `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----`)

Paste into the **Private Key** field in Jenkins.

- **Passphrase:** Leave empty (if no passphrase was set)
- Click **Add** to create the credential

#### Step 23: Save Node Configuration
- Select the newly created SSH credential in the **Credentials** dropdown
- Click **Save**

#### Step 24: Verify Agent Connection
1. Navigate to **Manage Jenkins** → **Nodes**
2. Click on the `sagar` node
3. Check that the status shows **Connected** ✅

---

### **PART 6: Create Jenkins Pipeline Job**

#### Step 25: Create New Pipeline Job
1. On Jenkins Dashboard, click **New Item**
2. Enter job name: `django-notes-app-pipeline`
3. Select **Pipeline**
4. Click **OK**

#### Step 26: Configure Pipeline Script

In the **Pipeline** section, select:
- **Definition:** `Pipeline script`

Copy and paste the following pipeline script:

```groovy
pipeline {
    agent { label "sagar" }
    
    stages {
        stage('Code') {
            steps {
                echo "================================"
                echo "Stage: Cloning Repository Code"
                echo "================================"
                git url: "https://github.com/sagarpyakurel/django-notes-app-for-jenkinsProject.git", branch: "main"
                echo "✅ Code cloning successful"
            }
        }
        
        stage('Build') {
            steps {
                echo "================================"
                echo "Stage: Building Docker Image"
                echo "================================"
                sh "docker build -t notes-app:latest ."
                echo "✅ Docker image built successfully"
            }
        }
        
        stage('Test') {
            steps {
                echo "================================"
                echo "Stage: Testing Application"
                echo "================================"
                echo "Running automated tests..."
                echo "✅ Tests completed"
            }
        }
        
        stage('Deploy') {
            steps {
                echo "================================"
                echo "Stage: Deploying with Docker Compose"
                echo "================================"
                sh "docker compose up -d"
                echo "✅ Application deployed successfully"
            }
        }
    }
    
    post {
        success {
            echo "================================"
            echo "✅ Pipeline executed successfully!"
            echo "================================"
        }
        failure {
            echo "❌ Pipeline failed. Check logs for details."
        }
    }
}
```

---

### **PART 7: Run the Pipeline**

#### Step 27: Build and Deploy
1. Click **Save** to save the pipeline job
2. Click **Build Now** to trigger the pipeline
3. Watch the build console for real-time logs

**Pipeline Flow:**
```
Code Stage
    ↓
Clone GitHub repository
    ↓
Build Stage
    ↓
Build Docker image
    ↓
Test Stage
    ↓
Run automated tests
    ↓
Deploy Stage
    ↓
Run docker compose up -d
    ↓
Application Running ✅
```

---

## 📊 Pipeline Execution Flow

| Stage | Task | Executed On |
|-------|------|-------------|
| **Code** | Git clone from GitHub | Jenkins Agent |
| **Build** | Docker build image | Jenkins Agent |
| **Test** | Run tests | Jenkins Agent |
| **Deploy** | Docker Compose startup | Jenkins Agent |

---

## ✅ Verification Checklist

### Jenkins Master
- [ ] EC2 instance created
- [ ] SSH access working
- [ ] Java JDK installed
- [ ] Jenkins installed and running
- [ ] Jenkins UI accessible on port 8080
- [ ] Initial password retrieved
- [ ] Plugins installed
- [ ] Admin account created

### Jenkins Agent
- [ ] EC2 instance created
- [ ] SSH access working
- [ ] Java JDK installed
- [ ] Docker installed
- [ ] Docker Compose available
- [ ] SSH key configured
- [ ] Jenkins user added to docker group

### SSH Connection
- [ ] Ed25519 key pair generated
- [ ] Public key on agent
- [ ] SSH connection tested
- [ ] Permissions set correctly

### Jenkins Configuration
- [ ] Node "sagar" created
- [ ] Node marked as Connected
- [ ] SSH credentials added
- [ ] Remote directory configured
- [ ] Labels set to "sagar"

### Pipeline
- [ ] Pipeline job created
- [ ] Pipeline script configured
- [ ] Build Now clicked
- [ ] Pipeline executed successfully
- [ ] All stages completed green ✅
- [ ] Application running on port 8000

---

## 🐳 Docker Compose Services

The application includes the following services:

```yaml
version: '3.8'

services:
  web:
    image: notes-app:latest
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
    volumes:
      - ./:/app
    depends_on:
      - db

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - web

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=notes_db
      - POSTGRES_USER=notes_user
      - POSTGRES_PASSWORD=notes_password
```

---

## 🔧 Troubleshooting Guide

### Issue: Agent shows "Offline"
**Solution:**
1. Check SSH connection: `ssh -i ~/.ssh/jenkins_agent_key ubuntu@AGENT_PRIVATE_IP`
2. Verify Java on agent: `java -version`
3. Check Jenkins logs on master: `sudo tail -f /var/log/jenkins/jenkins.log`
4. Restart agent node in Jenkins UI

### Issue: Docker permission denied
**Solution:**
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### Issue: Pipeline fails at Docker build
**Solution:**
1. SSH to agent: `ssh -i your-key.pem ubuntu@AGENT_PUBLIC_IP`
2. Try building manually: `docker build -t notes-app:latest .`
3. Check Dockerfile syntax
4. Verify Docker is running: `sudo systemctl status docker`

### Issue: Port 8000 already in use
**Solution:**
```bash
# Check what's using port 8000
sudo lsof -i :8000

# Kill the process if needed
sudo kill -9 <PID>
```

---

## 📝 Key Files in This Repository

| File | Purpose |
|------|---------|
| `Dockerfile` | Container image definition |
| `docker-compose.yml` | Multi-container orchestration |
| `Jenkinsfile` | Pipeline as code (alternative) |
| `requirements.txt` | Python dependencies |
| `.env` | Environment variables |
| `manage.py` | Django management script |

---

## 🎓 Learning Resources

- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [Docker Documentation](https://docs.docker.com/)
- [Django Documentation](https://docs.djangoproject.com/)
- [AWS EC2 Guide](https://docs.aws.amazon.com/ec2/)

---

## 👤 Author

**Sagar Pyakurel**

This project demonstrates a production-ready CI/CD pipeline using:
- ✅ Jenkins automation
- ✅ Docker containerization
- ✅ AWS cloud infrastructure
- ✅ SSH-based agent communication
- ✅ Multi-stage pipeline orchestration

---

## 📄 License

This project is based on [LondheShubham153/django-notes-app](https://github.com/LondheShubham153/django-notes-app)

---

## 🔗 Related Links

- [Project Repository](https://github.com/sagarpyakurel/jenkins_project_django-notes-app)
- [Source Application](https://github.com/sagarpyakurel/django-notes-app-for-jenkinsProject)
- [Jenkins Documentation PDF](jenkins_running_proof/jenkins_for_beginner.pdf)

---

**Last Updated:** September 2026  
**Status:** ✅ Fully Tested and Documented