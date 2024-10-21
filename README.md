#!/bin/bash

# Nextcloud Setup and Troubleshooting on Hyper-V

# Initial Update and Upgrade
sudo apt update && sudo apt upgrade -y

# Installing exFAT Utilities for External Drive Support
sudo apt install exfat-fuse exfat-utils

# Mounting the External exFAT Drive for Nextcloud
sudo mount -t exfat /dev/sdb2 /media/externalhd -o uid=www-data,gid=www-data,fmask=0022,dmask=0022

# Downloading and Installing Nextcloud Manually
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
tar -xjf latest.tar.bz2 -C /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud

# Installing Apache, MariaDB, PHP, and Necessary Modules
sudo apt install apache2 mariadb-server libapache2-mod-php php php-mysql php-xml php-mbstring php-curl php-zip php-gd

# Configuring Apache Virtual Host for Nextcloud
sudo nano /etc/apache2/sites-available/nextcloud.conf
# Inside the file, add:
# <VirtualHost *:80>
#    DocumentRoot /var/www/nextcloud
#    ServerName 192.168.1.4
#
#    <Directory /var/www/nextcloud/>
#        Require all granted
#        AllowOverride All
#        Options FollowSymLinks MultiViews
#    </Directory>
#
#    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
#    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
# </VirtualHost>

# Enabling the New Apache Site and Restarting Apache
sudo a2ensite nextcloud.conf
sudo systemctl reload apache2

# Configuring MariaDB for Nextcloud
sudo mysql -u root -p
# Inside MariaDB prompt, run the following:
# CREATE DATABASE nextcloud;
# CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'yourpassword';
# GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
# FLUSH PRIVILEGES;
# EXIT;

# Running Nextcloud Setup Wizard in Browser
# Navigate to http://192.168.1.4 in the browser and complete the wizard using the created database credentials

# Adding External Storage in Nextcloud
# Enable the External Storage app in Nextcloud admin panel
# Add the external drive mounted at /media/externalhd as a new storage location

# Manually Scanning for New Files and Changes in Nextcloud
sudo -u www-data php /var/www/nextcloud/occ files:scan --all

# Automating File Scans Using a Cron Job
crontab -e
# Add the following line to run the scan every 15 minutes:
# */15 * * * * php -f /var/www/nextcloud/cron.php

# Fixing Permissions Issues for External Storage
sudo chown -R www-data:www-data /media/externalhd
sudo chmod -R 770 /media/externalhd

# Disabling Server-Side Encryption After Issues with External Storage
sudo -u www-data php /var/www/nextcloud/occ encryption:decrypt-all
sudo -u www-data php /var/www/nextcloud/occ encryption:disable

# Installing and Configuring SSL with Let’s Encrypt for Secure HTTPS Access
sudo apt install certbot python3-certbot-apache
sudo certbot --apache
# Follow the prompts to obtain and apply the SSL certificate

# Final Troubleshooting: Unmount and Remount External Drive to Resolve Sync Issues
sudo umount /media/externalhd
sudo mount -t exfat /dev/sdb2 /media/externalhd -o uid=www-data,gid=www-data,fmask=0022,dmask=0022

# Running a Final Manual File Scan to Ensure Everything is Synced Correctly
sudo -u www-data php /var/www/nextcloud/occ files:scan --all

# Summary of Nextcloud Setup and Troubleshooting Complete
# The Nextcloud installation is now fully configured, external storage is set up, and SSL is applied for secure access.
