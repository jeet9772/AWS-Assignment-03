# AWS Nginx High Availability, Auto Scaling, S3, ALB & CloudFront Assignment

## Project Overview

This project demonstrates how to deploy and manage a highly available Nginx-based web application on AWS.

The implementation covers:

* Nginx Reverse Proxy
* AMI-based version management
* Nginx V1 and V2
* Auto Scaling Group (ASG)
* Application Load Balancer (ALB)
* Scaling Policies
* Load Testing
* Rolling Deployment
* Rollback
* S3 Static Assets
* IAM Roles and Policies
* Bastion Host
* ALB Path-Based Routing
* Blue-Green Deployment
* CloudFront CDN
* Instance Health Recovery
* AWS CLI without hardcoded Access Keys
* Deployment automation utility

---

# Architecture Overview

```text
                         Internet
                            |
                            |
                       Public IP
                            |
                    +----------------+
                    |      ALB       |
                    |  Port 80/443   |
                    +-------+--------+
                            |
             +--------------+--------------+
             |                             |
        Target Group 1                Target Group 2
             |                             |
       +-----+------+               +------+-----+
       |   Nginx    |               |   Nginx    |
       |   Server 1 |               |   Server 2 |
       +-----+------+               +------+-----+
             |                             |
             +-------------+---------------+
                           |
                         S3
                    Static Images
                           |
                       CloudFront
```

---

# Prerequisites

## AWS Services

* EC2
* VPC
* Subnets
* Security Groups
* AMI
* Auto Scaling Group
* Application Load Balancer
* Target Groups
* S3
* IAM
* CloudFront
* CloudWatch

## Software

* Nginx
* Git
* AWS CLI
* curl
* stress / stress-ng
* Linux

---

# Important Security Requirements

The following security requirements must be maintained throughout the assignment.

1. SSH to Bastion Host should be allowed only from my public IP.
2. SSH to private Nginx servers should be allowed only from Bastion Host Security Group.
3. Nginx port 80 should accept traffic only from ALB Security Group.
4. ALB port 80 should be accessible only from my public IP.
5. AWS Access Key and Secret Access Key should not be hardcoded on EC2.
6. EC2 should use IAM Role / Instance Profile for AWS CLI access.
7. IAM permissions should follow the principle of least privilege.
8. S3 bucket access should be controlled through IAM and bucket policies.
9. Production and non-production objects should be separated.

---

# DAY 1 - Nginx, AMI, ASG, ALB and Scaling

## Step 1 - Create EC2 Instance

Create an EC2 instance.

Example:

```text
Name: nginx-base
AMI: Amazon Linux / Ubuntu
Instance Type: t2.micro / t3.micro
Subnet: Public subnet initially
Security Group: SSH + HTTP
```

Connect to the instance:

```bash
ssh -i key.pem ec2-user@<PUBLIC-IP>
```

---

# Step 2 - Install Nginx

For Amazon Linux:

```bash
sudo dnf update -y
sudo dnf install nginx -y
```

For Ubuntu:

```bash
sudo apt update
sudo apt install nginx -y
```

Start Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Check status:

```bash
sudo systemctl status nginx
```

Test:

```bash
curl http://localhost
```

---

# Step 3 - Create Base AMI

After configuring the base Nginx server:

AWS Console:

```text
EC2
  |
  +-- Instances
       |
       +-- Select nginx-base
            |
            +-- Actions
                 |
                 +-- Image and templates
                      |
                      +-- Create Image
```

Create:

```text
AMI Name: nginx-base-ami
```

This AMI will be used as the base image.

---

# Step 4 - Create V1

Launch a new EC2 instance from the base AMI.

Example:

```text
Instance Name: nginx-v1
AMI: nginx-base-ami
```

Modify the Nginx webpage:

```bash
sudo vim /usr/share/nginx/html/index.html
```

Example:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Nginx V1</title>
</head>
<body>
    <h1>Nginx Version V1</h1>
    <p>This is Nginx Version 1.</p>
</body>
</html>
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

Verify:

```bash
curl http://localhost
```

---

# Step 5 - Create AMI-1

Create an AMI from the V1 instance.

```text
AMI Name:
nginx-v1-ami
```

---

# Step 6 - Create V2

Launch another EC2 instance from AMI-1.

```text
Instance Name: nginx-v2
AMI: nginx-v1-ami
```

Make changes:

```bash
sudo vim /usr/share/nginx/html/index.html
```

Example:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Nginx V2</title>
</head>
<body>
    <h1>Nginx Version V2</h1>
    <p>This is the upgraded Nginx version.</p>
</body>
</html>
```

Restart:

```bash
sudo systemctl restart nginx
```

Verify:

```bash
curl http://localhost
```

---

# Step 7 - Create AMI-2

Create AMI from V2:

```text
AMI Name:
nginx-v2-ami
```

At this stage we have:

```text
AMI-1 -> Nginx V1
AMI-2 -> Nginx V2
```

---

# Step 8 - Create Target Group

Create target group:

```text
Target Type: Instances
Protocol: HTTP
Port: 80
Health Check Path: /
```

---

# Step 9 - Create Application Load Balancer

Create:

```text
Load Balancer Type:
Application Load Balancer
```

Configuration:

```text
Scheme: Internet-facing
Listener: HTTP : 80
```

Attach target group.

Verify:

```text
http://<ALB-DNS>
```

---

# Step 10 - Create Launch Template

Create launch template using the Nginx AMI.

Example:

```text
Launch Template:
nginx-v1-template

AMI:
nginx-v1-ami

Instance Type:
t3.micro

Security Group:
nginx-sg
```

---

# Step 11 - Create Auto Scaling Group

Create ASG:

```text
ASG Name:
nginx-asg

Launch Template:
nginx-v1-template
```

Example:

```text
Minimum Capacity: 2
Desired Capacity: 2
Maximum Capacity: 5
```

Select multiple Availability Zones.

Attach the ALB target group.

Enable:

```text
ELB Health Checks
```

---

# Step 12 - Configure Scaling Policies

## Policy 1 - Average CPU Utilization

Example:

```text
Target Tracking
Metric:
Average CPU Utilization

Target:
50%
```

When CPU increases above the target, ASG can add instances.

When demand decreases, ASG can reduce instances according to the configured limits.

---

# Step 13 - Network Traffic Scaling

Use CloudWatch metrics such as:

```text
NetworkIn
NetworkOut
```

Create an appropriate scaling policy based on the selected metric.

---

# Step 14 - ALB Request Count Per Target

Use:

```text
ALB
  |
  +-- Target Group
       |
       +-- RequestCountPerTarget
```

Configure scaling using:

```text
RequestCountPerTarget
```

---

# Step 15 - Load Testing

Install stress tool.

Amazon Linux:

```bash
sudo dnf install stress -y
```

Ubuntu:

```bash
sudo apt install stress -y
```

Generate CPU load:

```bash
stress --cpu 2 --timeout 300
```

Or:

```bash
stress-ng --cpu 2 --timeout 300s
```

Monitor:

```text
CloudWatch
   |
   +-- CPUUtilization
   +-- NetworkIn
   +-- NetworkOut
   +-- RequestCountPerTarget
```

Observe ASG activity:

```text
Desired Capacity
InService Instances
Pending Instances
Terminating Instances
```

---

# Step 16 - Test Nginx V2 Deployment

Update the Launch Template from V1 AMI to V2 AMI.

Example:

```text
Old:
nginx-v1-ami

New:
nginx-v2-ami
```

Use the updated launch template for the ASG.

Test the application through:

```text
http://<ALB-DNS>
```

---

# Step 17 - Rollback V2 to V1

Client reports that V2 is incompatible.

Rollback:

```text
V2 AMI
   |
   X
   |
V1 AMI
```

Update the Launch Template to:

```text
nginx-v1-ami
```

Perform an instance refresh / rolling replacement according to the deployment configuration.

Verify:

```bash
curl http://<ALB-DNS>
```

Expected:

```text
Nginx V1
```

---

# DAY 2 - Nginx Web Hosting + S3

## Objective

Nginx should now host a webpage.

Images should be:

```text
Git Repository
       |
       v
     EC2
       |
       v
    AWS CLI
       |
       v
      S3
```

---

# Step 1 - Clone Git Repository

From EC2:

```bash
git clone <GIT-REPOSITORY-URL>
```

Enter repository:

```bash
cd <repository-name>
```

---

# Step 2 - Configure IAM Role

Do NOT use:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Instead attach an IAM Role to EC2.

Example role:

```text
nginx-s3-role
```

Required permissions should be limited to the required S3 operations.

Verify:

```bash
aws sts get-caller-identity
```

---

# Step 3 - Create S3 Bucket

Create bucket:

```bash
aws s3 mb s3://<bucket-name> --region ap-south-1
```

For US-East-1:

```bash
aws s3 mb s3://<bucket-name> --region us-east-1
```

---

# Step 4 - Create S3 Folders

```bash
aws s3api put-object \
--bucket <bucket-name> \
--key assets/
```

Upload images:

```bash
aws s3 cp ./images/ s3://<bucket-name>/assets/ --recursive
```

Verify:

```bash
aws s3 ls s3://<bucket-name>/assets/
```

---

# Step 5 - Configure Nginx Webpage

Edit:

```bash
sudo vim /usr/share/nginx/html/index.html
```

Example:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Nginx AWS Website</title>
</head>

<body>

<h1>Welcome to Nginx Web Server</h1>

<p>This website is hosted using Nginx on AWS.</p>

<img src="https://<BUCKET-NAME>.s3.<REGION>.amazonaws.com/assets/image1.jpg">

<img src="https://<BUCKET-NAME>.s3.<REGION>.amazonaws.com/assets/image2.jpg">

</body>
</html>
```

Restart:

```bash
sudo systemctl restart nginx
```

---

# DAY 3 - ASG Health Check and Automation

## Objective

Test whether Auto Scaling can maintain the desired number of healthy instances.

---

# Step 1 - Make Instance Unhealthy

Connect to an ASG instance.

Example:

```bash
sudo systemctl stop nginx
```

Now Nginx becomes unhealthy from the ALB health-check perspective if the health check depends on Nginx.

Check:

```bash
sudo systemctl status nginx
```

---

# Step 2 - Observe Target Health

AWS Console:

```text
EC2
  |
  +-- Target Groups
       |
       +-- Targets
```

Observe:

```text
Healthy
Unhealthy
```

---

# Step 3 - Observe Auto Scaling

Go to:

```text
EC2
  |
  +-- Auto Scaling Groups
```

Check:

```text
Desired Capacity
Current Capacity
Healthy Instances
Unhealthy Instances
```

ASG should replace an unhealthy instance when configured health checks and replacement behavior detect the unhealthy instance.

---

# Step 4 - AMI Version Utility

Create a utility to manage AMI versions.

Example workflow:

```text
Create AMI
     |
     v
Create Launch Template Version
     |
     v
Attach Launch Template to ASG
     |
     v
Rolling Deployment
     |
     v
Health Check
```

---

# Step 5 - Rollback Utility

Rollback workflow:

```text
Current Version
      |
      v
Detect Issue
      |
      v
Select Previous AMI
      |
      v
Create/Select Launch Template Version
      |
      v
Update ASG
      |
      v
Rolling Replacement
      |
      v
Validate
```

---

# Manual First Requirement

Before automation, perform the complete process manually.

Manual process:

```text
EC2
 ↓
Nginx
 ↓
AMI
 ↓
V1
 ↓
AMI-1
 ↓
V2
 ↓
AMI-2
 ↓
Launch Template
 ↓
ASG
 ↓
ALB
 ↓
Scaling
 ↓
Deployment
 ↓
Rollback
```

After successful manual implementation, automate the process.

---

# Blue-Green Deployment

## Blue Environment

```text
ALB
 |
 +--> Blue Target Group
        |
        +--> Nginx V1
```

## Green Environment

```text
ALB
 |
 +--> Green Target Group
        |
        +--> Nginx V2
```

Deployment:

```text
Blue = V1
Green = V2
```

Test Green environment.

If validation succeeds, traffic can be switched according to the deployment design.

If a problem is detected, traffic can be directed back to the previous environment.

---

# CloudFront Integration

Architecture:

```text
User
 |
 v
CloudFront
 |
 v
ALB
 |
 v
Nginx
 |
 v
S3
```

CloudFront helps cache and deliver content from edge locations closer to users.

Create CloudFront distribution with the required origin.

Possible origin:

```text
ALB DNS
```

For images, another distribution/origin can be configured according to the required design.

---

# DAY 4 - ALB Path-Based Routing

## Objective

Use one ALB DNS for two different applications.

Example:

```text
http://<ALB-DNS>/ninja1
http://<ALB-DNS>/ninja2
```

Architecture:

```text
                     ALB
                      |
            +---------+---------+
            |                   |
        /ninja1              /ninja2
            |                   |
            v                   v
       Target Group 1      Target Group 2
            |                   |
            v                   v
        Nginx EC2-1        Nginx EC2-2
        Private Subnet     Private Subnet
```

---

# Step 1 - Create Bastion Host

Create:

```text
Bastion Host
Public Subnet
```

Security Group:

```text
SSH : 22
Source:
YOUR_PUBLIC_IP/32
```

Do not allow:

```text
0.0.0.0/0
```

for SSH.

---

# Step 2 - Create Private Nginx Servers

Create:

```text
Nginx-1 -> Private Subnet 1
Nginx-2 -> Private Subnet 2
```

SSH should be allowed only from:

```text
Bastion Security Group
```

---

# Step 3 - Configure Nginx-1

Configure:

```text
/ninja1
```

Example:

```bash
sudo mkdir -p /usr/share/nginx/html/ninja1
sudo vim /usr/share/nginx/html/ninja1/index.html
```

Example:

```html
<!DOCTYPE html>
<html>
<body>
<h1>Ninja 1</h1>
<img src="https://<BUCKET-NAME>.s3.<REGION>.amazonaws.com/images/Image-1.jpg">
</body>
</html>
```

---

# Step 4 - Configure Nginx-2

```bash
sudo mkdir -p /usr/share/nginx/html/ninja2
sudo vim /usr/share/nginx/html/ninja2/index.html
```

Example:

```html
<!DOCTYPE html>
<html>
<body>
<h1>Ninja 2</h1>
<img src="https://<BUCKET-NAME>.s3.<REGION>.amazonaws.com/images/Image-2.jpg">
</body>
</html>
```

---

# Step 5 - Create Target Groups

Create two target groups.

## Target Group 1

```text
Name: nginx-ninja1-tg
Protocol: HTTP
Port: 80
Target: Nginx-1
Health Check: /
```

## Target Group 2

```text
Name: nginx-ninja2-tg
Protocol: HTTP
Port: 80
Target: Nginx-2
Health Check: /
```

---

# Step 6 - Create ALB

```text
Type:
Application Load Balancer

Scheme:
Internet-facing

Listener:
HTTP : 80
```

---

# Step 7 - Configure ALB Listener Rules

Rule 1:

```text
IF Path = /ninja1/*
THEN Forward to nginx-ninja1-tg
```

Rule 2:

```text
IF Path = /ninja2/*
THEN Forward to nginx-ninja2-tg
```

---

# Step 8 - Security Groups

## Bastion SG

```text
Inbound:
SSH 22 -> YOUR_PUBLIC_IP/32
```

## Nginx SG

```text
Inbound:
SSH 22 -> Bastion SG
HTTP 80 -> ALB SG
```

## ALB SG

```text
Inbound:
HTTP 80 -> YOUR_PUBLIC_IP/32
```

This ensures that Nginx port 80 is not directly accessible from the Internet.

---

# Step 9 - Test

Open:

```text
http://<ALB-DNS>/ninja1
```

Expected:

```text
Ninja 1
Image-1
```

Open:

```text
http://<ALB-DNS>/ninja2
```

Expected:

```text
Ninja 2
Image-2
```

---

# Step 10 - Upload Images to S3

From an EC2 instance:

```bash
aws s3 cp Image-1.jpg s3://<bucket-name>/images/
aws s3 cp Image-2.jpg s3://<bucket-name>/images/
```

Verify:

```bash
aws s3 ls s3://<bucket-name>/images/
```

---

# DAY 5 - IAM and S3 Environment Separation

## Objective

Create separate environments:

```text
S3 Bucket
 |
 +-- prod/
 |
 +-- nonprod/
```

---

# Step 1 - Create S3 Bucket in US-East-1

```bash
aws s3 mb s3://<bucket-name> --region us-east-1
```

For `us-east-1`, region handling differs from most other regions, so use the AWS CLI command appropriate for that region.

---

# Step 2 - Create Folders

```bash
aws s3api put-object \
--bucket <bucket-name> \
--key prod/
```

```bash
aws s3api put-object \
--bucket <bucket-name> \
--key nonprod/
```

---

# Step 3 - Upload Images

```bash
aws s3 cp prod-image.jpg s3://<bucket-name>/prod/
```

```bash
aws s3 cp nonprod-image.jpg s3://<bucket-name>/nonprod/
```

Verify:

```bash
aws s3 ls s3://<bucket-name>/prod/
aws s3 ls s3://<bucket-name>/nonprod/
```

---

# Step 4 - Create IAM User

Create:

```text
IAM User:
s3-nonprod-user
```

The user should have access only to the required `nonprod/` prefix.

The user should NOT have access to:

```text
prod/
```

---

# Step 5 - IAM Policy

Example policy concept:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::BUCKET-NAME/nonprod/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::BUCKET-NAME",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "nonprod/*"
          ]
        }
      }
    }
  ]
}
```

Replace:

```text
BUCKET-NAME
```

with the actual bucket name.

---

# Step 6 - IAM Role for EC2

Create:

```text
Role:
nginx-s3-role
```

Attach only the permissions required by the Nginx/EC2 application.

Attach the role to EC2 using an Instance Profile.

Test:

```bash
aws sts get-caller-identity
```

---

# Step 7 - Bucket Policy

Use a bucket policy to control access to the bucket.

The policy should allow the required principals only.

Required access may include:

```text
Root account
IAM user
EC2/Nginx IAM Role
```

Avoid making the complete bucket public unless specifically required.

---

# Step 8 - Test IAM Restrictions

Test non-production:

```bash
aws s3 ls s3://<bucket-name>/nonprod/
```

Expected:

```text
Access Allowed
```

Test production:

```bash
aws s3 ls s3://<bucket-name>/prod/
```

Expected for restricted IAM user:

```text
Access Denied
```

---

# DAY 6 - CloudFront, IAM Trust Relationship and Least Privilege

## Objective

Implement CDN for faster content delivery and validate IAM role/trust configuration.

---

# Step 1 - Understand IAM Policy Types

There are two important policy concepts:

## Identity-Based Policy

Defines:

```text
What actions can the identity perform?
```

Example:

```text
s3:GetObject
```

## Trust Policy

Defines:

```text
Who can assume/use the role?
```

Example:

```text
Service Principal
```

The trust relationship is attached to the IAM Role.

---

# Step 2 - Validate Role Trust Relationship

Check:

```text
IAM
 |
 +-- Roles
      |
      +-- Select Role
           |
           +-- Trust relationships
```

Verify that the correct AWS service/principal is configured for the intended use.

---

# Step 3 - CloudFront

Architecture:

```text
User
 |
 v
CloudFront
 |
 v
Origin
 |
 +------> S3
 |
 +------> ALB
```

CloudFront caches content at edge locations.

---

# Step 4 - Create CloudFront Distribution

Create:

```text
CloudFront Distribution
```

Configure the appropriate origin.

For S3 content:

```text
Origin:
S3 Bucket
```

For application content:

```text
Origin:
ALB DNS
```

---

# Step 5 - Protect S3 Origin

Where applicable, configure CloudFront origin access using the supported CloudFront/S3 origin access mechanism.

The objective is:

```text
User
  |
  v
CloudFront
  |
  v
S3
```

instead of directly accessing the S3 object URL.

---

# Step 6 - Test CloudFront

CloudFront provides a domain such as:

```text
https://<distribution-id>.cloudfront.net
```

Test:

```text
https://<distribution-id>.cloudfront.net/image.jpg
```

Check CloudFront response and caching behavior.

---

# Step 7 - Create Personal IAM User

Create an IAM user for administration/testing.

Follow least privilege.

Do not assign:

```text
AdministratorAccess
```

unless it is genuinely required.

Give only the permissions required for the current task.

---

# Complete Final Architecture

```text
                         USER
                           |
                           v
                     CloudFront CDN
                           |
                           v
                         ALB
                           |
              +------------+------------+
              |                         |
        /ninja1 path               /ninja2 path
              |                         |
              v                         v
       Target Group 1             Target Group 2
              |                         |
              v                         v
        Nginx EC2-1               Nginx EC2-2
       Private Subnet             Private Subnet
              |                         |
              +------------+------------+
                           |
                           v
                          S3
                           |
                +----------+----------+
                |                     |
              prod/                nonprod/
```

---

# AMI Versioning Architecture

```text
Base Nginx
    |
    v
AMI-Base
    |
    +------> V1
    |          |
    |          v
    |       AMI-V1
    |
    +------> V2
               |
               v
            AMI-V2
```

---

# Deployment Strategy

```text
                    Launch Template
                           |
                    +------+------+
                    |             |
                  V1 AMI        V2 AMI
                    |             |
                    v             v
                  Blue          Green
                    |             |
                    +------+------+
                           |
                           v
                           ALB
```

---

# Rollback Strategy

```text
V2 Deployment
     |
     v
Health Check
     |
     v
Problem Detected
     |
     v
Select V1 AMI
     |
     v
Update Launch Template
     |
     v
ASG Instance Refresh
     |
     v
V1 Restored
```

---

# Health Check Strategy

The following should be monitored:

## EC2

```text
StatusCheckFailed
CPUUtilization
NetworkIn
NetworkOut
```

## ALB

```text
RequestCount
RequestCountPerTarget
HTTPCode_Target_2XX_Count
HTTPCode_Target_4XX_Count
HTTPCode_Target_5XX_Count
TargetResponseTime
HealthyHostCount
UnHealthyHostCount
```

## ASG

```text
GroupDesiredCapacity
GroupInServiceInstances
GroupPendingInstances
GroupTerminatingInstances
```

---

# Testing Checklist

## Day 1

* [ ] EC2 created
* [ ] Nginx installed
* [ ] Base AMI created
* [ ] V1 created
* [ ] AMI-1 created
* [ ] V2 created
* [ ] AMI-2 created
* [ ] Target Group created
* [ ] ALB created
* [ ] Launch Template created
* [ ] ASG created
* [ ] Scaling policy configured
* [ ] CPU load tested
* [ ] Network metrics checked
* [ ] ALB request count checked
* [ ] V1 to V2 upgrade tested
* [ ] V2 to V1 rollback tested

## Day 2

* [ ] Git repository cloned from EC2
* [ ] IAM Role attached
* [ ] AWS CLI tested
* [ ] S3 bucket created
* [ ] Images uploaded
* [ ] Nginx webpage created
* [ ] S3 images displayed on webpage

## Day 3

* [ ] Nginx health tested
* [ ] Unhealthy instance simulated
* [ ] ALB health check verified
* [ ] ASG replacement verified
* [ ] AMI automation planned
* [ ] Rolling deployment tested
* [ ] Rollback functionality tested
* [ ] Blue-Green deployment tested
* [ ] CloudFront configured

## Day 4

* [ ] Bastion Host created
* [ ] Private Nginx-1 created
* [ ] Private Nginx-2 created
* [ ] SSH restricted to Bastion
* [ ] ALB created
* [ ] Target Group 1 created
* [ ] Target Group 2 created
* [ ] `/ninja1` routing tested
* [ ] `/ninja2` routing tested
* [ ] Images uploaded to S3

## Day 5

* [ ] S3 bucket created in us-east-1
* [ ] `prod/` folder created
* [ ] `nonprod/` folder created
* [ ] Images uploaded
* [ ] IAM user created
* [ ] IAM Role created
* [ ] IAM policy configured
* [ ] Bucket policy configured
* [ ] prod restriction tested
* [ ] nonprod access tested

## Day 6

* [ ] IAM trust relationship checked
* [ ] CloudFront created
* [ ] S3 origin configured
* [ ] ALB origin configured where required
* [ ] CDN access tested
* [ ] Least-privilege IAM user created
* [ ] Final architecture validated

---

# Important AWS CLI Commands

Check identity:

```bash
aws sts get-caller-identity
```

List buckets:

```bash
aws s3 ls
```

List bucket:

```bash
aws s3 ls s3://<bucket-name>
```

Upload file:

```bash
aws s3 cp file.jpg s3://<bucket-name>/folder/
```

Upload directory:

```bash
aws s3 cp ./images/ s3://<bucket-name>/images/ --recursive
```

Download file:

```bash
aws s3 cp s3://<bucket-name>/folder/file.jpg .
```

Sync directory:

```bash
aws s3 sync ./images/ s3://<bucket-name>/images/
```

---

# Final Project Flow

```text
                Git Repository
                      |
                      v
                     EC2
                      |
                  AWS CLI
                      |
                      v
                     S3
                      |
                      v
                CloudFront CDN
                      |
                      v
                     ALB
                      |
          +-----------+-----------+
          |                       |
       /ninja1                 /ninja2
          |                       |
          v                       v
      Nginx-1                  Nginx-2
          |                       |
          +-----------+-----------+
                      |
                     ASG
                      |
              Auto Scaling Policy
                      |
          +-----------+-----------+
          |           |           |
         CPU       Network     Requests
```

---

# Conclusion

This assignment demonstrates a complete AWS-based Nginx infrastructure with:

* High Availability
* Disaster Recovery concepts
* AMI versioning
* Nginx V1/V2
* Auto Scaling
* Load Balancing
* Health Checks
* Scaling Policies
* Rolling Deployment
* Rollback
* Blue-Green Deployment
* S3 Asset Management
* IAM Roles
* IAM Users
* Bucket Policies
* Bastion Host
* Path-Based Routing
* CloudFront CDN
* Least-Privilege Access
* AWS CLI based deployment

The implementation should first be completed manually and validated. After successful manual validation, the deployment, AMI creation, ASG update, rolling deployment and rollback processes can be automated.



day -1 screenshort


<img width="1440" height="900" alt="1" src="https://github.com/user-attachments/assets/7d9f4d42-451c-45a0-bdd7-6edec0393593" />



<img width="1440" height="900" alt="3" src="https://github.com/user-attachments/assets/1e07be12-714a-4786-be14-c30ba3c78720" />


<img width="1440" height="900" alt="3" src="https://github.com/user-attachments/assets/85846fe9-510c-4af2-8404-00e665ef9526" />


<img width="1280" height="800" alt="PHOTO-2026-09-26-09-49-12" src="https://github.com/user-attachments/assets/1cf91fbe-c1da-4e0c-9ee2-e7256bb8f53b" />



<img width="1280" height="800" alt="PHOTO-2026-09-26-09-49-48" src="https://github.com/user-attachments/assets/657ea03a-cb9d-4ec1-a79e-6894752759d3" />


<img width="1280" height="800" alt="PHOTO-2026-09-26-09-51-46" src="https://github.com/user-attachments/assets/6bbbb6d9-b788-4edc-afe0-b23ecb848941" />


<img width="1280" height="800" alt="PHOTO-2026-09-26-09-56-05" src="https://github.com/user-attachments/assets/b8fb75d1-6145-4604-878f-88fb243d5bb2" />


<img width="1280" height="800" alt="PHOTO-2026-09-26-13-36-25" src="https://github.com/user-attachments/assets/035d1326-6a22-483b-923c-7917ee8dafc2" />



<img width="1280" height="800" alt="PHOTO-2026-09-26-14-14-15" src="https://github.com/user-attachments/assets/831d88f4-1905-4ab6-a2f4-5f91407b689d" />




#######.   Day -2  Screenshort########




<img width="1440" height="900" alt="4" src="https://github.com/user-attachments/assets/13ea3d3c-9309-4f7b-9e55-d27a151b66c2" />


<img width="1440" height="900" alt="5" src="https://github.com/user-attachments/assets/83ec715a-dd22-4979-bbda-28edb6d2ef62" />

<img width="1440" height="900" alt="6" src="https://github.com/user-attachments/assets/df79c2b7-eaef-418d-9e2c-a8802d227f95" />








