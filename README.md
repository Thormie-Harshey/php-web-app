
# Building a Scalable WordPress Site on AWS
![Architecture Diagram](https://github.com/Thormie-Harshey/php-web-app/blob/main/images/three%20tier%20archi.jpg)
---

## Project Goal
The objective of this project was to design, implement, and deploy a highly available and scalable WordPress website on AWS cloud. The solution leverages a three-tier architecture to ensure security and separation of concerns, dividing the setup into Web, Application, and Database layers.

---

##  Architecture & Technologies Used

| Category        | Tools / Services |
|----------------|------------------|
| **Cloud** | AWS |
| **Networking** | VPC, Subnets (Public & Private), Route Tables, NAT Gateway |
| **Load Balancing** | Application Load Balancer (ALB) |
| **Compute** | EC2 (Auto Scaling Group, Launch Templates) |
| **Database** | Amazon RDS for MySQL |
| **Storage** | Amazon EFS |
| **Automation** | User Data Scripts |
| **Security** | Security Groups, EC2 Instance Connect Endpoint (ICE), AWS Certificate Manager (ACM) |
| **Web Server** | Apache (httpd) |
| **App Stack** | PHP, WordPress |
| **Domain** | GoDaddy |
---

##  Implementation & Workflow

### 1. Building the Foundation: VPC and Subnet Creation
To begin, we’ll create a Virtual Private Cloud (VPC) in the US-East-1 (North Virginia) region. Of course, feel free to pick the most appropriate region for your needs.
- **VPC CIDR**: `172.31.0.0/16`
- **3 Public & 3 Private Subnets** across multiple AZs.
- **Routing:**
  - Public Subnet → Internet Gateway
  - Private Subnet → NAT Gateway (for internet-bound traffic)
  - Public Route Table → Internet Gateway, allowing outbound traffic from public subnets.
  - Private Route Table → NAT Gateway to enable outbound internet access for resources in the private subnets (for updates and patches).
---

### 2. Security Groups

| SG | Description |
|----|-------------|
| **ICE SG** | Allows outbound TCP 22 within VPC for EC2 Instance Connect |
| **ALB SG** | Inbound HTTP/HTTPS from anywhere |
| **App Server SG** | Accepts traffic from ALB SG, SSH only from ICE SG |
| **RDS SG** | Allows MySQL (3306) from App Server SG only |
| **EFS SG** | NFS from App Server SG, and from itself |


Our configuration is now such that: Any traffic coming into our application server must come from the load balancer, or it will be dropped. Also, any request made to the database must come from the security group of the application server, or else, it will be dropped. Same thing for the EFS, any traffic not coming from the security group of the application server should be dropped as well.

---

### 3. EC2 Instance Connect Endpoint (ICE) Setup

EC2 Instance Connect Endpoint eliminates the need for bastion hosts and SSH keys. It enables keyless SSH access to private EC2 instances via AWS Console or CLI.

**For the CLI, run the command**:
```bash
aws ec2-instance-connect ssh --instance-id <INSTANCE_ID>
```
The `<instance_id>` is obtained from your EC2 instance details after creation. Also, make sure you have your AWS account configured on your CLI.

**For the console, follow the steps below:**
**Step 1: Navigate to the VPC Console**

-   Go to the VPC Dashboard in the AWS Management Console.
-   Select "Endpoints" from the left sidebar menu.

**Step 2: Create a New Endpoint**

-   Click on "Create Endpoint".
-   Under "Service category", select "EC2 Instance Connect Endpoint".

**Step 3: Configure Endpoint Settings**

-   **VPC**: Choose the VPC where your EC2 instances are located.
    -   **Security Groups**: Attach a security group `(EICE SG)` which was created earlier.
-   **Subnets**: Select the subnet `(private subnet)` where you want to create the Instance Connect Endpoint. 

**Step 4: Review and Create the Endpoint**

-   Click on "Create endpoint" to finish the setup.
-   Wait for the status to change to "Available".

**Step 5: Access Your EC2 Instances Using EC2 Instance Connect**

-   Go to the EC2 Dashboard.
-   Deploy an EC2 instance (Amazon Linux 2023) in a private subnet (without a keypair).
-   Click "Connect", then select the "EC2 Instance Connect Endpoint" option.
-   Select the "Endpoint" created earlier.
-   Click "Connect" to establish an SSH session to your instance directly through the AWS Console.

---

### 4. RDS (MySQL) Setup
The first step before creating a database is to create a subnet group.

-   Subnet groups are to be created in private subnets

As for the database, here are some important attributes:
    
-   **Engine**: MySQL 8.0.37
    
-   **Instance Class**: `t3.micro`
    
-   Auto-generated credentials and self-managed keys
    
-   The Initial Database Name must be defined in “Additional Config”

   We can decide to "disable automatic backup," since the environment is the dev environment and not production.

----------

### 5. EFS Setup

Amazon EFS was configured to serve as the shared, persistent storage for the WordPress application files. Its mount script is shown below:

**Mount Script**:
```bash
EFS_DNS_NAME=fs-xxxxxxxx.efs.us-east-1.amazonaws.com
sudo mkdir -p /var/www/html
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html
```
### 6. EC2 Instance & WordPress Setup
A single EC2 instance was launched to serve as the golden image for the auto scaling group. The WordPress code was installed and configured, and the EFS volume was mounted and validated.

#### Key Installations on the EC2 instance:

-   Apache, PHP with required extensions
    
-   MySQL client
    
-   WordPress (downloaded and extracted)
    
-   EFS mounted at `/var/www/html`
    

#### Setup Script :
```bash
#!/bin/bash

# Update the package repository
sudo yum update -y

# Create /var/www/html directory
sudo mkdir -p /var/www/html

# Variable for EFS
EFS_DNS_NAME=fs-xxxxxxxx.efs.us-east-1.amazonaws.com

# EFS Mount to the /var/www/html
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install apache2
sudo yum install git httpd -y

# Install apache and php dependencies for the php web app
sudo yum install -y \
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
php-soap \
php-tokenizer

# Install Mysql-Client 
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 

# Start and enable Apache & Mysql server
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl start mysqld
sudo systemctl enable mysqld

# set /var/www/html directory permissions
sudo usermod -aG apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
chown apache:apache -R /var/www/html 

# Download Wordpress files and copy to /var/www/html
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# create the wp-config.php file
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
```
Configure `wp-config.php` with RDS credentials (the name, the user, the password, and the host). From here, we can reload httpd with the command:
`(sudo systemctl reload httpd)`

----------

### 7. Load Balancer + Domain Mapping
To ensure scalability and high availability, an Application Load Balancer (ALB) was configured to distribute incoming internet traffic across EC2 instances running WordPress. This ALB was set to be internet-facing, as outlined in the solution architecture.

-   **Listeners** were configured with HTTP (80) and HTTPS (443) listeners
    
-   **Target group** registers EC2 instances
    
-   **Domain**: GoDaddy with its CNAME pointing to ALB DNS
    
-   The **WordPress site URL** was updated in the Wordpress website under **Settings > General**
    

----------

### 8. SSL Certificate via ACM
To secure the domain with HTTPS, an SSL certificate was provisioned using AWS Certificate Manager by:

-   Public certificate requested via **AWS ACM**
    
-   Domain validation done via GoDaddy DNS (CNAME record)
    
-   HTTPS listener added to ALB listener configuration using the ACM-issued certificate
    
-   Traffic Redirected from HTTP → HTTPS at the ALB level

#### WordPress SSL Config:
After configuring the ALB, connect to the EC2 instance via EC2 Instance Connect and edit the WordPress configuration file:`/var/www/html/wp-config.php`
Add the following lines to force SSL usage within the WordPress admin dashboard and ensure proper handling of HTTPS headers forwarded by the ALB:
```php
define('FORCE_SSL_ADMIN', true);
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}
```
### 9. Auto Scaling Setup
We allow autoscaling to create more instances and add them to the target group. A launch template was created from the configured EC2 instance. This template, including a user data script, was used to create an Auto Scaling Group that automatically provisions new instances as needed, ensuring the application remains highly available. 
Its User data ensures Apache, PHP, EFS mount, and WordPress setup are handled automatically
   
----------
**The architecture is now capable of handling traffic fluctuations gracefully, with minimal administrative overhead, and provides a solid foundation for future development and scaling.**

