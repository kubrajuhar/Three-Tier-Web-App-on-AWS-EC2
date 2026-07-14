# Three-Tier-Web-App-on-AWS-EC2

THREE-TIER WEB APP ON AWS EC2
High-Level Design, Deployment & Testing Documentation
AWS Region: eu-north-1 (Stockholm)
Services: EC2, Application Load Balancer, Auto Scaling, RDS (MySQL) and VPC

<img width="1160" height="674" alt="image" src="https://github.com/user-attachments/assets/dba4951f-8f37-41d0-a6ef-2ab29ad10d7c" />

Figure 1 — High-Level Design (HLD): Three-tier VPC architecture

1. Project Overview\
   
This project implements a classic three-tier web application architecture on AWS. The goal was to deploy a highly available, secure web application with clearly separated presentation, application, and data layers, each isolated at the network level.\

1.1 Goal
Deploy a classic web / app / DB tier architecture with high availability, using Amazon EC2, an Application Load Balancer (ALB), Auto Scaling, Amazon RDS, and a VPC with public/private subnet separation.\

1.2 Tech Stack
•Networking: Amazon VPC with public and private subnets across two Availability Zones
•Presentation tier: Application Load Balancer (ALB)
•Application tier: Amazon EC2 instances managed by an Auto Scaling Group (ASG)
•Data tier: Amazon RDS (MySQL), Multi-AZ subnet group
•Region: eu-north-1 (Stockholm), Availability Zones eu-north-1a and eu-north-1b
1.3 Security Model
Security group chain: alb-sg (0.0.0.0/0 → 80/443) → app-sg (only from alb-sg) → db-sg (only from app-sg). Each tier only accepts traffic from the tier directly above it  never a raw IP range for internal traffic so EC2 instances can be replaced by Auto Scaling without breaking any rule.
3. Architecture
Traffic flows from the internet through the ALB in the public subnets, which distributes requests across EC2 instances in the private application subnets. The application tier connects to the RDS database in the private data subnets. Only the ALB is reachable from the public internet the application and data tiers have no public IP addresses.
2.1 Traffic Flow Summary
•Internet users → ALB (public subnets, ports 80/443)
•ALB → EC2 Auto Scaling Group (private app subnets, port 80)
•EC2 app tier → RDS MySQL (private data subnets, port 3306)
<img width="1160" height="668" alt="image" src="https://github.com/user-attachments/assets/7e726605-f832-46ff-b4b0-b44344a4fa63" />

Figure 2 — VPC resource map (subnets, route tables, gateways)
3. Networking
VPC: three-tier-vpc — CIDR block 10.0.0.0/16, spanning two Availability Zones (eu-north-1a, eu-north-1b). Public subnets host the ALB; private subnets host the EC2 application instances and RDS database.
3.1 Subnets
Subnet Name	CIDR Block	Availability Zone	Type
public-subnet-1	10.0.1.0/24	eu-north-1a	Public
public-subnet-2	10.0.2.0/24	eu-north-1b	Public
app-subnet-1	10.0.11.0/24	eu-north-1a	Private
app-subnet-2	10.0.12.0/24	eu-north-1b	Private
db-subnet-1	10.0.21.0/24	eu-north-1a	Private
db-subnet-2	10.0.22.0/24	eu-north-1b	Private
<img width="1160" height="636" alt="image" src="https://github.com/user-attachments/assets/d3aea84b-e8c3-4fb1-b588-14def3473866" />

Figure 3 — Subnets as provisioned in the AWS Console
3.2 Routing
Route Table	Destination	Target	Associated Subnets
public-route-table	10.0.0.0/16	local	public-subnet-1, public-subnet-2
public-route-table	0.0.0.0/0	Internet Gateway (three-tier-igw)	public-subnet-1, public-subnet-2
private-route-table	10.0.0.0/16	local	app + db subnets (all four)
private-route-table	0.0.0.0/0	NAT Gateway (three-tier-nat)	app + db subnets (all four)
A single zonal NAT Gateway (three-tier-nat) sits in public-subnet-1 and allows outbound internet access from the private subnets (e.g. for OS package updates) without exposing them to inbound traffic.
<img width="1160" height="617" alt="image" src="https://github.com/user-attachments/assets/24dd845a-8508-409b-8b4d-b5be661d11a3" />

Figure 4 — Route tables
<img width="1160" height="596" alt="image" src="https://github.com/user-attachments/assets/28a17fbe-f733-4700-b45e-5ddc98a0ebe1" />

Figure 5 — Internet Gateway (three-tier-igw)
4. Security Groups
Each tier has its own security group, and each only accepts traffic from the tier directly above it.
Security Group	Inbound Rule	Source	Purpose
alb-sg	HTTP 80 / HTTPS 443	0.0.0.0/0	Public entry point
app-sg	HTTP 80	alb-sg	Only ALB can reach app tier
app-sg	SSH 22 (optional)	My IP	Troubleshooting access
db-sg	MySQL/Aurora 3306	app-sg	Only app tier can reach database
<img width="1160" height="666" alt="image" src="https://github.com/user-attachments/assets/45ebc38d-995c-4ec8-bd94-af2d352413af" />

Figure 6 — NAT Gateway (three-tier-nat)
5. Deployment Steps Summary
5.1 Networking Foundation
•Created VPC three-tier-vpc (10.0.0.0/16)
•Created 6 subnets across 2 AZs (public, app, db tiers)
•Created and attached Internet Gateway three-tier-igw
•Created zonal NAT Gateway three-tier-nat in public-subnet-1 with an Elastic IP
•Created public and private route tables and associated the correct subnets
**5.2 Security Groups**
•Created alb-sg, app-sg, and db-sg, each scoped to allow traffic only from the tier above
**5.3 Database Tier**
•Created DB subnet group three-tier-db-subnet-group across db-subnet-1 and db-subnet-2
•Created RDS MySQL instance three-tier-db (db.t3.micro, 20 GB gp3, Free Tier eligible)
•Public access disabled; attached security group db-sg only
•Initial database name: appdb
**5.4 Application Tier**
•Created EC2 key pair, three-tier-key
•Created Launch Template, three-tier-app-template (Amazon Linux 2023, t2/t3.micro, app-sg, user-data installs Apache httpd)
•Created Auto Scaling Group three-tier-asg across app-subnet-1 and app-subnet-2 (min 2, max 4, target tracking on CPU 50%)
<img width="1160" height="544" alt="image" src="https://github.com/user-attachments/assets/ab3c1a4f-cf3a-4cbe-838a-9cab3ca74ce8" />
Figure 7 — Database configuration (three-tier-asg)
**5.5 Load Balancer**
•Created an internet-facing Application Load Balancer (three-tier-alb) in the public subnets
•Created a target group app-tier-tg (HTTP:80, target type: Instance) pointing to the Auto Scaling Group
•Attached a listener on port 80 to route traffic to the target group
DNS: three-tier-alb-16239136.eu-north-1.elb.amazonaws.com
<img width="1160" height="598" alt="image" src="https://github.com/user-attachments/assets/ae8e986b-8cb6-436b-ada5-4ab98d0e10d8" />
<img width="1204" height="635" alt="image" src="https://github.com/user-attachments/assets/591e9eec-a415-44be-a0d8-587d27320b82" />

Figure 8 — Application Load Balancer successfully distributing traffic to a backend EC2 instance across Availability Zones
**5.6 Validation & Testing**
•Confirmed the ALB DNS name serves the web page from the app tier
•Verified high availability by testing instance replacement / failover across Availability Zones
•Confirmed EC2 instances have no public IP and RDS is not publicly accessible
6. Testing & Validation
**6.1 Results Summary**
Test	Expected Result	Outcome
Hit ALB DNS name in browser	Web page served from an app-tier EC2 instance	Passed
High availability / failover	Traffic continues to be served if an instance is replaced	Passed
EC2 public IP check	No public IP assigned to app-tier instances	Passed
RDS public accessibility	Database not reachable from the internet	Passed
6.2 Load Balancing Verification
Refreshed the ALB's DNS endpoint multiple times and confirmed responses alternated between both app-tier instances (identified by their internal hostnames, e.g. ip-10-0-11-170.eu-north-1.compute.internal), confirming round-robin distribution across healthy targets.
6.3 Failover / Self-Healing Test
•Deliberately terminated one of the two running app-tier EC2 instances to simulate a failure.
•Observed: The Auto Scaling Group automatically detected the capacity drop and launched a replacement instance within the same AZ, restoring the desired count of 2.
•Observed: The ALB continued serving traffic without interruption throughout, using the surviving instance.
6.4 Issue Discovered: Target Group Not Linked to ASG
During the failover test, the newly launched replacement instance did not automatically appear in the target group only the original manually-registered instance was present, dropping healthy target count to 1 instead of 2.
Root cause: The target group's initial instance registration (during setup) was a one-time manual registration, not a persistent link between the Auto Scaling Group and the target group. The ASG had no knowledge of the target group and therefore couldn't register new instances into it.
Fix: Edited the Auto Scaling Group (three-tier-asg → Edit → Load balancing) and attached the existing target group app-tier-tg under "Application, Network or Gateway Load Balancer target groups" (rather than accidentally creating a new load balancer, which the console defaults to if this option isn't explicitly selected first).
Result after fix: Target group showed 2/2 healthy targets, and subsequent ALB refreshes correctly alternated between both instances again. This link persists going forward  any future instance replacements by the ASG will automatically register with the target group without manual steps.
6.5 Connectivity Troubleshooting Note
Early in ALB testing, the DNS endpoint returned ERR_CONNECTION_REFUSED despite all configuration (security groups, listener, target health) appearing correct. This resolved itself after allowing a few minutes for ALB provisioning/DNS propagation to complete  a useful reminder that a newly "Active" ALB isn't always immediately reachable.
**7. Key Learnings**
•ASG + ALB integration requires an explicit, persistent link manually registering instances with a target group during setup does not extend to instances the ASG creates later. The target group must be attached at the ASG level for self-healing to work end-to-end.
•Stopping vs. terminating an ASG-managed instance behaves differently than expected an ASG treats a "stopped" instance as unhealthy and will terminate + replace it rather than leaving it stopped, since ASGs are designed to maintain a target count of running instances.
•DNS propagation delay on a freshly created ALB can cause transient connection failures even when configuration is correct worth waiting a minute before troubleshooting further.
•Security group chaining (ALB → App → DB, each restricted to the previous tier) is what actually enforces the "three-tier isolation" goal public internet can only ever reach the ALB, never the app or DB instances directly.
8. Conclusion
The three-tier architecture was successfully deployed and validated end-to-end in eu-north-1: the ALB DNS correctly serves traffic from the app tier, load is balanced across both Availability Zones, the Auto Scaling Group self-heals after instance failure once correctly linked to the target group, and both the application and database tiers remain fully isolated from direct internet access. The deployment meets the project goal of a highly available, network-segmented web/app/DB architecture on AWS.
