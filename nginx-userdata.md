#!/bin/bash
set -euxo pipefail

# Update system
dnf update -y

# Install nginx and git
dnf install -y nginx git

# Start and enable nginx
systemctl enable nginx
systemctl start nginx

# Clone repo
cd /tmp
rm -rf wakabetter-project-config
git clone https://github.com/BosunEnoch7/wakabetter-project-config.git

# Backup default nginx config
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup

# Replace with your reverse proxy config
cp /tmp/wakabetter-project-config/reverse.conf /etc/nginx/nginx.conf

# Restart nginx
systemctl restart nginx
