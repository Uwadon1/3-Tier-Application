
```markdown
# AWS 3-Tier Architecture with Custom Domain


A robust 3-tier application built on AWS with presentation, application, and database tiers, featuring auto-scaling, load balancing, and custom domain configuration via Route 53.

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [AWS Services Used](#aws-services-used)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [1. VPC Creation](#1-vpc-creation)
  - [2. S3 Bucket and IAM Role Setup](#2-s3-bucket-and-iam-role-setup)
  - [3. Database Configuration](#3-database-configuration)
  - [4. Application Tier Setup](#4-application-tier-setup)
  - [5. Web Tier Setup](#5-web-tier-setup)
  - [6. SSL Certification and Domain Mapping](#6-ssl-certification-and-domain-mapping)
- [Application Details](#application-details)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Cleanup Instructions](#cleanup-instructions)

## Architecture Overview

![3-Tier Architecture Diagram](https://i.imgur.com/example-diagram.png)

The application follows a standard 3-tier architecture:

1. **Presentation Tier**:
   - EC2 instances in public subnets
   - Internet-facing Application Load Balancer
   - Auto Scaling group
   - IAM roles for access control

2. **Application Tier**:
   - EC2 instances in private subnets
   - Internal Application Load Balancer
   - IAM roles for access control
   - Node.js application with PM2 process manager

3. **Database Tier**:
   - RDS MySQL instance in private subnets
   - Custom security group restricting access

4. **Security**:
   - Custom VPC with public & private subnets
   - Multiple security groups for each tier
   - Route 53 for custom domain
   - ACM for SSL certificates

## AWS Services Used

- **Compute**: EC2
- **Database**: RDS (MySQL)
- **Networking**: VPC, Subnets, Route 53, ALB
- **Storage**: S3
- **Security**: IAM, Security Groups, ACM
- **Management**: Systems Manager (SSM)

## Prerequisites

1. AWS account with appropriate permissions
2. Registered domain name (for custom domain setup)
3. AWS CLI configured (optional)
4. Basic knowledge of AWS services and Linux commands

## Setup Instructions

### 1. VPC Creation

1. Navigate to VPC service in AWS Console
2. Click "Create VPC" → Select "VPC and more"
3. Configure with these parameters:
   - Name: `my-demo-3-tier-vpc`
   - IPv4 CIDR: `192.168.0.0/22`
   - Number of AZs: `2`
   - Public Subnets: `2`
   - Private Subnets: `4` (2 for app tier, 2 for DB tier)
   - NAT Gateway: `1`
4. After creation, rename subnets appropriately:
   - App1, App2 (for application tier)
   - Web1, Web2 (for web tier)
   - Db1, Db2 (for database tier)

**Security Groups Setup**:

Create 5 security groups with these configurations:

1. **Web-tier-ALB-SG**:
   - Allow HTTP (80) from anywhere

2. **Web-tier-SG**:
   - Allow HTTP (80) from Web-tier-ALB-SG
   - Allow HTTP (80) from VPC CIDR

3. **App-tier-SG**:
   - Allow Custom TCP (4000) from VPC CIDR

4. **App-tier-ALB-SG**:
   - Allow HTTP (80) from VPC CIDR

5. **RDS-SG**:
   - Allow MySQL/Aurora (3306) from VPC CIDR

### 2. S3 Bucket and IAM Role Setup

**S3 Bucket**:
1. Create a new S3 bucket with unique name (e.g., `demo-3tier-bucket-007`)
2. Clone application code:  
   `git clone https://github.com/Uwadon1/3TierArchitectureApp.git`
   https://github.com/Uwadon1/3-Tier-Application.git
3. Upload application code to the S3 bucket

**IAM Role**:
1. Create new IAM role for EC2
2. Select "AmazonEC2RoleforSSM" policy
3. Name the role (e.g., `Demo-EC2-Role`)

### 3. Database Configuration

**RDS Setup**:
1. Create DB subnet group with Db1 and Db2 subnets
2. Launch RDS MySQL instance:
   - Engine: MySQL 8.0.35
   - Template: Free tier
   - DB instance identifier: `database-1`
   - Master username: `admin`
   - Master password: [secure password]
   - VPC: Select custom VPC
   - Security group: RDS-SG
3. Configure database:
   ```sql
   CREATE DATABASE webappdb;
   USE webappdb;
   CREATE TABLE IF NOT EXISTS transactions(
     id INT NOT NULL AUTO_INCREMENT,
     amount DECIMAL(10,2),
     description VARCHAR(100),
     PRIMARY KEY(id)
   );
   INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
   ```

### 4. Application Tier Setup

**EC2 Instances**:
1. Launch 2 EC2 instances in App1 and App2 subnets
   - AMI: Amazon Linux 2
   - Instance type: t2.micro
   - Security group: App-tier-SG
   - IAM role: Demo-EC2-Role
   - No key pair needed (using SSM)

**Application Configuration**:
1. Connect via Session Manager
2. Install dependencies:
   ```bash
   sudo yum install mysql -y
   curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
   source ~/.bashrc
   nvm install 16
   nvm use 16
   npm install -g pm2
   ```
3. Configure DB connection in `DbConfig.js`
4. Start application:
   ```bash
   pm2 start index.js
   pm2 startup
   pm2 save
   ```

**Internal ALB**:
1. Create target group `internal-ALB-TG` (port 4000)
2. Create internal Application Load Balancer:
   - Name: `App-internal-LB`
   - Scheme: internal
   - Subnets: App1, App2
   - Security group: App-tier-ALB-SG
   - Target group: internal-ALB-TG

### 5. Web Tier Setup

**EC2 Instances**:
1. Launch EC2 instances in Web1 and Web2 subnets
   - AMI: Amazon Linux 2
   - Instance type: t2.micro
   - Security group: Web-tier-SG
   - IAM role: Demo-EC2-Role

**Application Setup**:
```bash
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
aws s3 cp s3://demo-3tier-bucket-007/application-code/web-tier/ web-tier --recursive
cd web-tier
npm install
npm run build
sudo amazon-linux-extras install nginx1 -y
cd /etc/nginx
sudo rm nginx.conf
sudo aws s3 cp s3://demo-3tier-bucket-007/application-code/nginx.conf .
sudo service nginx restart
chmod -R 755 /home/ec2-user
sudo chkconfig nginx on
```

**External ALB**:
1. Create target group `external-web-TG` (port 80)
2. Create internet-facing ALB:
   - Name: `Web-External-LB`
   - Scheme: internet-facing
   - Subnets: Web1, Web2
   - Security group: Web-tier-ALB-SG
   - Target group: external-web-TG

### 6. SSL Certification and Domain Mapping

1. Request ACM certificate for your domain
2. Validate via DNS (Route 53)
3. Add HTTPS listener to external ALB (port 443)
4. Select ACM certificate
5. In Route 53, create A record alias pointing to the external ALB

## Application Details

The application consists of:
- Frontend: React.js (served via Nginx)
- Backend: Node.js (running on port 4000)
- Database: MySQL (RDS)

Endpoint paths:
- `/` - Frontend application
- `/api` - Backend API
- `/health` - Health check endpoint

## Security Considerations

1. All private subnets have no internet access except through NAT
2. Database is in isolated private subnets
3. Minimal port openings in security groups
4. IAM roles with least privilege
5. HTTPS enforced via ACM certificates

## Troubleshooting

1. **Cannot connect to private instances**:
   - Verify IAM role is attached
   - Check Systems Manager setup

2. **Database connection issues**:
   - Verify RDS security group allows from app tier
   - Check DB endpoint in `DbConfig.js`

3. **Application not loading**:
   - Check ALB target group health checks
   - Verify security group rules

## Cleanup Instructions

To avoid ongoing charges:
1. Delete RDS instance
2. Terminate all EC2 instances
3. Delete load balancers
4. Remove S3 bucket contents then delete bucket
5. Delete VPC (will remove associated resources)
6. Release Route 53 hosted zone (if no longer needed)
```

This README provides comprehensive documentation for your project with:
1. Clear architecture overview
2. Step-by-step setup instructions
3. Security considerations
4. Troubleshooting guide
5. Cleanup instructions

You can enhance it further by:
- Adding actual screenshots where referenced
- Including the architecture diagram image
- Adding specific error messages and solutions you encountered
- Including cost estimation for the resources


[Read the full implementation process here on Medium](https://medium.com/@uwadon1/building-a-robust-3-tier-architecture-app-using-core-aws-services-with-a-custom-domain-included-bedf8b23133b)
