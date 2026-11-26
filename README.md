# aws-highly-available-scalable-web-app
This Project demonstrates how modern businesses can host web applications in the cloud while maintaining availability, distributing incoming traffic, automatically scaling computing resources and protecting application infrastructure.


## 1. Project Overview
This project demonstrates the design, deployment, and management of a highly available, scalable web application infrastructure on AWS. Built for a fictional growing technology company, the architecture is designed to handle increased traffic, distribute incoming requests, automatically scale compute resources, and maintain high availability across multiple Availability Zones.

**Core Stack:** 

Amazon EC2

Application Load Balancer (ALB)

EC2 Auto Scaling 

Amazon VPC 

AWS Systems Manager

## 2. Architecture Diagram
[Architecture Diagram](architecture-diagram.png)

**Traffic Flow:**
Internet Users → Application Load Balancer (Public Subnets) → EC2 Auto Scaling Group → EC2 Instances (Private Subnets across 2 AZs) → S3 (website code)


## 3. AWS Services Used

| Service | Role in this Project | Why it was chosen |
| :--- | :--- | :--- |
| **Amazon VPC** | Network isolation | Provides a secure boundary with public subnets for the ALB and private subnets for the EC2 instances. |
| **Amazon EC2** | Compute (Web Servers) | Hosts the web application. Uses `t3.micro` instances for cost-efficiency. |
| **Application Load Balancer** | Traffic distribution | Distributes incoming HTTP/HTTPS traffic across multiple instances to improve availability and fault tolerance. |
| **EC2 Auto Scaling** | Elasticity & Self-Healing | Automatically adjusts the number of instances based on demand and replaces unhealthy instances. |
| **AWS Systems Manager** | Secure Administration | Used Session Manager to access instances securely without opening SSH port 22 or using key pairs. |
| **Amazon S3** | Application Storage | Stores the static application files, which instances pull from during boot via a secure IAM role. |
| **Amazon CloudWatch** | Monitoring | Tracks CPU utilization to trigger Auto Scaling policies and monitor infrastructure health. |

## 4. Infrastructure & Network Design

The infrastructure is deployed in a custom VPC _(10.0.0.0/16)_ across two Availability Zones: _us-east-1a_ and _us-east-1b_ to ensure High Availability.

*   **Public Subnets:** _public-subnet-01-east-1a_ and _public-subnet-02-east-1b_. These host the Application Load Balancer and the NAT Gateway. They route to the Internet Gateway.
  
*   **Private Subnets:** _private-subnet-01-east-1a_ and _private-subnet-02-east-1b_. These host the EC2 instances. They have no inbound internet access.
  
*   **NAT Gateway:** A single NAT Gateway is deployed in a public subnet to provide outbound internet access for the private EC2 instances.
  
    *   *Design Tradeoff:* A single NAT Gateway was chosen over a highly available NAT per AZ to minimize costs for this demonstration project, accepting a slight risk of an AZ failure affecting outbound connectivity.
 
[Network Map](screenshots/vpc-screenshot.png)
 ## 5. Security Configuration

*   **Security Group Referencing:** The EC2 instances were configured in a security group that only allows inbound traffic on ports 80 and 443 from the ALB's security group, not from the public internet. The ALB allows inbound HTTP/HTTPS from the internet.
  
*   **No SSH Access:** Port 22 is completely closed. Administrative access is handled via **AWS Systems Manager Session Manager**, eliminating the need for bastion hosts, key pairs, or public IPs on the instances.

*   **IAM Least Privilege:** The EC2 instances use an IAM Instance Profile with the _AmazonSSMManagedInstanceCore_ policy and a custom inline policy allowing _s3:ListBucket_ and _s3:GetObject_ scoped specifically to the application S3 bucket.

[ALB SG](screenshots/alb-sg-screenshot.png)

[App Server SG](screenshots/app-sg-screenshot.png)

## 6. Application & Launch Template

The application is a static website that includes a dynamic instance ID badge in the bottom right corner (injected via User Data). This badge proves that the ALB is actively distributing traffic across multiple servers.

The **Launch Template** automates instance provisioning:
*   **AMI:** Amazon Linux 2023
*   **Instance Type:** t3.micro
*   **User Data:** Installs Apache ( _httpd_ ), starts the service, uses the AWS CLI to _sync_ files from S3 to _/var/www/html/_.

## 7. Load Balancer Configuration

*   **Scheme:** Internet-facing Application Load Balancer.
*   **Listener:** HTTP on Port 80.
*   **Target Group:** Protocol HTTP/1.1 on Port 80.
*   **Health Checks:** Configured for / with a 15-second interval, 5-second timeout, and 2 healthy/unhealthy thresholds. 

[Application LB](screenshots/ALB-screenshot.png)

## 8. Auto Scaling Configuration

*   **Capacity:** Minimum 2, Desired 2, Maximum 4.
*   **Health Check Type:** ELB
*   **Scaling Policy:** Target Tracking on Average CPU Utilization at **50%**.
*   **Instance Warm-up:** 120 seconds (allows newly launched instances time to pull files from S3 and pass health checks before contributing to ASG metrics).

## 9. Testing & Validation

### 9.1 Application & Load Balancing
*   **Test:** Accessed the ALB DNS name and refreshed multiple times.
*   **Result:** The site loaded successfully, and the instance ID badge alternated between the two instances across different AZs, proving traffic distribution.

[app server 1](screenshots/app-server-1.png)

[app server 2](screenshots/app-server-2.png)

### 9.2 Scale-Out
*   **Test:** Used the _stress_ utility on the existing instances to drive CPU above 50%.
*   **Result:** CloudWatch triggered the Target Tracking policy. Within **3 to 5 minutes**, the ASG launched new instances, and they successfully joined the Target Group as Healthy.

[Metrics before stress](screenshots/cloud-metrics-before -stress.png)

[Instances before stress](screenshots/instances-before-stress.png)

[Metrics during stress](screenshots/cloud-metrics-during-stress-showing-increased-CPU-utilization.png)

[Instances during stress](screenshots/instances-during-stress.png)

### 9.3 Scale-In
*   **Test:** Terminated the stress process.
*   **Result:** After a cooldown/stabilization period of **7 to 10 minutes**, the CPU metrics normalized, and the ASG successfully terminated the extra instances, returning capacity to the desired state of 2.

[Metrics after stress](screenshots/cloud-metrics-after-stress.png)

[Instances after stress](screenshots/instances-after-stress.png)

## 10. Monitoring

A CloudWatch Dashboard was created to monitor the infrastructure. It includes:
*   CPUUtilization (Per-Instance Metrics) to visualize the stress testing and ASG triggers.
*   HealthyHostCount and UnhealthyHostCount from the ALB.


## 11. Challenges Encountered & Solutions

*   **Challenge 1: IAM S3 Access Denied.** During initial manual testing, the _aws s3 sync_ command failed with an _AccessDenied_ error.
    *   *Solution:* The IAM policy required both _s3:ListBucket_ on the bucket ARN ( _arn:aws:s3:::bucket-name_ ) and _s3:GetObject_ on the object ARN ( _arn:aws:s3:::bucket-name/*_ ). The object ARN was missing from the policy so I had to cross check and add it.

*   **Challenge 2: Target Group Not Appearing in ALB Creation.** When creating the ALB, the target group created earlier was missing from the dropdown menu.
    *   *Solution:* The target group was configured with HTTP/2, but the ALB listener defaulted to HTTP/1.1. AWS filters out incompatible target groups so I could not find my target group from the dropdown menu. I recreated the target group using HTTP/1.1 to match the backend Apache server configuration.

## 13. Lessons Learned

Building this architecture from scratch highlighted the absolute necessity of understanding how interconnected AWS services are. Reading about Auto Scaling and Load Balancing is one thing, but building everything from scratch, hitting hurdles, stepping away, coming back to it and watching everything work as supposed really helps one understand it better.

