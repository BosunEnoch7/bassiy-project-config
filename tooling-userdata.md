#!/bin/bash
set -euxo pipefail

dnf update -y

# Enable required repo
dnf install -y epel-release

# Install EFS client, Apache, Git, MySQL client, and PHP 8.1
dnf install -y amazon-efs-utils httpd git wget mysql php php-cli php-common php-mbstring php-xml php-gd php-curl php-mysqlnd php-fpm

# Create mount point
mkdir -p /var/www

# Mount EFS with access point
mount -t efs -o tls,accesspoint=fsap-087db5a73c7b8725b fs-0cf945c5a63141f5b:/ /var/www

# Persist EFS mount
echo "fs-0cf945c5a63141f5b:/ /var/www efs _netdev,tls,accesspoint=fsap-087db5a73c7b8725b 0 0" >> /etc/fstab

# Start and enable Apache + PHP-FPM
systemctl enable httpd php-fpm
systemctl start httpd php-fpm

# Clone application
cd /tmp
rm -rf tooling
git clone https://github.com/BosunEnoch7/tooling.git

# Prepare web root
mkdir -p /var/www/html
cp -R /tmp/tooling/html/* /var/www/html/

# Import database
cd /tmp/tooling
mysql -h zhikbee-rds.cg22cjxlsbrj.us-east-2.rds.amazonaws.com -u admin -p'YOUR_RDS_PASSWORD' toolingdb < tooling-db.sql

# Create health check file
touch /var/www/html/healthstatus

# Update DB connection
sed -i "s/mysql.tooling.svc.cluster.local/zhikbee-rds.cg22cjxlsbrj.us-east-2.rds.amazonaws.com/g" /var/www/html/functions.php
sed -i "s/'tooling'/'toolingdb'/g" /var/www/html/functions.php
sed -i "s/'admin', 'admin'/'admin', 'YOUR_RDS_PASSWORD'/g" /var/www/html/functions.php

# SELinux context
chcon -Rt httpd_sys_rw_content_t /var/www/html

# Restart Apache
systemctl restart httpd
