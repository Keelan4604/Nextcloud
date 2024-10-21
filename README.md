# Title: Nextcloud Setup, Troubleshooting, and Resolution Documentation
# Author: Keelan4604

# Introduction
# This outlines the entire process I followed while setting up and troubleshooting Nextcloud
# on a Hyper-V virtual machine with external storage. This process involved multiple steps including setting up
# the environment, configuring external storage, addressing file sync issues, and more.

# 1. Setting up Hyper-V
# I started by creating a new Hyper-V virtual machine on my Windows 10 system to host Ubuntu. The goal was to install Nextcloud for personal cloud storage purposes.
# 
# This setup allowed me to easily manage my own cloud storage, syncing with my external hard drive, and using an exFAT file system.

# 2. Configuring External Storage on Hyper-V
# After creating the VM and installing Ubuntu, I connected my external hard drive to the VM, formatted as exFAT.
# I mounted it using the following command:

sudo mount -t exfat /dev/sdb2 /media/ehd -o uid=www-data,gid=www-data,fmask=0022,dmask=0022

# This command mounts the external hard drive with permissions assigned to the www-data user and group,
# which is important because Nextcloud uses the www-data user for handling file read/write operations.

# 3. Installing Nextcloud Manually
# Initially, I tried installing Nextcloud via Snap, but Snap's confinement prevented me from properly accessing my external drive.
# I uninstalled the Snap package and opted for a manual installation, which provided more flexibility. Here's how I installed it manually:

# Downloading and extracting Nextcloud
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
tar -xjf latest.tar.bz2 -C /var/www/

# I then set the proper ownership for the Nextcloud directory to allow the web server to manage it:
sudo chown -R www-data:www-data /var/www/nextcloud

# Next, I installed Apache, PHP, MariaDB, and the required PHP modules for Nextcloud:
sudo apt install apache2 mariadb-server libapache2-mod-php php php-mysql php-xml php-mbstring php-curl php-zip php-gd

# 4. Configuring Apache for Nextcloud
# After installing Apache and PHP, I needed to configure Apache to serve the Nextcloud instance. I created a new virtual host configuration file:

sudo nano /etc/apache2/sites-available/nextcloud.conf

# The configuration looked like this:

<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud
    ServerName 192.168.1.4

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>

# After saving the file, I enabled the new site and restarted Apache:
sudo a2ensite nextcloud.conf
sudo systemctl restart apache2

# 5. Setting up MariaDB for Nextcloud
# I set up MariaDB as the database for Nextcloud. I logged into the MySQL shell using:

sudo mysql -u root -p

# Then I created a database and user for Nextcloud:

CREATE DATABASE nextcloud;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;

# 6. Resolving File Sync Issues Between Nextcloud and External Storage
# One of the major issues I encountered was that Nextcloud wasn’t detecting new files that were created or deleted on the external drive when the drive was connected to Windows. 
# Initially, files created in Nextcloud were visible on the drive when accessed through Windows, but files added in Windows weren’t appearing in Nextcloud.

# I manually triggered a file scan to resolve this:

sudo -u www-data php /var/www/nextcloud/occ files:scan --all

# This command forced Nextcloud to rescan the external storage, updating its index with any new or deleted files. I also used a more specific scan command for just the external storage:

sudo -u www-data php /var/www/nextcloud/occ files:scan --path="keelan4604/files_external"

# I realized that manually triggering scans every time was not ideal, so I investigated setting up automated scans. After some testing, I decided to periodically run the scan command via a cron job to keep Nextcloud's file index up to date.

# 7. Fixing Permissions Issues
# File permissions were another challenge. At first, Nextcloud couldn’t write to the external drive because the correct permissions weren’t applied. I made sure that Nextcloud (www-data) had full access to the mounted drive:

sudo chown -R www-data:www-data /media/ehd
sudo chmod -R 770 /media/ehd

# I also mounted the drive with the appropriate options each time I needed to reconnect it:
sudo mount -t exfat /dev/sdb2 /media/ehd -o uid=www-data,gid=www-data,fmask=0022,dmask=0022

# 8. Addressing File Locking Issues
# Another issue I faced was with file locking. Sometimes, files would get locked, preventing any modifications or deletions. To troubleshoot this, I temporarily disabled file locking by adding the following line to Nextcloud’s configuration file:

sudo nano /var/www/nextcloud/config/config.php

# Inside the config.php, I added:
'filelocking.enabled' => false,

# This temporarily solved the problem, allowing Nextcloud to manage files without getting locked out.

# 9. Enabling and Disabling Encryption
# At one point, I decided to enable Nextcloud’s **server-side encryption** feature, thinking it would be an added layer of security.
# However, this quickly became more of a hassle than a benefit, especially for managing files on external storage.
# When I decided to disable encryption, I used the following commands:

sudo -u www-data php /var/www/nextcloud/occ encryption:decrypt-all
sudo -u www-data php /var/www/nextcloud/occ encryption:disable

# After running these commands, I also uninstalled the **Default Encryption Module** from Nextcloud’s admin settings to prevent any future issues with encrypted files.

# 10. Securing Nextcloud with SSL
# To ensure my Nextcloud instance was secure, I installed SSL using Let’s Encrypt. This allowed me to access Nextcloud via HTTPS, securing all data transmissions.
sudo apt install certbot python3-certbot-apache
sudo certbot --apache

# I followed the prompts to generate the SSL certificates, which were then automatically applied to the Apache configuration.
# After setting up SSL, I could securely access Nextcloud from all devices, both on my local network and remotely.

# 11. Troubleshooting Sync and File Visibility
# Despite enabling SSL and configuring external storage, I still ran into occasional issues with file syncing between Windows and Nextcloud.
# Files deleted in Windows wouldn’t disappear from Nextcloud until I ran the following rescan:

sudo -u www-data php /var/www/nextcloud/occ files:scan --all

# I also ensured that the Nextcloud **External Storage App** was configured properly. I had enabled the "Check for Changes" option, but for some reason, it wasn’t always working automatically.
# Running the **occ** command became my go-to solution for these syncing problems.

# 12. Addressing Final External Storage Permissions
# To finalize everything, I ensured that all files on the external drive had proper permissions for Nextcloud to read and write.
# I verified permissions on the drive using the **ls -l** command and ensured all files belonged to **www-data**:

sudo chown -R www-data:www-data /media/ehd
sudo chmod -R 770 /media/ehd

# With these final changes, the syncing issues were resolved, and I had full read/write access to the external drive through both Windows and Nextcloud.

# 13. Conclusion
# Throughout this project, I learned a lot about server configuration, file permissions, and troubleshooting cloud storage systems.
# The biggest takeaways were understanding the relationship between file systems (like exFAT) and cloud storage platforms, as well as dealing with encryption and permissions.

# The journey of setting up Nextcloud on a Hyper-V virtual machine while using an external exFAT drive taught me valuable lessons in system administration, which I can now apply to larger-scale cloud storage solutions.
