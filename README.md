# Host-a-Wordpress-site
# Hosting a WordPress Website on AWS

This repository provides comprehensive resources and scripts to facilitate the deployment of a WordPress website on Amazon Web Services (AWS). By leveraging various AWS services, this solution ensures high availability, scalability, and security for the WordPress application.

## **Architecture Overview**

The WordPress website is hosted on AWS using an architecture designed for fault tolerance, scalability, and security. Below are the core components of the architecture:

### **AWS Services Utilized:**

- **Virtual Private Cloud (VPC):** A custom VPC is created to isolate network resources and ensure secure communication between AWS services. The VPC spans multiple Availability Zones (AZs) for redundancy.
- **Subnets:** Both public and private subnets are provisioned across two AZs to ensure high availability.
  - *Public Subnets:* Used for NAT Gateway and Application Load Balancer to provide external access.
  - *Private Subnets:* Used for hosting the web servers, enhancing security by limiting direct internet exposure.
- **Internet Gateway:** Facilitates internet access for instances in the public subnets.
- **Security Groups:** Acts as virtual firewalls to control inbound and outbound traffic for the instances.
- **Application Load Balancer (ALB):** Distributes incoming web traffic across multiple EC2 instances in different AZs, ensuring better performance and fault tolerance.
- **Auto Scaling Group (ASG):** Automatically adjusts the number of EC2 instances based on traffic demand, ensuring scalability and resilience.
- **Amazon RDS (Relational Database Service):** Provides a managed MySQL database instance with automated backups, patching, and scaling capabilities.
- **Amazon EFS (Elastic File System):** Offers scalable and shared file storage that can be mounted across multiple EC2 instances.
- **AWS Certificate Manager (ACM):** Manages SSL/TLS certificates for secure communication.
- **AWS Simple Notification Service (SNS):** Sends notifications related to ASG activities.
- **Amazon Route 53:** Provides domain name registration and DNS management for the website.

## **Deployment Scripts**

This repository contains two primary scripts to automate the setup and configuration of the WordPress website on AWS.

### **1. WordPress Installation Script**

This script is executed on the EC2 instance to set up the WordPress application. It installs necessary packages, mounts the EFS file system, and configures Apache and MySQL services.

#### **Script Details:**
```bash
# Switch to the root user
sudo su

# Update software packages on the EC2 instance
sudo yum update -y

# Create an HTML directory
sudo mkdir -p /var/www/html

# Set environment variable for EFS DNS name
EFS_DNS_NAME=fs-064e9505819af10a4.efs.us-east-1.amazonaws.com

# Mount the EFS to the HTML directory
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install Apache web server, enable it on boot, and start it
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP and necessary extensions
sudo dnf install -y \
php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext \
php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo \
php-openssl php-pdo php-tokenizer

# Install MySQL 8 Community Repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"

# Install MySQL server
sudo dnf install -y mysql-community-server

# Start and enable MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set permissions for web directory
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
chown apache:apache -R /var/www/html

# Download WordPress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# Create the wp-config.php file
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# Edit the wp-config.php file
sudo vi /var/www/html/wp-config.php

# Restart the web server
sudo service httpd restart
```

### **2. Auto Scaling Group Launch Template Script**

This script is part of the launch template used by the Auto Scaling Group to ensure new EC2 instances are configured correctly upon launch.

#### **Script Details:**
```bash
#!/bin/bash
# Update software packages
sudo yum update -y

# Install Apache web server, enable it on boot, and start it
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP and necessary extensions
sudo dnf install -y \
php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext \
php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo \
php-openssl php-pdo php-tokenizer

# Install MySQL 8 Community Repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"

# Install MySQL server
sudo dnf install -y mysql-community-server

# Start and enable MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set EFS DNS name environment variable
EFS_DNS_NAME=fs-02d3268559aa2a318.efs.us-east-1.amazonaws.com

# Mount the EFS to the HTML directory
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# Set permissions
chown apache:apache -R /var/www/html

# Restart the web server
sudo service httpd restart
```

## **How to Use**

1. Clone this repository to your local machine.
2. Follow the AWS documentation to create the necessary resources as outlined in the architecture overview.
3. Execute the provided scripts to set up the WordPress application on EC2 instances within the VPC.
4. Configure the Auto Scaling Group, Load Balancer, and other AWS services as per the architecture.
5. Access your WordPress website via the DNS name of the Application Load Balancer.

## **Problems Solved by This Project**

By completing this project, I was able to solve several key challenges in deploying a scalable and secure WordPress website on AWS:

1. **High Availability and Fault Tolerance:** The architecture leverages multiple Availability Zones, ensuring the website remains accessible even if one AZ experiences an outage.
2. **Scalability:** The Auto Scaling Group dynamically adjusts the number of EC2 instances based on traffic, ensuring the website can handle varying loads without manual intervention.
3. **Secure Access:** Security Groups and private subnets provide enhanced security by limiting direct access to web servers.
4. **Persistent Storage:** Amazon EFS offers scalable and shared file storage that persists data across multiple instances, ensuring data availability and consistency.
5. **Database Management:** Using Amazon RDS simplifies database administration by providing automated backups, patching, and scaling.
6. **SSL/TLS Encryption:** AWS Certificate Manager simplifies the management of SSL/TLS certificates, securing communication between the website and its users.
7. **DNS Management:** Amazon Route 53 provides reliable domain name resolution, ensuring users can easily access the website.
8. **Automated Deployment:** The provided scripts automate the setup process, reducing manual configuration errors and speeding up deployment.

---
**End of Documentation**

