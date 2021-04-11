# AWS Cloud Solution For 2 Company Websites Using A Reverse Proxy Technology

## Intro
The following project demonstrates the setup of a secure infrastructure on the cloud. The fictitious company solution uses the wordpress content management system for its main business and a tooling website for the DevOps team.

In order for the solution to be highly available and scalable, cost effective, and secure, the services of a public cloud provider (AWS) was utilized. 

![AWS-Tooling-Website](https://user-images.githubusercontent.com/54307445/113601638-809ce400-9639-11eb-8763-28cfc0bdb3fc.png)

## Project prerequisites
1. An AWS account (with a new sub-account)
2. A domain name for the fictitious company
##### N.B: 
The above infrastructure is not categorised under the free tier.
Considering the number of resources to be created, it is quite pertinent to carry out proper tagging.

Below are excerpts from the project implementation:

## Virtual Private Network setup
- Based on the architectural diagram above, a VPC with subnets and a CIDR block of 10.0.0.0/16 was created.
![project-15-vpc-created](https://user-images.githubusercontent.com/54307445/113603962-85af6280-963c-11eb-9393-45df9f98e114.png)

![project-15-subnets-created](https://user-images.githubusercontent.com/54307445/113604050-a677b800-963c-11eb-9ae9-28634a45a5bc.png)

- The public and private subnets were associated with public and private route tables respectively and an internet Gateway was created.
- An edit of the public route table was made in order to associate the internet gateway with it (this is required in order for the public subnet to publicly accessible)
![project-15-route_table_plus_igw](https://user-images.githubusercontent.com/54307445/113604402-21d96980-963d-11eb-9812-259852a51218.png)

- A Nat Gateway and an elastic IP was created. For higher availability, it is advised to create one in the public subnet of each availability zone and associate them with the private subnets route table.

![project-15-nat](https://user-images.githubusercontent.com/54307445/113604643-77ae1180-963d-11eb-9adb-d05ffb191c34.png)

- Next, create security groups for the resources;
1. Nginx Servers:
![project-15-nginx-inbound](https://user-images.githubusercontent.com/54307445/113605000-f73be080-963d-11eb-95d5-52678458e46d.png)
2. Bastion Servers:
![project-15-bastion-sc](https://user-images.githubusercontent.com/54307445/113605197-39652200-963e-11eb-86c0-cdce75e25357.png)
3. Application Load Balancers:
![project-15-alb](https://user-images.githubusercontent.com/54307445/113605307-5f8ac200-963e-11eb-8c53-fa9bc7fe3de9.png)
4. Webservers:
![project-15-webservers](https://user-images.githubusercontent.com/54307445/113605449-952fab00-963e-11eb-80ee-230860ad4fd4.png)
5. Data Layer:
![project-15-datalayer-sc](https://user-images.githubusercontent.com/54307445/113605575-c27c5900-963e-11eb-9e73-a01c3886bb31.png)



## Compute Resources Setup
### Nginx Servers
- An EC2 instance (RHEL) was created and the necessary softwares (python, ntp, net-tools, vim, wget, telnet, epel-release, htop) were installed;
- An AWS AMI was made out of the instance
![project-15-ami](https://user-images.githubusercontent.com/54307445/113606162-88f81d80-963f-11eb-9ed5-4e0edeebc2fc.png)

- prepare launch template for the Nginx servers by;
1. making use of the AMI created
2. Assigning the appropriate security group
3. providing the script to update the package repository and install Nginx in the userdata section.
4. Tagging the template and its resources (resource tags)

- Configure Nginx target groups by;
1. Selecting Instances as the target type
2. Ensuring that the health check path is set (/healthstatus was used)
3. Registering the Nginx Instances as targets (The Nginx instances will act as reverse proxy on port 80 for the two websites).
4. Ensure that health check passes for the target group
5. Make sure to have the reverse proxy configuration on the nginx targets for the two websites.

- Configure autoscaling for the Nginx target group by;
1. Making the neccessary selections (such as appropriate launch template, VPC public subnets)
2. Enabling an Application Load Balancer for the Auto-scaling group
3. Selecting the target group initially created and ensuring health checks pass for both EC2 and ELB
4. configuring the auto-scaling bounds (i.e; minimum, maximum and desired capacity)
5. specify the condition to scale the group (eg; based on 90% cpu utilization)
6. select an SNS topic to send notifications to

## Bastion Servers
- An EC2 instance (RHEL) was created per availability zone and the necessary softwares (python, ntp, net-tools, vim, wget, telnet, epel-release, htop) were installed;
- The instances were associated with elastic IPs
- An AWS AMI was made out of the instance

- prepare launch template for the bastion servers by;
1. making use of the AMI created
2. Assigning the appropriate security group
3. providing the script to update the package repository and install Ansible and Git in the userdata section.
4. Tagging the template and its resources (resource tags).

- configure a target group for bastion servers. An auto-scaling group should also be configured

## Web Servers
- Here, two separate launch templates were created for both the Wordpress and Tooling websites
1. The EC2 instances (RHEL) were created and the necessary softwares (python, ntp, net-tools, vim, wget, telnet, epel-release, htop) were installed;
2. AMIs were made out of the instances

- Launch templates were created for the webservers
1. making use of the AMIs created
2. Assigning the appropriate security groups
3. providing the script to update the package repository and install wordpress (for the wordpress server only) and Git (to clone the ) in the userdata section.
4. Tagging the template and its resources (resource tags).

## set up TLS Certificates from amazon certifacte managar (ACM)
- In order to handle secure connectivity to the load balancers, we will need to setup TLS certificate.
1. Navigate to AWS ACM
2. Request a public wildcard certificate for the domain name you purchased from Freenom
3. Use DNS to validate the domain name
4. Tag the resource
![project-15-review-aws-cm](https://user-images.githubusercontent.com/54307445/114293202-2615e480-9a8c-11eb-82ac-a7251a33847c.png)

## Application Load Balancer To Route Traffic To NGINX
The Nginx Instances will have configurations that accept incoming traffic only from the load balancer. With this setup, we will benefit from intelligent routing of requests from the ALB to Nginx servers across the 2 availability zones. We will also be able to offload SSL/TLS termination from Nginx to the ALB. Therefore Nginx will be able to perform faster since it will not require extra compute resources to check for certificates for every request. We may achieve this by;
1. creating an Internet facing ALB (This will be the frontend load balancer)
2. Ensuring that it listens on HTTPS protocol
3. Ensuring the ALB is created within the appropriate VPC | AZ | Subnets
4. Choosing the Certificate from ACM
5. Selecting the needed Security Group
6. Selecting the Nginx Instances as the target group
7. Also, setting a listener to redirect http to https
![project-15-frontend-lb](https://user-images.githubusercontent.com/54307445/114293277-b18f7580-9a8c-11eb-9953-87d68fa9f610.png)


## Application Load Balancer To Route Traffic To the Web Servers
Since the web servers are configured for auto-scaling, there is going to be a problem if a server gets dynamically scaled up or down. Nginx will not know about the new IP address, or the one that gets removed. Hence, Nginx will not know where to direct the traffic.
To solve this problem, we must use a load balancer. But this time, it will be an internal load balancer. Not Internet facing since the web servers are within a private subnet, and we do not want direct access to them.
- Setup a target group for the wordpress instances and tooling instances respectively
- Setup an internal load balancer for the wordpress targets and tooling instances respectively. You won’t be allowed to use one load balancer with the same listener port (port 80) to route to multiple target groups; hence, two internal load balancers.

N:B:
you can also use one load balancer by specifying incoming request header rules

- Ensure that the reverse proxy configuration for both sites exist on the nginx targets. Depending on your decision of the number of internal loadbalancers to use edit the reverse proxy configuration
PROXYPASS directive appropriately.

## Setup EFS
Amazon Elastic File System (Amazon EFS) provides a simple, scalable, fully managed elastic NFS file system for use with AWS Cloud services and on-premises resources.
- We will target the EFS service and mount filesystems on both Nginx and web servers to store data.
1. Create an EFS file system
2. Create an EFS mount target per AZ in the VPC, and associate with both subnets dedicated for the data layer
3. Install the amazon-efs-utils package
4. Associate the Security groups created earlier for the data layer.
5. Ensure the security group assigned to efs network AZ (data layer security group) satisfies nfs requirements.
6. Create an EFS access point. (Give it a name and leave all other settings at default)
![project-15-efs](https://user-images.githubusercontent.com/54307445/114293444-ed770a80-9a8d-11eb-9e8f-8932b83c7ee3.png)

## Setup RDS
Prerequisite: we will need to create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance.

To ensure that the databases are highly available and also have failover support in case one availability zone becomes unavailable, we will configure a multi-AZ MySQL database instance. In our case, since we are only using 2 AZs, we can only failover to one. If there was need for even more availability.
![project-15-db](https://user-images.githubusercontent.com/54307445/114293484-4e064780-9a8e-11eb-807d-3cec02e728b8.png)

- next login to the database and create the necessary admin user, databases (for the wordpress and tooling sites)
- grant necessary privileges
- import any existing tooling website database tables
- Next update the userdata scripts for wordpress and tooling launch templates to reflect the setup of efs, and mysql connections 

## Configuring DNS with Route53
Part of the early tasks in this project was to get a free domain from Freenom, and configure hosted zones in Route53. But that is not all that needs to be done as far as DNS configuration is concerned.
- Ensure that the main domain for the wordpress can be reached, and the subdomain for tooling website can also be reached from the browser.
- Create other records such as CNAME, alias and A records.
- Create a cname (or alias) record for the root domain, and direct its traffic to the ALB DNS name.
- Create a cname (or alias) record for tooling.<yourdomain>.com and direct its traffic to the ALB DNS name.
![project-15-route53](https://user-images.githubusercontent.com/54307445/114293564-10ee8500-9a8f-11eb-9e25-7f9c02e2dbf8.png)

## Verify the setup
![project-15-final](https://user-images.githubusercontent.com/54307445/114303775-aa398d80-9ac7-11eb-9fb2-f12974732430.gif)




