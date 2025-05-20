# SDA-CAPSTONE : Blog Page Application (Django) deployed on AWS Application Load Balancer with Auto Scaling, S3, Relational Database Service(RDS), VPC's Components (SOLUTION)

## Description

The Clarusway Blog Page Application aims to deploy blog application as a web application written Django Framework on AWS Cloud Infrastructure. This infrastructure has Application Load Balancer with Auto Scaling Group of Elastic Compute Cloud (EC2) Instances and Relational Database Service (RDS) on defined VPC. User is able to upload pictures and videos on own blog page and these are kept on S3 Bucket. This architecture will be created by Firms DevOps Guy.

## Problem Solution Order

## Steps to Solution
  
### Step 1: Create Dedicated VPC And Whole Components

- Create a VPC with two public subnets and two private subnets, and place them across two Availability Zones (AZs). Each AZ should include one public subnet and one private subnet.
- We'll set CIDR block as 10.90.0.0/16 for our VPC. You can choose whatever you want. This gives us more than 65000 IP address. After setting CIDR block of VPC, AWS can't expend it, we can just add new CIDR block. 
        
    ### VPC

    - Be sure that you are in `Europe-(Frankfurt) eu-central-1` region. We will create our all resources in `Europe-(Frankfurt) eu-central-1` region

    - Lets create VPC first.

        - create a vpc named `sda-capstone-vpc` CIDR blok is `10.90.0.0/16`

        - no ipv6 CIDR block

        - tenancy: default (A service prepared according to the dedication logic in EC2. We can call this VPC as a dedicate.)

        - After creating VPC, some services are created automatically along with it. For example `Route Table` or `NACL Table` ---> !!! SHOW MAIN ROUTE TABLE !!! This is the `main route table` and we are gonna use this `route table` as either `public route table` or `private route table`. We'll see this a couple of steps later.

    - After creating VPC, the first thing we need to do;
    
      - Select `sda-capstone-vpc`, click `Actions` and `enable DNS hostnames` for the `sda-capstone-vpc`. If we do not enable this, a DNS host name won't be assigned to any the machines under this VPC and these machines will not be able to communicate with each other. They need a DNS for this.

    ## Subnets

    - So, now, we can create subnets. In this example, we are gonna create `2 public` and `2 private` subnets within `2 AZs`. Each subnet's block must be covered by `10.90.0.0/16`. They are going to be /24 subnet masks. Subnet masks of Subnets can be shown on below:

        - Public Subnets
          `10.90.10.0/24` (This mask has 2 over 6 IP addresses, we will be able to use 251 of them. Why? 
            1. The first address of each block (10.90.10.0) is `network address` and the last one is `broadcast address` (10.90.10.255). Two addresses is reserved for this purpose  
            2. Normally, I can attach totally 254 IPs for each subnet, when we create it outside. But, in AWS world, 3 more IP addresses are reserved which are 
            3. 10.90.10.1 for VPC router
            4. 10.90.10.2 for DNS
            5. 10.90.10.3 for future usage.)

            `10.90.20.0/24`

        - Private Subnets
            `10.90.11.0/24`

            `10.90.21.0/24`
        
    - Go to the `subnet` section on left hand side and `click` it. Select `create subnet`. We are gonna create Subnets one by one to follow directions respectively on below;

        - Create a public subnet named `sda-capstone-public-subnet-1a` under the vpc `sda-capstone-vpc` in AZ eu-central-1a with `10.90.10.0/24`

        - Create a private subnet named `sda-capstone-private-subnet-1a` under the vpc `sda-capstone-vpc` in AZ eu-central-1a with `10.90.11.0/24`

        - Create a public subnet named `sda-capstone-public-subnet-1b` under the vpc `sda-capstone-vpc` in AZ eu-central-1b with `10.90.20.0/24`

        - Create a private subnet named `sda-capstone-private-subnet-1b` under the vpc `sda-capstone-vpc` in AZ eu-central-1b with `10.90.21.0/24`

    - We have said that we want the machines we put on public subnets to reach the outside. For this purpose, we need to coordinate `auto-assign IP` setting up for `public subnets`. After this modification, whenever we create a machine on public subnets, Public IP address will be automatically assigned on it. We don't need to worry about it anymore.

      - Select `public subnet-1A`>>> `actions` >>> `edit subnet settings` >>>> `enable auto assign ip`

    - Do you think our public subnets are really public? What is the reason we call them as public subnet? (The reason is connection of internet.)
    
      - We are gonna manage it with `route tables` and `internet gateway`. Lets create `internet gateway` first and then create `route table`. 


    ## Internet Gateway

    - Click `Internet gateway` section on left hand side. Create an internet gateway named `sda-capstone-igw`

    - This device is a virtual device that allows our public subnets to connect to the internet and at the same time to connect to our public subnets from the internet. Actually, we call it device, but this is a virtual device.

    - `ATTACH` the internet gateway `sda-capstone-igw` to the newly created VPC `sda-capstone-vpc`. Go to `Internet gateways` and select newly created `sda-capstone-igw` and click `Actions` ---> `Attach to VPC` ---> Select VPC we would like to attach to. (`sda-capstone-vpc`)

    ## Route Table

    - Go to `Route Tables` on left hand side. We have already one route table as main route table. I am gonna set it up as `public route table` and create a new one which is going to be `private route table`. 

    - What is the reason to CALL some route table as public route table? If any route table has a internet connection with Internet gateway, we call it as public route table. if it hasn't any connection from outside, it calls private route table.

    - First we create a route table and give a name as `sda-capstone-private-rt` private route table.

    - Next, Lets create a `public route table`. We have already a route table. It comes with VPC as default. I'm gonna turn it into public route table. Lets give it a name as `sda-capstone-public-rt` and add a rule in which destination 0.0.0.0/0 (any network, any host) to target the internet gateway `sda-capstone-igw` in order to allow access to the internet.

    - Then, we need to associate our subnets with these route tables in terms of being public or private. Select the `private route table`, come to the `subnet association` subsection and click `Edit subnet associations`. Add `private subnets` to this route table. Similarly, we will do it for `public route table` and `public subnets`. 


### Step 2: Create Security Groups (ALB ---> EC2 ---> RDS)

Before we start, we need to create our `security groups`. Our `EC2 and RDS` are going to be located in `Private subnet`, `ALB` is going to be located in `Public subnet`. In addition, `RDS` only accepts traffic coming from `EC2` and `EC2s` only accepts traffic coming from `ALB`. So, Lets create security groups based on these requirements.

1. ALB Security Group

Name            : sda-capstone-alb-sg
Description     : ALB Security Group allows traffic `HTTP` and `HTTPS` ports from `anywhere` 
Inbound Rules
VPC             : sda-capstone-vpc
HTTP(80)    ----> anywhere
HTTPS (443) ----> anywhere

2. EC2 Security Groups

Name            : sda-capstone-ec2-sg
Description     : EC2 Security Groups only allows traffic coming from `sda-capstone-alb-sg` Security Groups for HTTP and HTTPS ports. In addition, `ssh` port is allowed from anywhere
VPC             : sda-capstone_VPC
Inbound Rules
HTTP(22)    ----> anywhere
HTTP(80)    ----> sda-capstone-alb-sg
HTTPS (443) ----> sda-capstone-alb-sg
Custom TCP (9100) ----> anywhere

3. RDS Security Groups

Name            : sda-capstone-rds-sg
Description     : EC2 Security Groups only allows traffic coming from `sda-capstone-ec2-sg` Security Groups for `MYSQL/Aurora` port. 
VPC             : sda-capstone-vpc
Inbound Rules
MYSQL/Aurora(3306)  ----> sda-capstone-ec2-sg


## Step 3: Prepare Your Github Repository

- Create `private project repository` on your Github and `clone` it on `your local`. Copy all files and folders which are downloaded from clarusway repo (https://github.com/clarusway/clarusway-sda-infrastructure-online) under this folder. `Commit` and `push` them on your `private Githup Repo`.

- Frist Create `private` github repo 
- Go to `github.com`
- First create a `user access token`
- Select `your profile picture` on the to right side
- Choose `settings`
- On the left side choose `developer settings`
- Select `Personal Access Tokens` --> `Tokens (classic)`
- Click on `Generate new token`
- Click on `Generate new token (classic)`
- Enter `capstone project access` under `Note`
- Enter `7 Days` for expiration
- Select the check box `repo`
- Click `Generate token`
- Save your token somewhere safe on your local computer.
- Go to `github.com`
- Create a new repository
- Repository name: `infrastructure-capstone`
- Description: `Repo for AWS Dev Ops Capstone Project`
- Choose `Private`

- In your local machine go to Desktop
- Copy the git clone URL from github.com
- At a terminal type `git clone <Clone URL>`
- If you have authentication issues, you can enter your user name and use the token created above for your password OR try git clone https://<TOKEN>@<YOUR PRIVATE REPO URL>
- Copy the files from the `clarusway-sda-infrastructure-online` to the `infrastructure-capstone` folder on your desktop
- Go back to the terminal
- Make sure you are in the `infrastructure-capstone` folder
- Type the following commands
-   `git add .`
-   `git commit -m "initial commit"`
-   `git push`
- Verify that your new repo has your new files


## Step 4: Creating Parameters in SSM Parameter Store 

- Be sure that you are in `Europe-Frankfurt` `eu-central-1` region 

- Go to `SSM Service` and from left-hand menu, select `Parameters Store`

- Click `Create Parameter`

- Create a parameter for `database master password`  :

 `Name`         : /<yourname>/capstone/password               
 `Description`  : ---
 `Tier`         : Standard
 `Type`         : String   (So AWS encrypts sensitive data using KMS)
 `Data type`    : text
 `Value`        : Clarusway1234

- Create parameter for `database username`  :

 `Name`         : /<yourname>/capstone/username             
 `Description`  : ---
 `Tier`         : Standard
 `Type`         : String   (So AWS encrypts sensitive data using KMS)
 `Data type`    : text
 `Value`        : admin

- Create parameter for `Github TOKEN`  :

 `Name`         : /<yourname>/capstone/token             
 `Description`  : ---
 `Tier`         : Standard
 `Type`         : String   (So AWS encrypts sensitive data using KMS)
 `Data type`    : text
 `Value`        : xxxxxxxxxxxxxxxxxxxx

### Step 5: Create RDS

Lets create our `RDS instance` in which users information are kept. Before we create our database, first we need to do is to create `Subnet Group`. We've spoon up our RDS instance in our default VPC until now. That's why we haven't come across network issue. However, we should arrange `subnet group` for our `custom VPC`. Click `subnet Groups` on the left hand menu and click `create DB Subnet Group` 

```text
Name               : sda-capstone-rds-subnet-group
Description        : sda-capstone-rds-subnet-group
VPC                : sda-capstone_VPC
Add Subnets
Availability Zones : Select 2 AZ(`eu-central-1a` and `eu-central-1b`) in sda-capstone-vpc
Subnets            : Select 2 Private Subnets in these AZs (x.x.11.0, x.x.21.0)
```

- Now we can launch `DB instance` on `Aurora and RDS` console. Go to the `Aurora and RDS` console and `Databases` click `create database` button.

Choose a database creation method : `Standard Create`

Engine Type     : `MySQL`
Edition         : `MySQL Community`
Version         : `8.0.41 (or latest)`
Templates       : `Free Tier`
Settings        : 
    - DB instance identifier : `sda-capstone-rds`
    - Master username        : `admin` (must be same as the value of `/<yourname>/capstone/username`  )
    - Password               : `Clarusway1234` (must be same as the value of `/<yourname>/capstone/password`  )
DB Instance Class            : `Burstable classes` (includes t classes) ---> `db.t4g.micro`
Storage                      : `20 GB and enable autoscaling(up to 40GB)`
Connectivity:
    Compute                  : `Don't connect to EC2`
    VPC                      : `sda-capstone-vpc`
    Subnet Group             : `sda-capstone-rds-subnet-group`
    Public Access            : `No` (When we indicate this as No, RDS can be reached internally. This affects only requests coming from outside)
    VPC Security Groups      : `Choose existing` ---> `sda-capstone-rds-sg`
    Availability Zone        : `No preference`
    Additional Configuration : `Database port` ---> `3306`
Database authentication      : `Password authentication`
Additional Configuration:
    - Initial Database Name  : `clarusway` 
    - Backup                 : `Enable automatic backups`
    - Backup retention period: `7 days`
    - Select Backup Window   : `Select ---> 03:00 (am) Duration 1 hour`
    - Maintance window       : `Select window ---> 04:00(am) Duration:1 hour`

`create instance`


### Step 6: Create S3 Bucket

Go to the `S3` Console and lets `create a bucket`. 

This bucket is going to be used for videos and pictures which will be uploaded on `Blog page`.

- Click `Create Bucket`

Bucket Name : `sdacapstone-shadan-blog`
Region      : `Frankfurt`
Object Ownership
    - `ACLs enabled`
        - `Bucket owner preferred`

Block Public Access settings for this bucket
- Block all public access : `Unchecked` !!!!! (Public)
- Other Settings are keep them as are

`create bucket`


## Step 7: Create NAT Gateway in Public Subnet

Our EC2 created by autoscaling groups will be located in `private subnet` and they need to `update` themselves, install required files and also need to `download` folders and files from Github.

Go to the VPC console and click `NAT gateways`, and click `Create NAT gateway`

    - Name: sda-capstone-nat-gateway
    - Subnet  : sda-capstone-public-subnet-1a
    - Elastic IP allocation ID: click -> "Allocate Elastic IP"
    - Other features keep them as are

- Click on `Create NAT gateway`

- Go to `private route table` and `write` a rule

```
Destination : 0.0.0.0/0
Target      : NAT Gateway ---> Select NAT Gateway

```
- Click on `Save changes`


## Step 8: Update `settings.py` file and `push` to your Github Repo (`infrastructure-capstone`)

- Examine the existing `settings.py` file. We need to change `database configurations` and `S3 bucket` information. But since it is not secure to hardcode some critical data in side the code we use SSM parameters (boto3 function to retrieve database parameters).

- Movie and picture files are kept on S3 bucket named `sdacapstone-<yourname>-blog` as object. We created an S3 bucket, write the name of it on `/src/cblog/settings.py` file as `AWS_STORAGE_BUCKET_NAME` variable. In addition, you must assign region of S3 as `AWS_S3_REGION_NAME` variable.

- As for database; Users credentials and blog contents are going to be kept on RDS database. To connect EC2 to RDS, following variables must be assigned on `/src/cblog/settings.py` file after you create RDS;

    a. Database name - `clarusway`
    b. Database endpoint - `HOST` variables
    c. Port - `3306`
    d. User -  >>>>> `from parameter store` 
    e. PASSWORD >>>> `from parameter store` 

-  So you need to  to change `/<yourname>/capstone/username` and `/<yourname>/capstone/password` with your own parameter name. And also `Database endpoint`

===> PUSH YOUR CHANGES to github:

- git add .
- git commit -m "configuration changes"
- git push


## Step 9: Prepare A Test Instance For Userdata

`Userdata` has already given by `developer` to us. In production environment, you should write it based on application and developer instructions. We will create a `role` for test instance. Later on, we will use the same role for `SDA-Admin-Node`. Because `SDA-Admin-Node` will operate with `Terraform`, we will need extra roles for `EC2, RDS, IAM`. We are adding all of them in advance.

First we will need a role for our ec2 instance:

- Go to `IAM` Service
- Click `Roles`
- Click `Create Role`
- Trusted Entity type: `AWS Service`
- Use case: `EC2`
- Add permission `AmazonS3FullAccess`
                 `AmazonSSMFullAccess`
                 `AmazonEC2FullAccess`
                 `AmazonRDSFullAccess`
                 `IAMFullAccess`
- Role name: ` sda-capstone-ec2-ssm-s3-full-access` 
- Click `Create role`

- Create an ec2 instance in `Public Subnet` and test the `userdata`.

- Run one command at a time and look for any errors

We will create an `Ubuntu` EC2 instance:

- Name: `sda-capstone-test-instance`
- Ubuntu `22.04`
- Instance Type: `t3.micro`
- Key Pair: `your key`
- Subnet: `sda-capstone-public-1a` !!!!!!!!!!!!! (PUBLIC SUBNET)
- Security Group
    - `sda-capstone-ec2-sg`
    - `sda-capstone-alb-sg`
- Advanced details:
    - IAM Instance Profile: 
        - `sda-capstone-ec2-ssm-s3-full-access`

- `Laucnh Instance`

- !!!!!!!! userdata : 3 things must be changed !!!!!!!!!

- token: `TOKEN=$(aws --region=eu-central-1 ssm get-parameter --name /<yourname>/capstone/token --with-decryption --query 'Parameter.Value' --output text)`
- git clone command: `git clone https://$TOKEN@github.com/<your-repo-name>/infrastructure-capstone.git`
- and folder paths: `cd /home/ubuntu/infrastructure-capstone` and `cd /home/ubuntu/infrastructure-capstone/src`


- Connect to `Ubuntu Instance` with remote-SSH

- Apply this these commands to ec2 `manually`

```bash
sudo su
apt-get update -y
apt-get upgrade -y
apt-get install git -y
apt-get install python3 -y
apt install python3-pip -y
pip3 install boto3
apt  install awscli -y
cd /home/ubuntu/
TOKEN=$(aws --region=eu-central-1 ssm get-parameter --name /<yourname>/capstone/token --query 'Parameter.Value' --output text)
git clone https://$TOKEN@github.com/<your-repo-name>/infrastructure-capstone.git
cd /home/ubuntu/infrastructure-capstone
apt-get install python3.10-dev default-libmysqlclient-dev -y
pip3 install -r requirements.txt
cd /home/ubuntu/infrastructure-capstone/src
python3 manage.py collectstatic --noinput
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:80
```

In a browser check:
    - http://<public_dns_of_test_server>

If everything works, terminate the test server, RDS Database otherwise repeat or fix the steps.

 
## Step 10: Create SDA-Admin-Node EC2 Instance (JumpBox/Ansible Control Node/Monitoring/Management)

This instance serves as a centralized `JumpBox, Ansible Control Node, and Monitoring Server` using `Prometheus and Grafana`. We will use this for administrative tasks.

Create a security group:
- Name              : `sda-capstone-admin-node-sg`
- Ports             : `Port 22, 3000, 9090, 9100` ---> `Anywhere(0.0.0.0/0)`

EC2 Configuration   :
- Name              : `SDA-Admin-Node`
- AMI               : `Amazon Linux 2023`
- Instance Type     : `t3.medium`
- keypair           : `your-pem-key`
- Subnet            : `sda-capstone-public-1a`
- Security Group    : `sda-capstone-admin-node-sg`
- Advanced details  :
    - IAM Instance Profile: `sda-capstone-ec2-ssm-s3-full-access`

`Remote-SSH` into `SDA-Admin-Node` and make the following instalations:

```bash
# Set Hostname
sudo hostnamectl set-hostname SDA-Admin-Node && bash

# Install System Dependencies
sudo dnf update -y
sudo dnf install -y git curl wget unzip tar

# Install Terraform
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform
terraform --version

# Install Ansible
sudo dnf install -y ansible
ansible --version

# Boto3 and Botocore for Dynamic Inventory
sudo dnf install -y python3-pip
pip3 install --upgrade boto3 botocore

# Install Graphana
sudo yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.4.0-1.x86_64.rpm
sudo systemctl start grafana-server.service

# Install Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v3.0.1/prometheus-3.0.1.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
cd prometheus-*-amd64/
```

- You can start Prometheus with following command when you need it.

```bash
./prometheus --config.file=prometheus.yml
```


## Step 11: Create The Infrastructure With Terraform

Prepare terraform files to create HA infrastructure on AWS! TG, ALB, LT, ASG, IAM role with Policies, listeners, subnet group, RDS ...

```bash
cd && mkdir sda-capstone-infra-tf && cd sda-capstone-infra-tf
```
Create `userdata.sh`:

```bash
touch userdata.sh
```
```sh

#!/bin/bash
apt-get update -y
apt-get upgrade -y
apt-get install git -y
apt-get install python3 -y
apt install python3-pip -y
pip3 install boto3
apt  install awscli -y
cd /home/ubuntu/
TOKEN=$(aws --region=eu-central-1 ssm get-parameter --name ${ssm-git-token} --query 'Parameter.Value' --output text)
git clone https://$TOKEN@github.com/<your-repo-name>/infrastructure-capstone.git
cd /home/ubuntu/infrastructure-capstone
apt-get install python3.10-dev default-libmysqlclient-dev -y
pip3 install -r requirements.txt
cd /home/ubuntu/infrastructure-capstone/src
python3 manage.py collectstatic --noinput
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:80
```
-------------<>-------------
-------------<>-------------
Create `providers.tf`:

```bash
touch providers.tf
```
```h

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    github = {
      source = "integrations/github"
      version = "6.6.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region  = "eu-central-1"
}

provider "github" {
  token = data.aws_ssm_parameter.git_token.value
}

```
-------------<>-------------
-------------<>-------------
Create `variables.tf`:

```bash
touch variables.tf
```
```h

variable "key-name" {
  default = "firstkey"
}

variable "ec2-type" {
  default = "t3.micro"
}

variable "ssm_db_username" {
  description = "SSM path for DB username"
  type        = string
  default     = "/<yourname>/capstone/username"
}

variable "ssm_db_password" {
  description = "SSM path for DB password"
  type        = string
  default     = "/<yourname>/capstone/password"
}

variable "ssm_git_token" {
  description = "SSM path for GitHub token"
  type        = string
  default     = "/<yourname>/capstone/token"
}

```
-------------<>-------------
-------------<>-------------
Create `sec-gr.tf`:

```bash
touch sec-gr.tf
```
```h

resource "aws_security_group" "alb-sg" {
  name = "TF-sda-capstone-alb-sg"
  tags = {
    Name = "TF_ALBSecurityGroup"
  }
  vpc_id = data.aws_vpc.selected.id

  ingress {
    from_port   = 80
    protocol    = "tcp"
    to_port     = 80
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    protocol    = "-1"
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }

}

resource "aws_security_group" "server-sg" {
  name   = "TF-sda-capstone-ec2-sg"
  vpc_id = data.aws_vpc.selected.id
  tags = {
    Name = "TF_WebServerSecurityGroup"
  }

  ingress {
    from_port       = 80
    protocol        = "tcp"
    to_port         = 80
    security_groups = [aws_security_group.alb-sg.id]
  }

  ingress {
    from_port   = 22
    protocol    = "tcp"
    to_port     = 22
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 9100
    protocol    = "tcp"
    to_port     = 9100
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    protocol    = "-1"
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }

}

resource "aws_security_group" "db-sg" {
  name   = "TF-sda-capstone-rds-sg"
  vpc_id = data.aws_vpc.selected.id
  tags = {
    "Name" = "TF_RDSSecurityGroup"
  }

  ingress {
    from_port       = 3306
    protocol        = "tcp"
    to_port         = 3306
    security_groups = [aws_security_group.server-sg.id]
  }

  egress {
    from_port   = 0
    protocol    = "-1"
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }

}

```
-------------<>-------------
-------------<>-------------
Create `outputs.tf`:

```bash
touch outputs.tf
```
```h

output "dns-name" {
  value = "http://${aws_alb.app-lb.dns_name}"
}

output "db-addr" {
  value = aws_db_instance.db-server.address
}

output "db-endpoint" {
  value = aws_db_instance.db-server.endpoint
}

```
-------------<>-------------
-------------<>-------------
Create `iam.tf`:

```bash
touch iam.tf
```
```h

resource "aws_iam_role" "s3_ssm_ec2_role" {
  name = "s3-ssm-ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
        Effect = "Allow",
        Sid    = ""
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ssm_full_access" {
  role       = aws_iam_role.s3_ssm_ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMFullAccess"
}

resource "aws_iam_role_policy_attachment" "s3_full_access" {
  role       = aws_iam_role.s3_ssm_ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_instance_profile" "ssm_instance_profile" {
  name = "ssm-ec2-instance-profile"
  role = aws_iam_role.s3_ssm_ec2_role.name
}

```
-------------<>-------------
-------------<>-------------

Create `data.tf`:

```bash
touch data.tf
```
```h

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/vpc
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Name"
    values = ["sda-capstone-vpc"]
  }
}

data "aws_subnets" "public" {
  filter {
    name   = "tag:Name"
    values = ["sda-capstone-public-subnet-*"]
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "tag:Name"
    values = ["sda-capstone-private-subnet-*"]
  }
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
}

data "aws_ssm_parameter" "db_username" {
  name            = var.ssm_db_username
  with_decryption = false
}

data "aws_ssm_parameter" "db_password" {
  name            = var.ssm_db_password
  with_decryption = false
}

data "aws_ssm_parameter" "git_token" {
  name            = var.ssm_git_token
  with_decryption = false
}

```
-------------<>-------------
-------------<>-------------
Create `main.tf`:

```bash
touch main.tf
```
```h

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group

resource "aws_alb_target_group" "app-lb-tg" {
  name        = "blog-lb-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = data.aws_vpc.selected.id
  target_type = "instance"

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb
resource "aws_alb" "app-lb" {
  name               = "blog-lb-tf"
  ip_address_type    = "ipv4"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb-sg.id]
  subnets            = data.aws_subnets.public.ids
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener
resource "aws_alb_listener" "app-listener" {
  load_balancer_arn = aws_alb.app-lb.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_alb_target_group.app-lb-tg.arn
  }
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template
# https://developer.hashicorp.com/terraform/language/functions/templatefile
resource "aws_launch_template" "asg-lt" {
  name                   = "blog-lt"
  image_id               = "ami-04542995864e26699"
  instance_type          = var.ec2-type
  key_name               = var.key-name
  vpc_security_group_ids = [aws_security_group.server-sg.id]
  user_data              = base64encode(templatefile("userdata.sh", { ssm-git-token = var.ssm_git_token }))
  iam_instance_profile {
    name = aws_iam_instance_profile.ssm_instance_profile.name
  }
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "Web Server of Blog App"
      App = "BlogPage"
      Project = "Capstone"
    }
  }
  depends_on = [github_repository_file.settings_py]
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group
resource "aws_autoscaling_group" "app-asg" {
  max_size                  = 3
  min_size                  = 2
  desired_capacity          = 2
  name                      = "blog-asg"
  health_check_grace_period = 300
  health_check_type         = "ELB"
  target_group_arns         = [aws_alb_target_group.app-lb-tg.arn]
  vpc_zone_identifier       = aws_alb.app-lb.subnets

  launch_template {
    id      = aws_launch_template.asg-lt.id
    version = aws_launch_template.asg-lt.latest_version
  }
}

resource "aws_db_subnet_group" "db-subnet-group" {
  name       = "db-subnet-group"
  subnet_ids = data.aws_subnets.private.ids

  tags = {
    Name = "TF_DBSubnetGroup"
  }
}

# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance
resource "aws_db_instance" "db-server" {
  instance_class              = "db.t3.micro"
  allocated_storage           = 20
  vpc_security_group_ids      = [aws_security_group.db-sg.id]
  db_subnet_group_name        = aws_db_subnet_group.db-subnet-group.name
  allow_major_version_upgrade = false
  auto_minor_version_upgrade  = true
  backup_retention_period     = 0
  identifier                  = "blog-app-db"
  db_name                     = "clarusway"
  engine                      = "mysql"
  engine_version              = "8.0.35"
  username                    = data.aws_ssm_parameter.db_username.value
  password                    = data.aws_ssm_parameter.db_password.value
  monitoring_interval         = 0
  multi_az                    = false
  port                        = 3306
  publicly_accessible         = false
  skip_final_snapshot         = true
}

resource "github_repository_file" "settings_py" {
  repository          = "infrastructure-capstone"
  branch              = "main"
  file                = "src/cblog/settings.py"
  content             = templatefile("${path.module}/settings.py.tmpl", {db_endpoint = aws_db_instance.db-server.address})
  commit_message      = "Update settings.py with DB endpoint"
  overwrite_on_create = true
  depends_on = [aws_db_instance.db-server]
  // lifecycle { 
  //   prevent_destroy = true # this will prevent settings.py file to be deleted from capstone github repo!
  // }
}

```
-------------<>-------------
-------------<>-------------
Create `settings.py.tmpl`:

```bash
touch settings.py.tmpl
```
```py

"""
Django settings for cblog project.

Generated by 'django-admin startproject' using Django 3.1.4.

For more information on this file, see
https://docs.djangoproject.com/en/3.1/topics/settings/

For the full list of settings and their values, see
https://docs.djangoproject.com/en/3.1/ref/settings/
"""

from pathlib import Path
from decouple import config
import os
import boto3

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent


# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/3.1/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = config('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = False

ALLOWED_HOSTS = ['*']


# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # my_apps
    'blog.apps.BlogConfig',
    'users.apps.UsersConfig',

    # third party
    'crispy_forms',
    'storages'
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'cblog.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR, "templates"],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'cblog.wsgi.application'

def get_ssm_parameters():
    ssm = boto3.client('ssm', region_name='eu-central-1')

    # AWS SSM Parametr define
    username_param = ssm.get_parameter(Name="/<yourname>/capstone/username", WithDecryption=False)
    password_param = ssm.get_parameter(Name="/<yourname>/capstone/password", WithDecryption=False)


    # Parametre retrieve
    username = username_param['Parameter']['Value']
    password = password_param['Parameter']['Value']

    return username, password

# SSM put
db_username, db_password = get_ssm_parameters()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'clarusway',
        'USER': db_username,
        'PASSWORD': db_password,
        'HOST': '${db_endpoint}',
        'PORT': '3306',
    }
}

# Password validation
# https://docs.djangoproject.com/en/3.1/ref/settings/#auth-password-validators

AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]


# Internationalization
# https://docs.djangoproject.com/en/3.1/topics/i18n/

LANGUAGE_CODE = 'en-us'

TIME_ZONE = 'UTC'

USE_I18N = True

USE_L10N = True

USE_TZ = True


# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/3.1/howto/static-files/

STATIC_URL = '/static/'
MEDIA_URL = '/media/'
STATICFILES_DIRS = [BASE_DIR / "static"]

MEDIA_ROOT = BASE_DIR / "media_root"

CRISPY_TEMPLATE_PACK = 'bootstrap4'

LOGIN_REDIRECT_URL = "blog:list"

LOGIN_URL = "login"


AWS_STORAGE_BUCKET_NAME = 'sdacapstone-<yourname>-blog' # please enter your s3 bucket name
AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % AWS_STORAGE_BUCKET_NAME
AWS_S3_REGION_NAME = "eu-central-1" # please enter your s3 region 
AWS_DEFAULT_ACL = 'public-read'

AWS_LOCATION = 'static'
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'static'),
]

STATIC_URL = 'https://%s/%s/' % (AWS_S3_CUSTOM_DOMAIN, AWS_LOCATION)
STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
DEFAULT_FILE_STORAGE = 'cblog.storages.MediaStore'
# new line

```

`-----WARNING-----    !    -----WARNING-----     !     -----WARNING-----    !    -----WARNING-----`

In settings.py.tmpl file, update line 159,87,86 AWS_STORAGE_BUCKET_NAME=sdacapstone-<yourname>-blog
In variables.tf file, update all the necessary default values.

```bash
terraform init
terraform apply
```
Go to `aws console` and check the resources created with terraform. 

Visit `ALB DNS` to see the application. !!! YOU CAN NOT USE EC2 PUBLIC IP (sec grp configuration) !!! `Sign Up` and click `New Post` to create a new post so that you can be sure that your application can use database and s3 bucket.


Congrats! You successfully created the infrastructure and deployed your application using terraform.

After this point, resources must be monitored and maintained. Prometheus can collect ec2 related metrics if only metric-server is installed in target machines. We will configure ansible in SDA-Admin-Node and use it to make necassary instalations in target machines.
