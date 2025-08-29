# CI/CD Full-Flow Setup Guide
Jenkins on EC2 (Docker), GitHub PR CI & Merge-to-Master CD

## Architecture Overview

This setup implements a production-style CI/CD pipeline using:
- **Jenkins Server**: EC2 instance running Jenkins in Docker
- **Production Server**: Separate EC2 instance for application deployment
- **ECR**: AWS container registry for Docker images
- **GitHub**: Source code repository with webhook integration

## Prerequisites

### AWS Resources Required
1. **Two EC2 instances**:
   - Jenkins Server (e.g., t3.medium)
   - Production Server (e.g., t3.small)
2. **ECR Repository**: For storing Docker images
3. **IAM Roles**: With appropriate permissions
4. **Security Groups**: Configured for communication

### GitHub Resources
1. **Two GitHub repositories**:
   - Platform repository (Jenkins setup files)
   - Application repository (calculator app + Jenkinsfile)

## Part 1: AWS Infrastructure Setup

### 1.1 Create ECR Repository
```bash
aws ecr create-repository --repository-name lir-repo --region us-east-1
```
Note the registry URI: `{ACCOUNT-ID}.dkr.ecr.us-east-1.amazonaws.com/lir-repo`

### 1.2 EC2 Instance Configuration

#### Jenkins Server (JENKINS-PRIVATE-IP)
- **Instance Type**: t3.medium or larger
- **Security Group**: Allow inbound ports 22 (SSH), 8080 (Jenkins)
- **IAM Role**: Not required (uses credentials for ECR access)

#### Production Server (PRODUCTION-PRIVATE-IP)
- **Instance Type**: t3.small or larger  
- **Security Group**: Allow inbound ports 22 (SSH), 5000 (Application)
- **IAM Role**: ECR permissions for docker pull operations

**Production Server IAM Role Policy**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetAuthorizationToken",
                "ecr:BatchGetImage"
            ],
            "Resource": "*"
        }
    ]
}
```

## Part 2: Production Server Setup

### 2.1 Install Dependencies on Production Server
```bash
# SSH to production server (PRODUCTION-PRIVATE-IP)
ssh -i your-key.pem ec2-user@PRODUCTION-PRIVATE-IP

# Install Docker
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf awscliv2.zip aws

# Re-login to apply docker group membership
exit
ssh -i your-key.pem ec2-user@10.0.1.83
```

### 2.2 Create SSH Key for Jenkins Access
```bash
# On production server, create dedicated SSH key for Jenkins
ssh-keygen -t rsa -b 4096 -f ~/.ssh/jenkins_deploy_key

# When prompted:
# - Enter file path: /home/ec2-user/.ssh/jenkins_deploy_key (default)
# - Enter passphrase: [Leave empty - press Enter]
# - Enter same passphrase again: [Leave empty - press Enter]

# Set correct permissions for SSH key files
chmod 700 ~/.ssh
chmod 600 ~/.ssh/jenkins_deploy_key
chmod 644 ~/.ssh/jenkins_deploy_key.pub

# Add public key to authorized_keys
cat ~/.ssh/jenkins_deploy_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Display private key (copy this ENTIRE output for Jenkins credentials)
echo "=== COPY THIS PRIVATE KEY FOR JENKINS ==="
cat ~/.ssh/jenkins_deploy_key
echo "=== END OF PRIVATE KEY ==="
```

**Important SSH Key Notes:**
- The private key output includes `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`
- Copy the ENTIRE output including these header/footer lines
- The key should be approximately 30-40 lines long
- Never share the private key - only use it in Jenkins credentials

### 2.3 Test SSH Key Setup
```bash
# Test SSH connection locally (should work without password)
ssh -i ~/.ssh/jenkins_deploy_key ec2-user@localhost 'echo "SSH key works"'

# Test Docker access
docker --version
aws --version

# Test ECR access (if IAM role is configured)
aws sts get-caller-identity
aws ecr get-login-password --region us-east-1 --dry-run
```

## Part 3: Jenkins Server Setup

### 3.1 Install Docker on Jenkins Server
```bash
# SSH to Jenkins server (JENKINS-PRIVATE-IP)  
ssh -i your-key.pem ec2-user@JENKINS-PRIVATE-IP

# Install Docker and Docker Compose
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Re-login to apply docker group membership
exit
ssh -i your-key.pem ec2-user@10.0.1.254
```

### 3.2 Setup Jenkins Platform Repository

Clone the platform repository and start Jenkins:
```bash
git clone https://github.com/your-username/jenkins-platform-repo.git
cd jenkins-platform-repo

# Build and start Jenkins
docker-compose up -d

# Get initial admin password
docker logs jenkins
# Look for: "Please use the following password to proceed to installation"
```

### 3.3 Jenkins Initial Configuration

1. **Access Jenkins**: http://your-jenkins-server-ip:8080
2. **Enter initial admin password** from logs
3. **Install suggested plugins** plus additional required plugins:
   - Docker Pipeline
   - SSH Agent
   - AWS Credentials (if using AWS credentials instead of IAM roles)
   - Multibranch Scan Webhook Trigger

## Part 4: Jenkins Configuration (Detailed)

### 4.1 Add SSH Credentials (Step-by-Step)

1. **Navigate to Credentials**:
   - Go to Jenkins Dashboard
   - Click **Manage Jenkins** (left sidebar)
   - Click **Manage Credentials** 
   - Click **System** under "Stores scoped to Jenkins"
   - Click **Global credentials (unrestricted)**

2. **Add New Credential**:
   - Click **Add Credentials** (left sidebar)
   - **Kind**: Select "SSH Username with private key"
   - **Scope**: Global (Jenkins, nodes, items, all child items, etc.)
   - **ID**: `production-ssh-key` (EXACTLY this ID - Jenkinsfile references it)
   - **Description**: `SSH key for production server deployment`
   - **Username**: `ec2-user`
   
3. **Configure Private Key**:
   - **Private Key**: Select "Enter directly"
   - **Key**: Paste the ENTIRE private key from production server
   - Include `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`
   - **Passphrase**: Leave empty (if you created key without passphrase)
   
4. **Save the credential**

**Testing SSH Credential:**
After saving, you can test the credential works by creating a simple pipeline job that runs:
```groovy
pipeline {
    agent any
    stages {
        stage('Test SSH') {
            steps {
                sshagent(['production-ssh-key']) {
                    sh 'ssh -o StrictHostKeyChecking=no ec2-user@10.0.1.83 "echo SSH connection successful"'
                }
            }
        }
    }
}
```

### 4.2 Add AWS Credentials (if not using IAM roles)

**Only needed if Jenkins server doesn't have IAM role with ECR permissions**

1. **Add New Credential**:
   - **Kind**: AWS Credentials  
   - **Scope**: Global
   - **ID**: `aws-ecr-credentials`
   - **Description**: `AWS credentials for ECR access`
   - **Access Key ID**: Your AWS access key ID
   - **Secret Access Key**: Your AWS secret access key

### 4.3 Create Multibranch Pipeline Job (Detailed Steps)

1. **Create New Job**:
   - From Jenkins Dashboard, click **New Item**
   - **Enter an item name**: `calculator-app`
   - **Select**: Multibranch Pipeline
   - Click **OK**

2. **Configure Branch Sources**:
   - **Add source** → **Git**
   - **Project Repository**: `https://github.com/your-username/calculator-app.git`
   - **Credentials**: None (for public repo) or add GitHub credentials if private

3. **Configure Discover Strategies** (Important for PR handling):

   **Discover branches**:
   - **Strategy**: "Exclude branches that are also filed as PRs"
   - This prevents duplicate builds when PR is created

   **Discover pull requests from origin**:
   - **Strategy**: "The current pull request revision" 
   - This builds the PR head commit

   **Discover pull requests from forks**:
   - **Strategy**: "The current pull request revision"
   - **Trust**: "From users with Admin or Write permission"

4. **Build Configuration**:
   - **Mode**: by Jenkinsfile
   - **Script Path**: Jenkinsfile (default)

5. **Scan Multibranch Pipeline Triggers**:
   - ☑ **Periodically if not otherwise run**
     - **Interval**: 1 day
   - ☑ **Scan by webhook** 
     - This enables webhook triggering from GitHub

6. **Orphaned Item Strategy**:
   - ☑ **Discard old items**
   - **Days to keep old items**: 7
   - **Max # of old items to keep**: 10

7. **Click Save**

**After Saving:**
- Jenkins will immediately scan the repository
- Look for "Branch Indexing" in the job log
- It should discover your branches (main, any feature branches)
- If there's a Jenkinsfile in the repository, it will prepare pipelines for each branch

## Part 5: GitHub Repository Setup

### 5.1 Application Repository Structure
Your application repository should contain:
```
calculator-app/
├── Jenkinsfile
├── Dockerfile
├── requirements.txt
├── api.py
├── calculator_app.py
├── calculator_logic.py
├── tests/
│   ├── test_calculator_logic.py
│   └── test_calculator_app_integration.py
└── README.md
```

### 5.1.5 Manual Application Testing (Before CI/CD)

Before setting up the CI/CD pipeline, verify the application works correctly by running it manually on the production server.

#### Test Application Locally on Production Server

**Step 1: Clone Application Repository**
```bash
# SSH to production server (10.0.1.83)
ssh -i your-key.pem ec2-user@10.0.1.83

# Clone your application repository
git clone git@github.com:Lirhen/lir-app.git
cd lir-app
```

**Step 2: Install Python Dependencies and Test**
```bash
# Install Python 3 if not already installed
sudo yum update -y
sudo yum install -y python3 python3-pip

# Install application dependencies
pip3 install -r requirements.txt

# Run unit tests to verify code works
python3 -m unittest discover -s tests -v

# Expected output:
# test_add (test_calculator_logic.TestCalculatorLogic) ... ok
# test_subtract (test_calculator_logic.TestCalculatorLogic) ... ok
# test_multiply (test_calculator_logic.TestCalculatorLogic) ... ok
# test_divide (test_calculator_logic.TestCalculatorLogic) ... ok
# ...
# OK
```

**Step 3: Run Application Directly**
```bash
# Start the Flask application
python3 api.py

# Expected output:
# * Running on all addresses (0.0.0.0)
# * Running on http://127.0.0.1:5000
# * Running on http://172.31.x.x:5000
```

**Step 4: Test Application Access**
```bash
# Open new SSH session to production server (keep app running in first session)
ssh -i your-key.pem ec2-user@10.0.1.83

# Test health endpoint locally
curl http://localhost:5000/health
# Expected: {"status":"ok"}

# Test from external access (if Security Group allows port 5000)
curl http://10.0.1.83:5000/health
```

**Step 5: Test Docker Build Manually**
```bash
# Stop the Python application (Ctrl+C in first session)

# Build Docker image manually
docker build -t calculator-test .

# Run container
docker run -d --name calculator-test -p 5000:5000 calculator-test

# Test containerized application
curl http://localhost:5000/health

# Check container logs
docker logs calculator-test

# Clean up test container
docker stop calculator-test
docker rm calculator-test
docker rmi calculator-test
```

**Step 6: Verify External Access (Important for CI/CD)**
If you need external access for health checks, update Security Group:
1. AWS Console → EC2 → Security Groups
2. Find production server's security group
3. Add Inbound Rule:
   - Type: Custom TCP
   - Port: 5000
   - Source: Jenkins server IP (10.0.1.254) or 0.0.0.0/0

Test external access:
```bash
# From Jenkins server or your local machine
curl http://PRODUCTION-SERVER-PUBLIC-IP:5000/health
```

**Troubleshooting Manual Testing:**
- **Import errors**: Ensure all Python files are in the same directory
- **Port 5000 in use**: Kill existing processes: `sudo lsof -ti:5000 | xargs sudo kill -9`
- **Permission denied**: Check file permissions: `chmod +x api.py`
- **Docker build fails**: Ensure Dockerfile syntax is correct and requirements.txt exists

Once manual testing passes, the application is ready for CI/CD pipeline integration.

### 5.2 Configure GitHub Webhook (Detailed Steps)

**Why Webhook is Important:**
The webhook automatically notifies Jenkins when:
- New commits are pushed to any branch
- Pull requests are created, updated, or merged
- Without webhook, Jenkins won't know about code changes until manual scan

**Step-by-Step Webhook Configuration:**

1. **Navigate to Repository Settings**:
   - Go to your GitHub repository
   - Click **Settings** tab (top right of repository page)
   - Click **Webhooks** (left sidebar under "Code and automation")

2. **Add Webhook**:
   - Click **Add webhook** button
   - GitHub may prompt for password confirmation

3. **Configure Webhook Settings**:
   - **Payload URL**: `http://JENKINS-PUBLIC-IP:8080/github-webhook/`
     - Replace `JENKINS-PUBLIC-IP` with actual public IP of Jenkins server
     - Example: `http://44.201.231.4:8080/github-webhook/`
     - **Important**: Must end with `/github-webhook/` (note the trailing slash)
   - **Content type**: `application/json`
   - **Secret**: Leave empty (or set if you want additional security)

4. **Select Events**:
   Choose **"Send me everything"** OR select individual events:
   - ☑ **Pushes** (triggers on commits to any branch)
   - ☑ **Pull requests** (triggers on PR creation/updates)
   - ☑ **Repository** (for repository changes)

5. **Final Settings**:
   - ☑ **Active** (webhook is enabled)
   - Click **Add webhook**

**Verify Webhook Works:**
- After creating webhook, GitHub will send a test ping
- Check webhook page - should show green checkmark if successful
- Make a test commit to trigger webhook and verify Jenkins receives it

**Webhook Troubleshooting:**
- **Red X on webhook**: Check Jenkins server is accessible on port 8080
- **Webhook shows deliveries but Jenkins doesn't trigger**: Check Jenkins has "Scan by webhook" enabled
- **Firewall issues**: Ensure EC2 Security Group allows inbound port 8080

## Part 6: Testing the CI/CD Pipeline

### 6.1 CI Flow (Pull Request)
1. **Create a feature branch**:
   ```bash
   git checkout -b feature-branch
   # Make some changes
   git add .
   git commit -m "Add new feature"
   git push origin feature-branch
   ```

2. **Create Pull Request** from `feature-branch` to `main`

3. **Expected CI Pipeline**:
   - Setup (install dependencies)
   - Build Container Image (tagged as `pr-{PR-ID}-{BUILD-NUMBER}`)
   - Test (run unit tests)
   - Push to ECR
   - **No deployment** (only for main branch)

### 6.2 CD Flow (Merge to Main)  
1. **Merge Pull Request** to `main` branch

2. **Expected CD Pipeline**:
   - Setup (install dependencies)
   - Build Container Image (tagged as `latest-{BUILD-NUMBER}`)
   - Test (run unit tests)
   - Push to ECR
   - **Deploy to Production** (SSH to production server, pull image, run container)
   - **Health Verification** (check `/health` endpoint)

## Part 7: Troubleshooting

### Common Issues

#### Docker Permission Issues in Jenkins
If you see "permission denied" errors accessing Docker socket:

**Quick Fix:**
```bash
# Fix Docker permissions in Jenkins container
docker exec -u root -it jenkins bash -c "
    groupadd -g 999 docker 2>/dev/null || true
    usermod -aG docker jenkins
    chgrp docker /var/run/docker.sock
    chmod 666 /var/run/docker.sock
"
docker-compose restart
```

**Permanent Fix (Update docker-compose.yml):**
Make sure your `docker-compose.yml` includes:
```yaml
services:
  jenkins:
    # ... other settings
    user: "0:999"  # root user with docker group
    group_add:
      - "999"      # docker group id
```

#### ECR Login Issues
**Test ECR access on production server:**
```bash
# Test IAM role permissions
aws sts get-caller-identity
aws ecr get-login-password --region us-east-1

# Test Docker login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 992382545251.dkr.ecr.us-east-1.amazonaws.com
```

**Common ECR Issues:**
- **"no basic auth credentials"**: IAM role missing ECR permissions
- **"repository does not exist"**: Wrong ECR repository name/region
- **"access denied"**: IAM role needs `ecr:GetAuthorizationToken` permission

#### SSH Connection Issues
**Test SSH connection from Jenkins server to production:**
```bash
# Test from Jenkins server command line
ssh -i /path/to/key -o StrictHostKeyChecking=no ec2-user@PRODUCTION-PRIVATE-IP 'echo "SSH works"'

# Or using the Jenkins SSH credential test (create temporary pipeline):
pipeline {
    agent any
    stages {
        stage('Test SSH') {
            steps {
                sshagent(['production-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@PRODUCTION-PRIVATE-IP '
                            echo "SSH connection successful"
                            docker --version
                            aws --version
                        '
                    '''
                }
            }
        }
    }
}
```

**SSH Troubleshooting:**
- **"Permission denied (publickey)"**: Private key not added to Jenkins credentials correctly
- **"Host key verification failed"**: Use `-o StrictHostKeyChecking=no` in SSH commands
- **"Connection refused"**: Check Security Group allows port 22 from Jenkins server
- **Key format issues**: Ensure entire private key including headers/footers is copied

#### Webhook Not Triggering
**Check webhook delivery:**
1. GitHub Repository → Settings → Webhooks
2. Click on your webhook
3. Check **Recent Deliveries** tab
4. Look for green checkmarks (success) or red X (failure)

**Webhook debugging:**
```bash
# Test webhook URL manually
curl -X POST http://your-jenkins-server:8080/github-webhook/

# Check Jenkins logs for webhook reception
docker logs jenkins | grep webhook
```

#### Application Not Starting
**Check container logs on production server:**
```bash
# List containers (running and stopped)
docker ps -a

# Check logs of specific container
docker logs calculator-app

# Check if port 5000 is in use
netstat -tlnp | grep 5000
```

**Common Application Issues:**
- **"Port already in use"**: Stop previous container: `docker stop calculator-app && docker rm calculator-app`
- **"No such image"**: ECR pull failed - check IAM permissions and ECR login
- **Container exits immediately**: Check Dockerfile CMD command and application code

### SSH Key Permissions Troubleshooting

**On Production Server:**
```bash
# Check SSH directory permissions
ls -la ~/.ssh/
# Should show:
# drwx------ (700) for .ssh directory  
# -rw------- (600) for private keys
# -rw-r--r-- (644) for public keys
# -rw------- (600) for authorized_keys

# Fix permissions if needed
chmod 700 ~/.ssh
chmod 600 ~/.ssh/jenkins_deploy_key ~/.ssh/authorized_keys
chmod 644 ~/.ssh/jenkins_deploy_key.pub
```

**In Jenkins:**
- Verify SSH credential ID exactly matches Jenkinsfile (`production-ssh-key`)
- Ensure private key includes header/footer lines
- Test credential with simple SSH command before using in pipeline

### Pipeline Debugging
- **Jenkins Logs**: Check individual build logs in Jenkins UI
- **ECR Console**: Verify images are being pushed
- **Production Health**: Test `curl http://production-server-ip:5000/health`

## Part 8: Pipeline Flow Summary

### CI (Pull Request Flow):
1. Developer pushes to feature branch
2. Creates Pull Request to `main`
3. GitHub webhook triggers Jenkins
4. Jenkins runs on Docker agent:
   - Builds application image
   - Runs tests  
   - Pushes PR-tagged image to ECR
5. No deployment occurs

### CD (Main Branch Flow):
1. Pull Request is merged to `main`
2. GitHub webhook triggers Jenkins
3. Jenkins runs on Docker agent:
   - Builds application image
   - Runs tests
   - Pushes production-tagged image to ECR
   - SSH to production server
   - Pulls image and deploys container
   - Runs health verification
4. Application is live on production server

This setup provides a complete CI/CD pipeline that meets enterprise-grade requirements with proper separation of concerns, automated testing, and production deployment verification.
