#!/bin/bash
set -euxo pipefail

dnf update -y

# Install required packages first
dnf install -y amazon-efs-utils httpd wget tar php php-cli php-common php-mbstring php-xml php-gd php-curl php-mysqlnd php-fpm

# Create mount point
mkdir -p /var/www

# Mount EFS
mount -t efs -o tls,accesspoint=fsap-028b8045c50f7e635 fs-0cf945c5a63141f5b:/ /var/www

# Persist EFS mount
echo "fs-0cf945c5a63141f5b:/ /var/www efs _netdev,tls,accesspoint=fsap-028b8045c50f7e635 0 0" >> /etc/fstab

# Start services
systemctl enable httpd
systemctl start httpd
systemctl enable php-fpm
systemctl start php-fpm

# Download and extract WordPress
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
rm -f latest.tar.gz

# Prepare WordPress config
cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php

# Create web root and copy files
mkdir -p /var/www/html
cp -R /tmp/wordpress/* /var/www/html/

# Health check file
touch /var/www/html/healthstatus

# Update WordPress DB settings
sed -i "s/database_name_here/wordpressdb/g" /var/www/html/wp-config.php
sed -i "s/username_here/admin/g" /var/www/html/wp-config.php
sed -i "s/password_here/YOUR_RDS_PASSWORD/g" /var/www/html/wp-config.php
sed -i "s/localhost/zhikbee-rds.cg22cjxlsbrj.us-east-2.rds.amazonaws.com/g" /var/www/html/wp-config.php

# SELinux context
chcon -Rt httpd_sys_rw_content_t /var/www/html

# Restart services
systemctl restart php-fpm
systemctl restart httpd
