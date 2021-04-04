# Pre-installation

**On a fresh Ubuntu Server 20.04 installatin, with a static IP and Domain Name *ubuntu.example*.**

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

3. Add a CNAME DNS record for ***cloud.ubuntu.example*** pointing to ***ubuntu.example***

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
    ServerName cloud.ubuntu.example
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

2. Install with the following behemoth of a multi-line command:

```bash
sudo -u www-data php occ maintenance:install \
database "mysql" --database-name "nextcloud" \
--database-user "nextcloud" --database-pass "somepassword" \
--admin-user "admin" --admin-pass "anotherpassword"
```

# Post-Install configuration

Edit the file **/var/www/nextcloud/config/config.php**, adding `1 => 'cloud.ubuntu.example',` as an entry within the `trusted_domains` array.



