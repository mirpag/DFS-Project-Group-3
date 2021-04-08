# Pre-installation

> Our group installed Nextcloud on both ubuntu1-group3 and ubuntu3-group3. This document uses `ubuntuN` as a stand-in for whichever one of the two is being worked on.

**On a fresh Ubuntu Server 20.04 installation, with a static IP and Fully-Qualified Domain Name *ubuntuN-group3.group3.local*.**

0. Update to latest software: `sudo apt upgdate && sudo apt upgrade`

1. Install dependencies:

```bash
sudo apt install unzip
sudo apt install apache2 mariadb-server libapache2-mod-php7.4
sudo apt install php7.4-gd php7.4-mysql php7.4-curl php7.4-mbstring php7.4-intl
sudo apt install php7.4-gmp php7.4-bcmath php-imagick php7.4-xml php7.4-zip
```

2. Set up Database: run `sudo mysql -uroot -p`

```SQL
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY 'somepassword';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SER utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost';
FLUSH PRIVILEGES;
```
*exit MariaDB with **Ctrl+d**.*

3. Add a CNAME DNS record for ***cloud.ubuntuN-group3.group3.local*** pointing to ***ubuntu.ubuntuN-group3.group3.local***

4. Download Nextcloud

```bash
sudo -i
cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-x.y.z.zip -O nextcloud.zip # replace x.y.z with the latest stable version (as of writing, 21.0.0)
sha256sum nextcloud.zip # Make sure this matches the SHA265 digest available on the nextcloud website!
unzip nextcloud
cp -r nextcloud /var/www

```

# Configure Apache

Create a configuration file at ***/etc/apache2/sites-available/nextcloud.conf*** with the following contents:

```apacheconf
<VirtualHost *:80>
    ServerName cloud.ubuntuN-group3.group3.local
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>
</VirtualHost>

```

Enable the site with the command `a2ensite nextcloud.conf`, and restart the server with `systemctl restart apache2`

# Installation

0. Set proper ownership: `sudo chown -R www-data:www-data /var/www/nextcloud/`

1. Switch to the Nextcloud web root: `cd /var/www/nextcloud`

2. Install Nextcloud with either the command-line or the in-browser wizard

## Command-line installation (done on `ubuntu1-group3`):

run the following behemoth of a multi-line command:

```bash
sudo -u www-data php occ maintenance:install \
database "mysql" --database-name "nextcloud" \
--database-user "nextcloud" --database-pass "somepassword" \
--admin-user "group3" --admin-pass "anotherpassword"
```
## In-browser wizard installation (done on `ubuntu3-group3`)

0. Install Firefox on mgmt01-group3 (Nextcloud does not work on Internet Explorer)

1. In Firefox, open *http<nolink>://cloud.ubuntu3-group3.group3.local*

2. Fill out the various forms with the requried information (Database is `mysql/mariadb`)

Note: on our `ubuntu3` installation, the data directory was */var/www/nextcloud-data*, rather than the default */var/www/nextcloud/data*

# Post-installation

## Add Trusted Domain

Edit the file **/var/www/nextcloud/config/config.php**, adding `1 => 'cloud.ubuntuN-group3.group3.local',` as an entry within the `trusted_domains` array.

## Set up backups

### On cent7a-group3

Install required software with `sudo yum -y install nfs-utils`

Run the following:

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
sudo mkdir -p /nfsshare/ubuntu{1,3}
echo '/nfsshare *(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo systemctl enable --now nfs
exportfs -a
```

### On ubuntu1-group3 and ubuntu3-group3

Install required software with `sudo apt install lsyncd nfs-common -y`

Mount the NFS directory: `sudo mkdir /nfsshare && sudo mount.nfs cent7a-group3:/nfsshare /nfsshare`

Start **lsyncd**: `sudo lsyncd -rsync /var/www/nextcloud/data /nfsshare/ubuntuN-group3`

Automatically do the above at boot using **cron**: run `sudo crontab -e` to edit the default crontab, and add the following line:

```cron
@reboot mount.ntfs cent7a-group3:/nfsshare /nfsshare && lsyncd -rsync /var/www/nextcloud/data /nfsshare/ubuntu1 # on ubuntu1-group3
@reboot mount.ntfs cent7a-group3:/nfsshare /nfsshare && lsyncd -rsync /var/www/nextcloud-data /nfsshare/ubuntu3 # on ubuntu3-group3
```

<details>
    <summary> About lsyncd </summary>
    <a href="https://github.com/axkibe/lsyncd"><code>lsyncd</code></a> uses <code>rsync</code>, a tool for efficient file transferring and syncing, and uses <code>inotify</code>, a Linux kernel subsystem, to monitor for file changes, and instantly back them up.
</details>

**Relevant documentation**

The following documentation and tutorials were immensly helpful in this project:

Nextcloud Administrator Documentation Pages:

* [Example installation on Ubuntu 20.04 LTS](https://docs.nextcloud.com/server/21/admin_manual/installation/example_ubuntu.html)
* [Installation wizard](https://docs.nextcloud.com/server/21/admin_manual/installation/installation_wizard.html)
* [Installation on Linux](https://docs.nextcloud.com/server/21/admin_manual/installation/source_installation.html)
* [Installing from command line](https://docs.nextcloud.com/server/21/admin_manual/installation/command_line_installation.html)

Online Tutorials:

* [Setting Up an NFS Server and Client on CentOS 7.2 | HowToForge](https://www.howtoforge.com/tutorial/setting-up-an-nfs-server-and-client-on-centos-7/)
* [Install NextCloud on Ubuntu 20.04 with Apache (LAMP Stack) | LinuxBabe](https://www.linuxbabe.com/ubuntu/install-nextcloud-ubuntu-20-04-apache-lamp-stack)

`man` pages:

* [lsyncd](http://manpages.ubuntu.com/manpages/focal/man1/lsyncd.1.html)

