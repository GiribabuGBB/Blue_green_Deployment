🚀 Blue-Green Deployment on AWS — ECS Fargate + Jenkins + Terraform
A production-style CI/CD pipeline that deploys a Node.js app to AWS ECS Fargate using blue-green deployment strategy, with zero downtime version switches via an Application Load Balancer.

📁 Project Structure
Blue_Green_Deployment/
├── server.js           # Node.js app (Express)
├── Dockerfile          # Container build instructions
├── Jenkinsfile         # CI/CD pipeline definition
├── package.json        # Node dependencies
└── terraform/
    ├── vpc.tf          # VPC, Subnets, Internet Gateway
    ├── provider.tf     # AWS provider config
    ├── variables.tf    # Input variables
    └── outputs.tf      # Output values (VPC ID, Subnet IDs)

🏗️ Architecture Overview
Developer
    │
    │  git push
    ▼
GitHub (main branch)
    │
    │  Jenkins polls / triggered
    ▼
Jenkins Pipeline (on EC2)
    │
    ├── 1. Checkout code from GitHub
    ├── 2. Build Docker image
    ├── 3. Login to ECR
    ├── 4. Push image to ECR
    ├── 5. Update ECS Task Definition
    ├── 6. Health Check (wait 120s)
    └── 7. Switch ALB traffic to new version
              │
              ▼
    ALB (Application Load Balancer)
    Port 3000 → BlueDeployment Target Group
              │
              ▼
    ECS Fargate Task (runs Docker container)
    Private IP: 10.0.x.x  Port: 3000
              │
              ▼
    App Response: "Version 3"

🔵🟢 What is Blue-Green Deployment?
Blue-Green is a deployment strategy that runs two identical environments:
EnvironmentPurposeBlueCurrently live — serving real user trafficGreenNew version — deployed and tested in background
How a release works:

New version is deployed to Green (users still on Blue)
Green passes health checks
ALB listener is switched → Green becomes live
Blue becomes standby (instant rollback available)

Why use it?
Problem (Traditional Deploy)Solution (Blue-Green)App goes down during updateZero downtimeNo easy rollbackSwitch back to Blue in secondsUsers see errors mid-deployUsers never notice the change

🧱 AWS Components Explained
1. ECR — Elastic Container Registry
What it is: AWS's private Docker image storage (like Docker Hub, but private).
In this project:

Every Jenkins build pushes a new image tagged with the build number
Example: 356627769740.dkr.ecr.ap-south-1.amazonaws.com/myapp:14
Old images (:v3, :12) remain stored for rollback

Jenkins builds image → tags it :14 → pushes to ECR
ECS pulls :14 from ECR → runs it as a container

2. ECS — Elastic Container Service
What it is: AWS's container orchestration service. It runs and manages your Docker containers without you managing servers (Fargate mode).
ECS has 3 key concepts:
Task Definition
Think of it as a blueprint for your container.
json{
  "family": "Blue_Green",
  "containerDefinitions": [{
    "name": "myapp",
    "image": "356627769740.dkr.ecr.ap-south-1.amazonaws.com/myapp:14",
    "portMappings": [{ "containerPort": 3000 }]
  }]
}
Every time Jenkins deploys a new version, it:

Fetches the current task definition
Updates the image tag to the new build number
Registers a new revision (e.g. Blue_Green:7)

Task
A running instance of a Task Definition. It's the actual container running your app.
Task Definition (blueprint) → Task (running container)
Blue_Green:7              → 10.0.2.207:3000
Service
The manager that keeps your tasks running.

Ensures desired count (e.g. 1 task) is always running
Replaces crashed tasks automatically
Connects tasks to the Load Balancer's Target Group
On update-service --force-new-deployment: starts new task with new image, waits for it to be healthy, then stops the old task

Service: Blue_Green-service
  ├── Desired: 1
  ├── Running: 1
  └── Registered to: BlueDeployment Target Group

3. ALB — Application Load Balancer
What it is: Receives user traffic and routes it to healthy containers.
In this project:

DNS: ALBLoadbalancerBluegreen-963860051.ap-south-1.elb.amazonaws.com
Port: 3000
Has one Listener that points to a Target Group

Target Groups
Target groups hold the IPs and ports of your running containers.
Target GroupARN SuffixConnected ToBlueDeployment6931a5cf0af3eaecECS Service (active)GreenDeployment0e097fc6fb1ec50b(standby)
Traffic switch = updating which target group the listener points to.

4. VPC — Virtual Private Cloud (Terraform managed)
What it is: Your private network in AWS.
VPC: 10.0.0.0/16
  ├── public-subnet-1  (10.0.1.0/24)  ap-south-1a
  ├── public-subnet-2  (10.0.2.0/24)  ap-south-1b
  ├── Internet Gateway (igw-0616997da14c7f11c)
  └── Route Table → 0.0.0.0/0 → IGW
Terraform created and manages all of this. Running terraform apply recreates the entire network from code.

🔧 Jenkinsfile — Stage by Stage
groovypipeline {
    agent any          // run on any available Jenkins node
    environment { ... } // global variables available to all stages
Stage 1 — Checkout
groovygit branch: 'main', url: 'https://github.com/GiribabuGBB/Blue_Green_Deployment.git'
Jenkins pulls the latest code from GitHub. Every build starts fresh from the repo.

Stage 2 — Build Docker Image
groovydocker build -t myapp:${BUILD_NUMBER} .
docker tag myapp:${BUILD_NUMBER} $ECR_REPO:${BUILD_NUMBER}

BUILD_NUMBER is Jenkins' auto-incrementing counter (1, 2, 3...)
Builds the image from Dockerfile
Tags it for ECR: 356627769740.dkr.ecr.ap-south-1.amazonaws.com/myapp:14


Stage 3 — Login to ECR
groovyaws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
Gets a temporary token from AWS and logs Docker into ECR so it can push images.
Uses aws-creds stored in Jenkins credentials.

Stage 4 — Push to ECR
groovydocker push $ECR_REPO:${BUILD_NUMBER}
Uploads the built image to ECR. Now AWS can pull it to run as a container.

Stage 5 — Update Task Definition
This is the most important stage. It:

Gets the current task definition JSON from ECS
Updates the image tag to the new build number using Python
Removes read-only fields AWS won't accept
Registers a new task definition revision
Updates the ECS service to use the new revision

Old task def: Blue_Green:6  (image: myapp:v3)
New task def: Blue_Green:7  (image: myapp:14)

Stage 6 — Health Check
groovysleep 120
Waits 2 minutes for ECS to:

Pull the new image from ECR
Start the new container
Pass health checks
Drain and stop the old container


Stage 7 — Switch Traffic
groovyaws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_GREEN
Points the ALB listener to the target group where the new task is registered.
Users now get the new version. Zero downtime.

Stage 8 — Deployment Info
Prints the app URL and build details to the Jenkins console output for easy access.

🔄 Full Deploy Flow (End to End)
1. You edit server.js → "Version 4"
2. git add . && git commit -m "Version 4" && git push origin main
3. Jenkins: Build Now
4. Jenkins pulls code (build #15)
5. Docker builds image → myapp:15
6. Image pushed to ECR → .../myapp:15
7. Task definition updated → Blue_Green:8 (image: myapp:15)
8. ECS service updated → starts new task with myapp:15
9. sleep 120 → new task becomes healthy
10. ALB listener updated → traffic flows to new task
11. Browser: http://ALBLoadbalancerBluegreen-963860051.ap-south-1.elb.amazonaws.com:3000
12. Response: "Version 4" ✅

🔁 Rollback
If anything goes wrong, switch the ALB back to the previous target group:
bashaws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:ap-south-1:356627769740:listener/app/ALBLoadbalancerBluegreen/c4e8f0d58f601261/f09c0f6b6d27d58c \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-south-1:356627769740:targetgroup/BlueDeployment/6931a5cf0af3eaec \
  --region ap-south-1
Users are instantly back on the previous version. The old task is still running.

🛠️ Tools & Technologies
ToolRoleNode.js + ExpressApplication runtimeDockerContainerisationGitHubSource code repositoryJenkinsCI/CD automationTerraformInfrastructure as Code (VPC, Subnets)AWS ECRPrivate Docker image registryAWS ECS FargateServerless container runtimeAWS ALBLoad balancer and traffic routing

🌐 Access the App
http://ALBLoadbalancerBluegreen-963860051.ap-south-1.elb.amazonaws.com:3000
Health check endpoint:
http://ALBLoadbalancerBluegreen-963860051.ap-south-1.elb.amazonaws.com:3000/health

📌 Jenkins Credentials Required
IDTypePurposeaws-credsAWS CredentialsECR login, ECS deploy, ALB updategithub-credsUsername/TokenGitHub repo access

👤 Author
Giribabu — MCA Graduate, SVU Tirupati
Self-directed R&D project for hands-on DevOps and Cloud experience.
