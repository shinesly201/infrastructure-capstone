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


