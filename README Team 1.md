# 🚀 SDA Capstone Project – Django Blog App on AWS

🎥 **Live Demo (Video):**
[![Watch the Demo](https://img.shields.io/badge/Demo-Video-blue?style=for-the-badge&logo=youtube)](https://drive.google.com/file/d/1i_xnwaK8zpymiDANDBJ9X7LsIB0Dx_lR/view?usp=sharing)

## 🌍 Project Overview

This project demonstrates the deployment of a scalable Django-based blog application on the AWS cloud using production-level architecture. It utilizes:

- ✅ EC2 (Auto Scaling Group)
- ✅ Application Load Balancer (ALB)
- ✅ Amazon RDS (MySQL)
- ✅ Amazon S3 (for media storage)
- ✅ AWS VPC with public/private subnets
- ✅ IAM, NAT Gateway, Route Tables, SSM Parameters
- ✅ Infrastructure as Code via Terraform
- ✅ Monitoring with Prometheus & Grafana

---

## 🛠️ Step-by-Step Deployment Guide

### 🧱 Step 1: VPC and Networking Setup

🔧 Created a custom VPC `sda-capstone-vpc` with CIDR block `10.90.0.0/16`  
🔗 Enabled **DNS hostnames** to allow communication between instances  
🌐 Created 2 Public & 2 Private Subnets across `eu-north-1a` and `eu-north-1b`

#### Subnets:

- Public: `10.90.10.0/24`, `10.90.20.0/24`
### 🌐 Enabling Auto-Assign Public IP for Public Subnet

<p align="center">
  <img src="images/subnet-auto-assign-ip.png" alt="Enable Auto-Assign IP Setting on Public Subnet" width="800"/>
</p>

> ✅ This screenshot shows how we enabled the *Auto-assign Public IPv4 address* for our public subnet (`sda-capstone-public-subnet-1a`).  
> This is a critical step to ensure EC2 instances launched in this subnet receive public IPs automatically.

- Private: `10.90.11.0/24`, `10.90.21.0/24`

### 🌐 VPC Subnets Structure

<p align="center">
  <img src="images/vpc-subnets-list.png" alt="VPC Subnets View in AWS Console" width="850"/>
</p>

> ✅ This screenshot shows the public and private subnets created in two Availability Zones under `sda-capstone-vpc`.  
> Each subnet was configured with specific CIDR blocks to support scalability and isolation.


✅ Auto-assign public IP enabled for public subnets  
✅ Internet Gateway created & attached  

### 🌐 Internet Gateway Attached to VPC

<p align="center">
  <img src="images/internet-gateway-attached.png" alt="Internet Gateway Attached to VPC" width="850"/>
</p>

> ✅ Internet Gateway `sda-capstone-igw` successfully created and attached to our custom VPC (`sda-capstone-vpc`).  
> This allows instances in public subnets to access the internet.

✅ Route Tables created:
- Public RT connected to IGW (0.0.0.0/0)
- Private RT created separately for private subnets

---

### 🔐 Step 2: Security Groups

- **ALB SG** – Allows HTTP (80) & HTTPS (443) from anywhere  
- **EC2 SG** – Allows traffic only from ALB SG on 80/443 + SSH from anywhere  
- **RDS SG** – Accepts MySQL (3306) traffic only from EC2 SG  

✅ All rules scoped within `sda-capstone-vpc`  
✅ Security hardened by linking SGs instead of IPs

---

### 🗂 Step 3: GitHub Repository Setup

🔐 Created **private GitHub repo**: `capstone`  
🔑 Generated personal access token with `repo` scope  
📥 Cloned repo locally and pushed project files  
🧠 Token stored securely in AWS SSM for later use in EC2 startup script

Commands:

git clone https://<TOKEN>@github.com/<username>/capstone.git
cd capstone
git add .
git commit -m "initial commit"
git push

🛡 Step 4: SSM Parameter Store
Stored sensitive credentials securely in AWS SSM:

/myname/capstone/username → admin

/myname/capstone/password → Clarusway1234

/myname/capstone/token → <GitHub Token>

➡️ These values will be retrieved in settings.py using boto3

### 💾 Step 5: RDS – MySQL Database
Created a DB Subnet Group with both private subnets

Launched RDS MySQL 8.0 instance named sda-capstone-rds

Settings:

Storage: 20GB (auto-scaling up to 40GB)

Public Access: ❌

SG: sda-capstone-rds-sg

Initial DB: clarusway

DB credentials pulled from SSM inside Django settings

---

### 🪣 Step 6: Create S3 Bucket

🧾 Created an S3 bucket named: `sdacapstone-<yourname>-blog`  
📦 Purpose: store images and videos uploaded by users on the blog

- Region: Stockholm
- ACL: Bucket Owner Preferred
- ❗ Unchecked "Block all public access"
- Objects will be stored via Django’s S3 storage backend

➡️ Linked later in `settings.py` via:
--python
AWS_STORAGE_BUCKET_NAME = 'sdacapstone-<yourname>-blog'
AWS_S3_REGION_NAME = 'eu-north-1'

### 🌐 Step 7: Create NAT Gateway
💡 Why? EC2s in private subnets need internet access for updates & GitHub clone

Steps:

Allocated new Elastic IP

Created NAT Gateway in sda-capstone-public-subnet-1a

Updated Private Route Table with:

Destination: 0.0.0.0/0
Target: NAT Gateway
✅ Now private EC2 instances can access the internet securely 🚀

### ⚙️ Step 8: Update settings.py Configuration
To secure sensitive data, we dynamically pull credentials from SSM using boto3:

def get_ssm_parameters():
    ssm = boto3.client('ssm', region_name='eu-north-1')
    username = ssm.get_parameter(Name="/<yourname>/capstone/username", WithDecryption=False)['Parameter']['Value']
    password = ssm.get_parameter(Name="/<yourname>/capstone/password", WithDecryption=False)['Parameter']['Value']
    return username, password

✅ Also updated S3 settings, database host (from RDS), and secret key
✅ Pushed the changes to GitHub

### 🧪 Step 9: Test UserData Script on EC2
🎯 Created a temporary EC2 instance (Ubuntu 22.04) in public subnet
🔐 Attached IAM Role: sda-capstone-ec2-ssm-s3-full-access

✅ Ran and verified UserData script:
TOKEN=$(aws ssm get-parameter --name /<yourname>/capstone/token --with-decryption --query 'Parameter.Value' --output text)
git clone https://$TOKEN@github.com/<your-repo-name>/capstone.git
cd capstone/src
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:80

📸 Verified app at http://<Public-DNS>
✅ If success, test server terminated to avoid extra billing

### 🛡️ Step 10: Launch Admin Node (Monitoring + JumpBox)
🖥️ Created an EC2 instance: SDA-Admin-Node

OS: Amazon Linux 2023

Subnet: public

IAM Role: same as above

Installed:

Terraform

Ansible

Prometheus

Grafana

Ports Open: 22, 3000, 9090, 9100

✅ hostnamectl set-hostname SDA-Admin-Node
### 🖥️ EC2 Instances Dashboard (Admin + Web Servers)

<p align="center">
  <img src="images/ec2-instances-dashboard.png" alt="EC2 Instances Running in AWS Console" width="850"/>
</p>

> ✅ This shows all EC2 instances running: the `SDA-Admin-Node` and two web servers deployed via Auto Scaling.  
> Status checks are passing, confirming that the instances are healthy and accessible.


### 🧱 Step 11: Build Full Infrastructure with Terraform
🎯 Created Terraform configuration with the following modules:

main.tf, userdata.sh, variables.tf, sec-gr.tf, iam.tf, data.tf, outputs.tf

ALB, Target Group, Launch Template, ASG, DB Subnet Group, RDS, S3
Ran:
terraform init
terraform apply

📸 Verified full stack deployed on AWS
🌐 App accessible via ALB DNS
📝 Created blog posts successfully

### 🌍 Deployed Django Blog App via ALB

<p align="center">
  <img src="images/django-blog-homepage-alb.png" alt="Clarusway Blog App Running on ALB" width="850"/>
</p>

> ✅ The Django blog application is successfully deployed and accessible through the Application Load Balancer DNS.  
> Interface loaded with login, register, and about pages.


### ⚙️ Step 12: Configure Ansible Dynamic Inventory
📦 On SDA-Admin-Node:

Created inventory_aws_ec2.yml

Added AWS region & EC2 filters by tags (Project: Capstone)

Configured ansible.cfg with PEM key

Test:
ansible-inventory --graph
✅ Able to discover EC2s dynamically using tags

### 🔧 Step 13: Deploy Prometheus Node Exporter via Ansible
🎯 Created Ansible playbook: prometheus_monitoring_setup.yaml
<p align="center">
  <img src="images/step-13-ansible-playbook-node-exporter.png" alt="Ansible Playbook Node Exporter Setup" width="850"/>
</p>
🛠 Installed Node Exporter on all EC2s automatically

ansible-playbook prometheus_monitoring_setup.yaml
✅ Service enabled and running on port 9100
### ✅ Node Exporter Running on EC2

<p align="center">
  <img src="images/node-exporter-running.png" alt="Node Exporter Running on EC2 - Port 9100" width="850"/>
</p>

> 🟢 Node Exporter is successfully up and running, serving metrics on port `9100`.  
> This confirms that our Ansible playbook has been executed correctly across all EC2 nodes.


### 📡 Step 14: Prometheus Service Discovery
Edited prometheus.yml to auto-discover EC2s using ec2_sd_configs:

- job_name: 'ec2-node-exporters'
  ec2_sd_configs:
    - region: eu-north-1
      port: 9100
      filters:
        - name: tag:App
          values: ["BlogPage"]
Ran Prometheus:
./prometheus --config.file=prometheus.yml

<p align="center">
  <img src="images/step-14-prometheus-targets.png" alt="Prometheus Service Discovery Targets" width="850"/>
</p>

> This screenshot confirms that **Prometheus** is successfully scraping metrics from all running EC2 instances tagged for monitoring.  
> Targets are dynamically discovered using `ec2_sd_configs` based on EC2 tags (`App: BlogPage`), which ensures high scalability for the monitoring system. 📊✅

📸 Visited http://<Admin-IP>:9090/targets → EC2s detected! ✅

### 📊 Step 15: Grafana Integration & Dashboard
🎯 Opened http://<Admin-IP>:3000
🔐 Logged in: admin / admin

Steps:

Connected Prometheus as data source

Imported Dashboard ID 1860 (Node Exporter Full)

Saw real-time metrics: CPU, RAM, disk, uptime, network...

✅ Monitoring complete! 🚀

### 📊 Grafana Monitoring Dashboard

<p align="center">
  <img src="images/grafana-node-exporter-dashboard.png" alt="Grafana Node Exporter Dashboard" width="850"/>
</p>

> ✅ Real-time monitoring with Grafana using Node Exporter Full Dashboard (ID: 1860).  
> Displays CPU, RAM, Disk, Network stats, and system load collected from EC2s.

🧼 Clean-up Instructions
To destroy infrastructure:

cd sda-capstone-infra-tf
terraform destroy --auto-approve
Manually delete:

RDS

S3 Buckets (empty first)

Subnet Groups

IAM Roles

IGW & NAT Gateway

SSM Parameters

VPC

## 🚧 Challenges Faced
Throughout the implementation of our capstone project, we encountered several challenges that required troubleshooting, collaboration, and adjustments. These included:

❌ The initial AMI image was incompatible with our region, so we had to search for and configure a suitable one manually.

🔐 Encountered several IAM permission errors that required time-consuming debugging and policy adjustments.

🧱 Some Terraform modules were outdated or misconfigured, leading to deployment failures and resource rollbacks.

🔑 Managing the SSH access for multiple teammates required careful coordination and key sharing.

🛑 File permission issues on the EC2 instance caused problems during Ansible deployment.

⚙️ Environment variables were not being picked up properly by the backend service initially, delaying testing.


### 👑 Final Notes

🎓 This project was built as a capstone for Clarusway DevOps Bootcamp

🧑‍💻 Team 1
📆 Year: 2025


✅ All steps documented, executed, tested, and monitored successfully
💪 Proudly completed with passion, precision, and practical proof!
