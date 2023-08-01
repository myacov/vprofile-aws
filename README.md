# vprofile-aws

## Objective: 
Host and run a web application (VPROFILE) on AWS using the Lift & Shift strategy. Move a physical/virtual stack to the cloud with automation using Infrastructure as Code (IaC). Utilize autoscaling for improved performance and manage costs effectively with a pay-as-you-go model.

## Architecture:
![Project diagram](./images/proj3.jpg)

## AWS Services
| Service | USE | 
| ------------- | ------------- | 
| EC2 Instances  | Replacing VNs for tomcat, RabbitMQ, Memcached, MySQL |
| ELB (Elastic Load Balancer)  | Replacing Nginx Load Balancer |
| autoscaling | automation fo VM scaling | 
| S3 + EFS | Shared Storage | 
| Route 53 | Private DNS Service  | 
| IAM | Identity & Access Manager  |
| ACM | Amazon Certificate Manager |
| EBS | Elastic Block Storage |

## Desired Learning outcomes
- Gain familiarity with various AWS services.
- Understand and implement autoscaling for optimal performance.

## Flow of Execution
1. **Create 3 Security Groups** for LoadBalancer, App, and Backend Services.
2. **Create Key Pairs** for EC2 Instances.
3. **Launch Instances** with user data (Bash Scripts).
4. **Update IP to name mapping** in Route 53.
5. **Build Application** from source code (locally).
6. **Upload artifact** to S3 Bucket.
7. D**ownload artifact** from bucket to Tomcat EC2 instance.
8. **Set up ELB** with HTTPS connection (using ACM).
9. **Map ELB Endpoint** to our website in GoDaddy DNS.
10. **Verify** the application.
11. Build **Autoscaling** Group for Tomcat Instances.

## Prerequisites:
- AWS Account
- jdk 11 and Maven 

## Create Security Groups
1. ELB Security group (**vprofile-elb-SG**):
    Description: Security group for vprofile prod Load Balancer
    - Inbound rules:
        - HTTP (Port 80) from anywhere (temporary rule)
        - HTTPS (Port 443) from anywhere
2. App Security group (**vprofile-app-SG**):
    Description: Security group for tomcat instances 
    - Inbound rules:
        - Custom (Port 8080) from **vprofile-elb-SG**
                Description: Allow traffic from vprofile prod ELB
        - SSH (Port 22) from **MY IP**
                Description: Allow SSH
        - Custom (Port 8080) from **MY IP**
                Description: Allow access from browser (troubleshooting)
3. Backend Security group (**vprofile-backend-SG**):
    Description: Security group for backend services 
      - Inbound rules:
        - MYSQL (Port 3306) from **vprofile-app-SG**
            Description: Allow 3306 from application servers
        - Custom TCP (Port 11211) from **vprofile-app-SG**
            Description: Allow Tomcat to connect to Memcache
        - Custom TCP (Port 5672) from **vprofile-app-SG**
            Description: Allow Tomcat to connect to RabbitMQ
        - All Traffic from **vprofile-backend-SG**
            Description: Allow internal traffic to flow on all ports
        - SSH (Port 22) from **MY IP**
            Description: Allow SSH

## Create Key Pair for EC2 Instances
1. **vprofile-prod-key**
    - type: RSA
    - format: .pem

## Launch Instances with user data
1. Database Instance
    - Name: **`vprofile-db01`**
    - Project: `vprofile`
    - AMI: `CentOS Stream 9 (x86_64)`
    - type: `t2.micro`
    - Key pair: `vprofile-prod-key`
    - Network settings: Security group: **vprofile-backend-SG**
    - Advanced details: User data : Paste contents of mysql.sh
2. Memcache Instance
    - Name: **`vprofile-mc01`**
    - Project: `vprofile`
    - AMI: `CentOS Stream 9 (x86_64)`
    - type: `t2.micro`
    - Key pair: `vprofile-prod-key`
    - Network settings: Security group: **vprofile-backend-SG**
    - Advanced details: User data : Paste contents of memcache.sh
3. RabbitMQ Instance
    - Name: **`vprofile-rmq01`**
    - Project: *`vprofile`
    - AMI: `CentOS Stream 9 (x86_64)`
    - type: `t2.micro`
    - Key pair: `vprofile-prod-key`
    - Network settings: Security group: **vprofile-backend-SG**
    - Advanced details: User data : Paste contents of rabbitmq.sh
4. Tomcat Instance
    - Name: **`vprofile-app01`**
    - Project: `vprofile`
    - AMI: `Ubuntu Server 22.04 LTS`
    - type: `t2.micro`
    - Key pair: `vprofile-prod-key`
    - Network settings: Security group: **vprofile-app-SG**
    - Advanced details: User data : Paste contents of tomcat_ubuntu.sh

## Update IP to name mapping in Route 53
1. Create a private hosted zone in Route 53:
- Domain name: `vprofile.in`
- Type: Private hosted zone (within an Amazon VPC)
- Region : `us-east-1`
- VPC: `Default vpc`

2. Create DNS records for the instances in the hosted zone:
Simple routing (simple record):
- Record name: db01
    Value/Route traffic to **private ip of `db01`**
- Record name: mc01
    Value/Route traffic to **private ip of `mc01`**
- Record name: rmq01
    Value/Route traffic to **private ip of `rmq01`**

## Build and Upload artifact to S3 Bucket
###  Build Application from source code (locally) 
```bash
mvn install
```
### for authentication we must first create IAM user
1. Create an IAM user: `s3admin`
    - Attach policies directly: AmazonS3FullAccess
2. Create access key for user (Command Line Interface (CLI))
3. Assign access key to Tomcat Instance (`aws configure`)
4. Create a role (IAM Roles)
    - Trusted entity type: `AWS service`
    - Use case: `EC2`
    - Permissions policies: `AmazonS3FullAccess`
    - Role name: `vprofile-artifact-store`

    EC2 > Instances > app01 > Modify IAM role >select role 
- Role: `vprofile-artifact-store`

### Create S3 Bucket and upload artifact
```bash
aws s3 mb s3://vproarts3
aws s3 cp target/vprofile-v2.war s3://vproarts3
```

### Download artifcat from bucket to Tomcat EC2 instance
1. Install awscli and fetch the artifact on the Tomcat instance (app01):
```bash
sudo -i
apt update
apt install awscli
aws s3 ls
aws s3 cp s3://vproarts3/vprofile-v2.war /tmp/
```
2. Stop tomcat Service ande replace default artifact
```bash
systemctl stop tomcat9
rm -rf /var/lib/tomcat9/webapps/ROOT/
cp /tmp/vprofile-v2.war /var/lib/tomcat9/webapps/ROOT.war
systemctl start tomcat9
```
checks:

```bash
ls /var/lib/tomcat9/webapps/
cat /var/lib/tomcat9/webapps/ROOT/WEB-INF/classes/application.properties
```
## Setup Elastic Load Balancer (ELB) with HTTPS Connection (using ACM)
### Creating Target groups
Target Groups: Create Target group
- target type: `Instances`
- Target group name: **`vprofile-app-TG`**
- Protocol:`HTTP`   Port:`8080`
- Health check path: `/login`
- Advanced health check settings: Health check port 
  - Override on port `8080`
- Healthy threshold: `3`
- Available instances: **vprofile-app01**
        
### Creating Load Balancer
- Type: Application Load Balancer
- name: **`vprofile-prod-elb`**
- Scheme:  `Internet-facing`
- Network mapping:  Select all
- Security groups:  **`vprofile-elb-SG`**
- Listeners and routing: 
    - Add Listener: 
        - HTTPS 443 Forward to: `vprofile-app-TG`
        - Secure listener settings: add ACM certificate


## Map ELB Endpoint to our website in godaddy DNS
- copy "DNS name" to GoDaddy as `CNAME record`
    - name: `vprofileapp` Value: "DNS name"

## Verify
Access through browser on `https://vprofileapp.medinay.com`

## Autoscaling
Building Autoscaling Group for tomcat Instances:

1. Create Image for the instance

   **EC2 > Instances > vprofile-app01 > Create image (AMI)**
- Image name: **`vprofile-app-image`**

2. Create Auto Scaling group

    **EC2 > Launch configurations > Create launch configuration**
- Name: **`vprofile-app-LC`**
- Amazon machine image: `vprofile-app-image`
- Instance type: `t2.micro`
- Additional configuration: IAM instance profile: vprofile-artifact-store(ROLE)
    - [x] Enable EC2 instance detailed monitoring within CloudWatch
- Security groups: 
    - [x] existing:	`vprofile-app-SG`	
- Key pair: 
    - [x] existing: `vprofile-prod-key`

    **EC2 > Auto Scaling groups > Create Auto Scaling group**
- Name: vprofile-app-ASG
- Switch to launch configuration: vprofile-app-LC
- Availability Zones and subnets: Select all
- Load balancing: 
    - [x] Attach to an existing load balancer
    - [x] Choose from your load balancer target groups
    - select target group: 
        - [x] vprofile-app-TG | HTTP
EC2 health checks: 
- [x] Turn on Elastic Load Balancing health check
- Group size: Set min max and desired
- Scaling policies: 
    - [x] Target tracking scaling policy
- Add notifications: SNS Topic
- [x] Launch
- [x] Terminate
- [x] Fail to launch
- [x] Fail to terminate