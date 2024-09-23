# wordpress_site
Setting Up VPC and Load Balancers

# 1. VPC setup
# 1.1 IP Address Range Definition
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=MyVPC}]'

# 1.2 VPC Creation
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet}]'

aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet}]'

# 1.3 Route Table Configuration
# Public Route Table
aws ec2 create-route-table --vpc-id <vpc-id> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]'

aws ec2 associate-route-table --route-table-id <public-route-table-id> --subnet-id <public-subnet-id>

aws ec2 create-route --route-table-id <public-route-table-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <igw-id>

# Private Route Table
aws ec2 create-route-table --vpc-id <vpc-id> --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTable}]'

aws ec2 associate-route-table --route-table-id <private-route-table-id> --subnet-id <private-subnet-id>

# 2 Public and Private Subnet with NAT Gateway 
# 2.1 Public Subnet Setup
aws ec2 create-nat-gateway --subnet-id <public-subnet-id> --allocation-id <eip-allocation-id> --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=MyNatGateway}]'

# 2.2 Private Subnet Setup 
aws ec2 create-route --route-table-id <private-route-table-id> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <nat-gateway-id>

aws ec2 create-security-group --group-name PublicSecurityGroup --description "Public security group" --vpc-id <vpc-id>

# 2.3 NAT Gateway Configuration
aws ec2 allocate-address --domain vpc

# 3. AWS MySQL RDS Setup 
# 3.1 RDS Instance Creation 
    aws rds create-db-instance \
    --db-instance-identifier mydbinstance \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --allocated-storage 20 \
    --master-username admin \
    --master-user-password password \
    --vpc-security-group-ids <sg-id> \
    --db-subnet-group-name mydbsubnetgroup \
    --backup-retention-period 7 \
    --availability-zone us-east-1a \
    --no-publicly-accessible

# 3.2 Security Group Configuration
aws ec2 create-security-group --group-name MySQLSecurityGroup --description "Security group for MySQL RDS" --vpc-id <vpc-id>

# 3.2 WordPress_RDS Connection 
mysql -h <rds-endpoint> -P 3306 -u admin -p

# 4. EFS Setup for WordPress Files 
# 4.1 EFS File System Creation 
aws efs create-file-system --performance-mode generalPurpose --throughput-mode bursting --output json

# Mounting EFS 
aws efs create-mount-target --file-system-id <your-efs-id> --subnet-id <your-public-subnet-id> --security-groups <your-security-group-id>

# 4.3 WordPress Configuration 
#1. create the html directory and mount the efs to it
sudo su
yum update -y
mkdir -p /var/www/html
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com:/ /var/www/html


#2. install apache 
sudo yum install -y httpd httpd-tools mod_ssl
sudo systemctl enable httpd 
sudo systemctl start httpd


#3. install php 7.4
sudo amazon-linux-extras enable php7.4
sudo yum clean metadata
sudo yum install php php-common php-pear -y
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y


#4. install mysql5.7
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld


#5. set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
chown apache:apache -R /var/www/html 


#6. download wordpress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
cp -r wordpress/* /var/www/html/


#7. create the wp-config.php file
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php


#8. edit the wp-config.php file
nano /var/www/html/wp-config.php


#9. restart the webserver
service httpd restart

# 5. Application Load Balancer and Auto Scaling 
# 5.1 ALB Creation 
aws elbv2 create-load-balancer --name WordPressALB --subnets <your-public-subnet-id-1> <your-public-subnet-id-2> --security-groups <your-security-group-id> --output json

# 5.2 Listener Rules Configuration 
aws elbv2 create-listener --load-balancer-arn <your-load-balancer-arn> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<your-target-group-arn>

# 5.3 Intergration with Auto Scaling 
aws autoscaling create-auto-scaling-group --auto-scaling-group-name WordPressASG --launch-template "LaunchTemplateName=WordPressTemplate,Version=$Latest" --min-size 1 --max-size 5 --desired-capacity 2 --vpc-zone-identifier "<your-private-subnet-id-1>,<your-private-subnet-id-2>" --output json

aws autoscaling put-scaling-policy --auto-scaling-group-name WordPressASG --policy-name ScaleOut --scaling-adjustment 1 --adjustment-type ChangeInCapacity
aws autoscaling put-scaling-policy --auto-scaling-group-name WordPressASG --policy-name ScaleIn --scaling-adjustment -1 --adjustment-type ChangeInCapacity

