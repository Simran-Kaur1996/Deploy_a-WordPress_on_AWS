# **Deployed and Hosted a Highly Available WordPress Application using EC2, RDS, Route53, ASG, and VPC**

This repository contains the resources and scripts used to deploy a WordPress website on Amazon Web Services (AWS). The project leverages various AWS services to ensure high availability, scalability, and security for the WordPress application.

## Architecture Overview:
![WordPress on AWS](2._Host_a_WordPress_Website_on_AWS_Architecture.png)

## Problem Statement

The purpose of the project was to deploy a secure, scalable, and resilient WordPress website utilizing the AWS infrastructure. The website must adjust its resource levels based on traffic, be distributed across at least two Availability Zones (AZs) for resilience, and include appropriate security measures such as encryption. The application should also leverage AWS-managed services to reduce the workload associated with infrastructure management.

## Architecture Overview

The WordPress website is hosted on EC2 instances within a highly available and secure architecture that includes:

- **Virtual Private Cloud (VPC):** Configured with both public and private subnets across two Availability Zones (AZs) for fault tolerance and high availability.
- **Internet Gateway:** Allows communication between instances in the VPC and the internet.
- **Security Groups:** Acts as a virtual firewall to control inbound and outbound traffic.
- **Public Subnets:** Hosts infrastructure components like the NAT Gateway and Application Load Balancer (ALB) for external access and load balancing.
- **Private Subnets:** Hosts the web servers to enhance security by limiting direct exposure to the internet.
- **EC2 Instance Connect Endpoint:** Provides secure SSH access to the EC2 instances in both public and private subnets.
- **Application Load Balancer (ALB):** Distributes incoming web traffic across multiple EC2 instances for better performance and fault tolerance.
- **Auto Scaling Group (ASG):** Ensures that the correct number of EC2 instances are running based on traffic, providing elasticity and availability.
- **Amazon RDS:** Managed relational database service for storing WordPress data.
- **Amazon EFS:** Scalable file storage system for storing WordPress files across multiple instances.
- **AWS Certificate Manager:** Provides SSL/TLS certificates to secure communications between users and web servers.
- **AWS Simple Notification Service (SNS):** Sends notifications related to activities within the Auto Scaling Group.
- **Amazon Route 53:** Manages domain name registration and DNS routing.

🧩 **Problems Solved**

- **High Availability:** Guarantees that the failure of one Availability Zone will not affect the website.
- **Scalability:** Automatically adjusts resources based on traffic levels.
- **Enhanced Security:** Private subnets and security groups ensure data and server security.
- **Simplified Management:** AWS RDS and EFS simplify administrative efforts.

🛑 **Issues Faced and Resolutions**

- **EC2 instances not connecting to the EFS:** 
   - *Cause:* Incorrect security group rules for NFS network traffic.
   - *Resolution:* Allowed NFS (Port 2049) traffic in EFS security groups.

- **WordPress unable to connect to the database:** 
   - *Cause:* Incorrect password for the XFS database.
   - *Resolution:* Updated the `wp-config.php` with correct credentials and verified the RDS security group rule.

## Deployment Scripts

### WordPress Installation Script

This script is used for the initial setup of the WordPress application on an EC2 instance. It includes steps for installing Apache, PHP, MySQL, and mounting Amazon EFS to the instance.
```bash
# create to root user
sudo su

# update the software packages on the ec2 instance 
sudo yum update -y

# create an html directory 
sudo mkdir -p /var/www/html

# environment variable
EFS_DNS_NAME=fs-064e9505819af10a4.efs.us-east-1.amazonaws.com

# mount the efs to the html directory 
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# install the apache web server, enable it to start on boot, and then start the server immediately
sudo yum install -y httpd
sudo systemctl enable httpd 
sudo systemctl start httpd

# install php 8 along with several necessary extensions for wordpress to run
sudo dnf install -y \
php \
php-cli \
php-cgi \
php-curl \
php-mbstring \
php-gd \
php-mysqlnd \
php-gettext \
php-json \
php-xml \
php-fpm \
php-intl \
php-zip \
php-bcmath \
php-ctype \
php-fileinfo \
php-openssl \
php-pdo \
php-tokenizer

# install the mysql version 8 community repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 
#
# install the mysql server
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm 
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 
#
# start and enable the mysql server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
chown apache:apache -R /var/www/html 

# download wordpress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# create the wp-config.php file
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# edit the wp-config.php file
sudo vi /var/www/html/wp-config.php

# restart the webserver
sudo service httpd restart
```


### **Auto Scaling Group Launch Template Script**
This script is included in the launch template for the Auto Scaling Group, ensuring that new instances are configured correctly with the necessary software and settings.
```bash
# update the software packages on the ec2 instance 
sudo yum update -y

# install the apache web server, enable it to start on boot, and then start the server immediately
sudo yum install -y httpd
sudo systemctl enable httpd 
sudo systemctl start httpd

# install php 8 along with several necessary extensions for wordpress to run
sudo dnf install -y \
php \
php-cli \
php-cgi \
php-curl \
php-mbstring \
php-gd \
php-mysqlnd \
php-gettext \
php-json \
php-xml \
php-fpm \
php-intl \
php-zip \
php-bcmath \
php-ctype \
php-fileinfo \
php-openssl \
php-pdo \
php-tokenizer

# install the mysql version 8 community repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 
#
# install the mysql server
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm 
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 
#
# start and enable the mysql server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# environment variable
EFS_DNS_NAME=fs-02d3268559aa2a318.efs.us-east-1.amazonaws.com

# mount the efs to the html directory 
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# set permissions
chown apache:apache -R /var/www/html

# restart the webserver
sudo service httpd restart
```
How to Use
Clone this repository to your local machine:

git clonehttps://github.com/Simran-Kaur1996/Deploy_a-WordPress_on_AWS.git
cd wordpress-on-aws
Follow the AWS documentation to create the required resources (VPC, subnets, Internet Gateway, etc.) as outlined in the architecture overview.

Use the provided scripts to set up the WordPress application on EC2 instances within the VPC.

Configure the Auto Scaling Group, Load Balancer, and other services as per the architecture.

Access the WordPress website through the Load Balancer's DNS name.
## **Deployed Website**

![WordPress on AWS](screenshot_of_website_one.png)
