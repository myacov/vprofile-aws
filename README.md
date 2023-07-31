# vprofile-aws
## Objective: 
Host and run a web application (VPROFILE) on AWS
Using Lift & Shift Strategy 
Moving a physical/virtual stack to the cloud
automation IaC
use autoscatling
managing cost - pay as you go
![Project diagram](./images/proj3.jpg)
## Desired Learning outcomes
familierizing with AWS services
autoscaling

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

## Flow of Execution
1. Create 3 Security Groups for:  LoadBalancer, App and Backend Services
2. Create Key Pairs for EC2 Instances
3. Launch Instances with user data (Bash Scripsts)
4. Update IP to name mapping in Route 53
5. Build Application from source code (locally)
6. Upload artifact to S3 Bucket
7. Download artifcat from bucket to Tomcat EC2 instance
8. Setup ELB with https connection (using ACM)
9. Map ELB Endpoint to our website in godaddy DNS
10. Verify
11. Build Autoscaling Group for tomcat Instances


## Prerequisites:
- AWS Account
- jdk 11 and Maven 
## Create Security Groups
1. ELB Security group
    Name: vprofile-elb-SG
    Description: Security group for vprofile prod Load Balancer
    Inbound rules:
    1. HTTP (Port 80) from anywhere (temp rule)
    2. HTTPS (Port 443) from anywhere
2. App Security group
    Name: vprofile-app-SG
    Description: Security group for tomcat instances 
    Inbound rules:
    1. Custom (Port 8080) from **profile-elb-SG**
            Description: Allow traffic from vprofile prod ELB
    2. SSH (Port 22) from **MY IP**
            Description: Allow SSH
    3. Custom (Port 8080) from **MY IP**
            Description: Allow access from browser (troubleshooting)
3. Backend Security group
    Name: vprofile-backend-SG
    Description: Security group for backend services 
    Inbound rules:
    1. MYSQL (Port 3306) from **vprofile-app-SG**
        Description: Allow 3306 from application servers
    2. Custom TCP (Port 11211) from **vprofile-app-SG**
        Description: Allow tomcat to connect to memcache
    3. Custom TCP (Port 5672) from **vprofile-app-SG**
        Description: Allow tomcat to connect to RabbitMQ
    4. All Traffic from **vprofile-backend-SG**
        Description: Allow internal traffic to flow on all ports
    5. SSH (Port 22) from **MY IP**
        Description: Allow SSH

## Create Key Pairs for EC2 Instances
1. vprofile-prod-key
    Key pair type: RSA
    Private key file format: .pem

## Launch Instances with user data
1. Database Instance
    Name: *vprofile-db01*
    Project: *vprofile*
    AMI: CentOS Stream 9 (x86_64)
    type: t2.micro
    Key pair: vprofile-prod-key
    Network settings: Security group: **vprofile-backend-SG**
    Advanced details: User data : Paste contents of mysql.sh
2. Memcache Instance
    Name: *vprofile-mc01*
    Project: *vprofile*
    AMI: CentOS Stream 9 (x86_64)
    type: t2.micro
    Key pair: vprofile-prod-key
    Network settings: Security group: **vprofile-backend-SG**
    Advanced details: User data : Paste contents of memcache.sh
3. RabbitMQ Instance
    Name: *vprofile-rmq01*
    Project: *vprofile*
    AMI: CentOS Stream 9 (x86_64)
    type: t2.micro
    Key pair: vprofile-prod-key
    Network settings: Security group: **vprofile-backend-SG**
    Advanced details: User data : Paste contents of rabbitmq.sh
4. RabbitMQ Instance
    Name: *vprofile-app01*
    Project: *vprofile*
    AMI: Ubuntu Server 22.04 LTS
    type: t2.micro
    Key pair: vprofile-prod-key
    Network settings: Security group: **vprofile-app-SG**
    Advanced details: User data : Paste contents of tomcat_ubuntu.sh

## Update IP to name mapping in Route 53
Route 53 > Hosted zones > Create hosted zone
    Domain name: vprofile.in
    Type: [x] Private hosted zone (within an Amazon VPC)
    Region : us-east-1
    VPC: Default vpc

vprofile.in > Create record
    [x] Simple routing
Define simple record:
1.  Record name: db01
    Value/Route traffic to **private ip of db01**
2. Record name: mc01
    Value/Route traffic to **private ip of mc01**
3. Record name: rmq01
    Value/Route traffic to **private ip of rmq01**

## Build Application from source code (locally)
```bash
mvn install
```
## Upload artifact to S3 Bucket
### for authentication we must first create IAM user
IAM dashboard > Users > Create user
    User name: s3admin
        [x] Attach policies directly: AmazonS3FullAccess	
IAM > Users > *s3admin* > Create access key > [x] Command Line Interface (CLI)
```bash
aws configure
```
enter access key and secret key

3. Create S3 bucket:
```bash
aws s3 mb s3://vproarts3
aws s3 cp target/vprofile-v2.war s3://vproarts3
```

## Download artifcat from bucket to Tomcat EC2 instance
1. we will install awscli and fetch the artifact
    we will create a role (IAM Roles)
    Trusted entity type: [x] AWS service
    Use case: [x] EC2
    Permissions policies: "AmazonS3FullAccess"
    Role name: vprofile-artifact-store
EC2 > Instances > app01 > Modify IAM role >select role 
    Role: vprofile-artifact-store 
2. ssh into app01
```bash
sudo -i
apt update
apt install awscli
aws s3 ls
aws s3 cp s3://vproarts3/vprofile-v2.war /tmp/
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
## Setup ELB with https connection (using ACM)
### Creating Load Balancer
EC2 > Target groups
    target type [x] Instances
    Target group name: **vprofile-app-TG**
    Protocol: HTTP ; Port : 8080
    Health check path: /login
    Advanced health check settings: Health check port [x] Override 8080
    Healthy threshold: 3

    Available instances: **vprofile-app01**
        > click on Include as pending below
        create Target group

EC2 > Load balancers > [x] Application Load Balancer
    Load balancer name: vprofile-prod-elb
    Scheme: [x] Internet-facing
    Network mapping: [x] Select all
    Security groups: [x] vprofile-elb-SG
Listeners and routing: Add Listener: HTTPS 443 Forward to: vprofile-app-TG
Secure listener settings: add ACM certificate
## Map ELB Endpoint to our website in godaddy DNS
    copy "DNS name" to godaddy as CNAME record
    name: vprofileapp Value: "DNS name"
## Verify
    Access through browser on https://vprofileapp.medinay.com
## Build Autoscaling Group for tomcat Instances
### Creating Image for instance
EC2 > Instances > vprofile-app01 > Create image (AMI)
    Image name: vprofile-app-image

### Create Auto Scaling group
EC2 > Launch configurations > Create launch configuration
Name: vprofile-app-LC
Amazon machine image: vprofile-app-image
Instance type: t2.micro
Additional configuration: IAM instance profile: vprofile-artifact-store(ROLE)
[x] Enable EC2 instance detailed monitoring within CloudWatch
Security groups: [x] existing:	vprofile-app-SG	
Key pair: [x] existing: vprofile-prod-key

EC2 > Auto Scaling groups > Create Auto Scaling group
Name: vprofile-app-ASG
Switch to launch configuration: vprofile-app-LC
Availability Zones and subnets: Select all
Load balancing : [x] Attach to an existing load balancer
[x] Choose from your load balancer target groups
select target group: [x] vprofile-app-TG | HTTP
EC2 health checks: [x] Turn on Elastic Load Balancing health check
Group size: min max and desired
Scaling policies: [x] Target tracking scaling policy
Add notifications: SNS Topic
[x] Launch
[x] Terminate
[x] Fail to launch
[x] Fail to terminate