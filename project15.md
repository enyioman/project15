# AWS Cloud Solution For 2 Company Websites Using A Reverse Proxy Technology

## Introduction

This project demonstrates how a secure infrastructure inside AWS VPC (Virtual Private Cloud) network is built for a particular company, which uses WordPress CMS for its main business website, and a Tooling Website for their DevOps team. As part of the company’s desire for improved security and performance, a decision has been made to use a reverse proxy technology from NGINX to achieve this. The infrastructure will look like following diagram:


![Project architecture](./media/archy.png)

## STEP 1: Setting Up a Sub-account And Creating A Hosted Zone

- Properly configure the AWS account and Organization Unit. Use this [video](https://www.youtube.com/watch?v=9PQYCc_20-Q&ab_channel=CloudAcademy) to implement the setup.
- Create an AWS Master account. (Also known as Root Account).
- Within the Root account, create a sub-account and name it DevOps. (A different email address is required to complete this).
- Within the Root account, create an AWS Organization Unit (OU). Name it Dev. (The Dev resources will be launched in there).

![Organization unit](./media/ou.png)

- Move the DevOps account into the Dev OU.

![OU user](./media/ouuser.png)

- Login to the newly created AWS account

![OU login](./media/oulogin.png)

- Creating a hosted zone in the Route 53 console and mapping it to the domain name acquired from freenom.

![Hosted zone](./media/hostedzone.png)

### TLS Certificates From Amazon Certificate Manager (ACM)

TLS certificates are required to handle secured connectivity to the Application Load Balancers (ALB).

- Navigate to AWS ACM
- Request a public wildcard certificate for the domain name previously registered
- Use DNS to validate the domain name
- Tag the resource

![TLS certificate](./media/certificate.png)


## STEP 2: Setting Up a Virtual Private Network (VPC)

1. Create a VPC and also enbale DNS Hostname resolution for the VPC.

![VPC](./media/vpccreation.png)

2. Create subnets as shown in the architecture.

![Subnet](./media/subnet1.png)

![Subnet](./media/subnets.png)

3. Create 2 route tables and associate each with public and private subnets accordingly. 


![Route table](./media/publicrtb.png)

![Route table](./media/privatertb2.png)


4. Create an Internet Gateway and attach it to the VPC.

![Internet gateway](./media/igw.png)

![Internet gateway](./media/igwattach.png)

5. Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accessible from the Internet). Specify 0.0.0.0/0 in the Destination box, and select the internet gateway ID in the Target list.

![Internet gateway](./media/igwasso.png)


6. Create 3 Elastic IPs.

7. Create a Nat Gateway and assign one of the Elastic IPs (The other 2 will be used by Bastion hosts). Add a new route on the private route table to configure destination as 0.0.0.0/0 and target as NAT Gateway.

![Nat gateway](./media/natgw.png)

![Nat gateway](./media/natasso.png)

8. Create a Security Group for:

- Nginx Servers: Access to Nginx should only be allowed from a External Application Load balancer (ALB). At this point, a load balancer has not been created, therefore the rules will be updatedlater. For now, just create it and put some dummy records as a place holder.

- Bastion Servers: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. Hence, use the workstation public IP address. To get this information, simply go to terminal and type curl www.canhazip.com.

- External Application Load Balancer: The External ALB will be available from the Internet and also will allow SSH from the bastion host.

- Internal Application Load Balancer: This ALB will only allow HTTP and HTTPS traffic from the Nginx Reverse Proxy Servers.

- Webservers: The security group should allow SSH from bastion host, HTTP and HTTPS from the internal ALB only.

- Data Layer: Access to the Data layer, which is comprised of Amazon Relational Database Service (RDS) and Amazon Elastic File System (EFS) must be carefully designed - webservers need to mount the file system and connect to the RDS database, bastion host needs to have SQL access to the RDS to use the MYSQL client.


![Security groups](./media/sgs.png)

## STEP 3: Create EFS

1. Create an EFS filesystem.
2. Create an EFS mount target per AZ in the VPC, associate it with both subnets dedicated for data layer. Mount the EFS in both AZ in the same private subnets as the webservers (private-subnet-01 and private-subnet-02).
3. Associate the Security groups created earlier for data layer.
4. Create two separate EFS access points- one for wordpress and one for tooling. (Give it a name and leave all other settings as default). Set the POSIX user and Group ID to root(0) and permissions to 0755.


![EFS](./media/efs.png)

![EFS access point](./media/routecreated.png)


## STEP 4: Create RDS

Create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance.

- On KMS Console, choose Symmetric and Click Next. Assign an Alias.

- Select user with admin privileges as the key administrator.

- Click Create Key.


![KMS](./media/kms.png)


## STEP 5:  Create DB Subnet Group

On the RDS Management Console, create a DB subnet group with the two datalayer private subnets in the two Availability Zones.

![RDS subnet group](./media/rdssubnetgroup.png)


### Create RDS

- Click on Create Database
- Select MySQL
- Choose Free Tier from the Templates
- Enter a name for the DB
- Create Master username and passsword
- Select the VPC, select the subnet group that was created earlier and the database security group
- Scroll down to Additional configuration
- Enter initial database name
- Scroll down and click Create database

![RDS creation](./media/rdscreation.png)


## STEP 6: Create Compute Resources

Configure compute resources inside the VPC. The recources related to compute are:

- EC2 Instances
- Launch Templates
- Target Groups
- Autoscaling Groups
- TLS Certificates
- Application Load Balancers (ALB)

Configure the Public Subnets as follows so the instances can be assigned a public IPv4 address:

- In the left navigation pane, choose Subnets.
- Select the public subnet for the VPC. By default, the name created by the VPC wizard is Public subnet.
- Choose Actions, Modify auto-assign IP settings.
Select the Enable auto-assign public IPv4 address check box, and then choose Save.
- Launch templates.

First create three EC2 instance for Nginx server, Bastion server and Webserver.
- Launch 3 EC2 instances (Red Hat Free Tier).


![Instances](./media/3servers.png)

### Install the following packages:


```
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm 
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm 
yum install wget vim python3 telnet htop git mysql net-tools chrony -y 
systemctl start chronyd
systemctl enable chronyd
```

- Disable senten so that our servers can function properly on all the redhat instance (Nginx & Webserver only):

```
setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1
```

- Install Amazon efs utils for mounting targets on the elastic file system (Nginx & webserver only):

```
git clone https://github.com/aws/efs-utils
cd efs-utils
yum install -y make
yum install -y rpm-build
make rpm 
yum install -y  ./build/amazon-efs-utils*rpm
```

- Install self-signed certificate on Nginx. The reason we are installing this is because the load balancer will be sending traffic to the webserver via port 443 and also listen on port 443 thus for the connection to be secured we need a self signed certificate on the nginx instance.

```
sudo mkdir /etc/ssl/private
sudo chmod 700 /etc/ssl/private
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/ACS.key -out /etc/ssl/certs/ACS.crt
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

- Confirm that the ACS.crt and ACS.key files exist.


![Confirm certificate](./media/certverify.png)


- Install self signed certificate for the Webserver:

```
yum install -y mod_ssl
openssl req -newkey rsa:2048 -nodes -keyout /etc/pki/tls/private/ACS.key -x509 -days 365 -out /etc/pki/tls/certs/ACS.crt
vi /etc/httpd/conf.d/ssl.conf
```

Edit the ssl.conf to conform with the key and crt file we created.


![Modify ssl.conf](./media/conf.png)


### Create AMIs from the Nginx, Bastion and Webserver Instances

## STEP 7: Create Launch Templates

For bastion host launch template: 

- Set up a launch template with the Bastion AMI.
- Ensure the instances are launched into the public subnet.
- Enter the userdata to update yum package repository and install ansible and mysql.

```
#!/bin/bash 
yum install -y mysql 
yum install -y git tmux 
yum install -y ansible
```

For Nginx Server Launch Template:

- Set up a launch template with the Nginx AMI.
- Ensure the instances are launched into the public subnet.
- Assign appropriate security group.
- Enter the following userdata:

```
#!/bin/bash
yum install -y nginx
systemctl start nginx
systemctl enable nginx
git clone https://github.com/enyioman/ACS-project-config.git
mv ACS-project-config/reverse.conf /etc/nginx/
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-distro
cd /etc/nginx/
touch nginx.conf
sed -n 'w nginx.conf' reverse.conf
systemctl restart nginx
rm -rf reverse.conf
rm -rf ACS-project-config
```

For Tooling Server Launch Template: 

- Set up a launch template with the Webserver AMI.
- Ensuring the instances are launched into the private subnet.
- Assign appropriate security group.
- Enter the following userdata. Ensure you modify the EFS mount point.

```
#!/bin/bash
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-056fe1371c5d5f2c0 fs-0c25c2be9879367d5:/ /var/www/
yum install -y httpd 
systemctl start httpd
systemctl enable httpd
yum module reset php -y
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl enable php-fpm
git clone https://github.com/enyioman/tooling.git
mkdir /var/www/html
cp -R tooling/html/*  /var/www/html/
cd tooling
mysql -h toolingdb.c9tkkvrdsgea.us-east-1.rds.amazonaws.com -u admin -p toolingdb < tooling-db.sql
cd /var/www/html/
touch healthstatus
sed -i "s/$db = mysqli_connect('mysql.tooling.svc.cluster.local', 'admin', 'password', 'toolingdb');/$db = mysqli_connect('toolingdb.c9tkkvrdsgea.us-east-1.rds.amazonaws.com', 'admin', 'password', 'toolingdb');/g" functions.php
chcon -t httpd_sys_rw_content_t /var/www/html/ -R
systemctl restart httpd
```

For Wordpress Server Launch Template

- Set up a launch template with the Webserver AMI.
- Ensure the instances are launched into the private subnet.
- Assign appropriate security group.
- Use the following userdata. Ensure you modify the RDS endpoint and EFS mount point.

```
#!/bin/bash
mkdir /var/www/
sudo mount -t efs -o tls,accesspoint=fsap-087ead6ff2a98bc10 fs-0c25c2be9879367d5:/ /var/www/
yum install -y httpd 
systemctl start httpd
systemctl enable httpd
yum module reset php -y
yum module enable php:remi-7.4 -y
yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
systemctl start php-fpm
systemctl enable php-fpm
wget http://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
mkdir /var/www/html/
cp -R /wordpress/* /var/www/html/
cd /var/www/html/
touch healthstatus
sed -i "s/localhost/toolingdb.c9tkkvrdsgea.us-east-1.rds.amazonaws.com/g" wp-config.php 
sed -i "s/admin/g" wp-config.php 
sed -i "s/password/g" wp-config.php 
sed -i "s/tooling/g" wp-config.php 
chcon -t httpd_sys_rw_content_t /var/www/html/ -R
systemctl restart httpd
```

