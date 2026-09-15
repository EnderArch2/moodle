# Moodle VPS Deployment

This guide deploys Moodle from this repository on a Debian or Ubuntu VPS with:

- Apache2
- MariaDB
- BIND9
- A self-signed HTTPS certificate

This guide uses `giovanni.net` for the main domain and `elearning.giovanni.net` for Moodle. Replace every value shown as `CHANGE_ME` before using the commands.

## 1. Deployment Layout

Keep Moodle's uploaded files outside the web root:

```text
/var/www/moodle/       Moodle source code
/var/moodledata/       Moodle uploads and runtime data
/etc/apache2/sites-available/moodle.conf
/etc/bind/db.giovanni.net
/etc/ssl/private/moodle.key
/etc/ssl/certs/moodle.crt
```

The Apache document root must be the repository's `public/` directory:

```text
/var/www/moodle/public
```

Never commit the server's `config.php`, `moodledata`, database dumps, or private keys.

## 2. Check the Operating System and PHP Version

This repository's current Moodle source requires PHP 8.3 or newer. Ubuntu 22.04 normally provides PHP 8.1, which is not compatible with this Moodle branch. Do not continue with PHP 8.1.

For a fresh VM, the cleanest option is Ubuntu 24.04 LTS, which provides PHP 8.3 from the standard repositories. If you must keep Ubuntu 22.04, use the PHP 8.3 packages from the maintained Ondrej PHP PPA:

```bash
sudo apt update
sudo apt install -y software-properties-common ca-certificates lsb-release
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```

## 3. Install Packages

Update the VPS and install the required services and common Moodle dependencies:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git apache2 bind9 openssl mariadb-server \
    php8.3 php8.3-cli php8.3-fpm php8.3-mysql php8.3-curl \
    php8.3-gd php8.3-intl php8.3-mbstring php8.3-xml php8.3-zip \
    php8.3-soap php8.3-bcmath php8.3-opcache unzip
```

Check the installed versions. This repository's Moodle source requires PHP 8.3 or newer:

```bash
php8.3 -v
mariadb --version
```

Make PHP 8.3 the command-line default if another PHP version is already installed:

```bash
sudo update-alternatives --install /usr/bin/php php /usr/bin/php8.3 83
sudo update-alternatives --set php /usr/bin/php8.3
```

Confirm that the active PHP version is correct:

```bash
php -v
```

The output must show PHP 8.3 or newer. If you choose Ubuntu 24.04 instead, install the equivalent unversioned `php-*` packages from its standard repositories, then verify that `php -v` reports 8.3 or newer.

Enable and start the services:

```bash
sudo systemctl enable --now mariadb apache2 bind9 php8.3-fpm
```

The rest of this guide assumes PHP 8.3 and the socket `/run/php/php8.3-fpm.sock`.

## 4. MariaDB Database

Run the MariaDB hardening wizard:

```bash
sudo mariadb-secure-installation
```

Recommended answers for a new server are:

- Set a MariaDB root password if requested.
- Remove anonymous users.
- Disable remote root login.
- Remove the test database.
- Reload privilege tables.

Create a dedicated database and user. Do not use the MariaDB `root` account for Moodle.

```bash
sudo mariadb
```

```sql
CREATE DATABASE moodle
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

CREATE USER 'moodleuser'@'localhost'
    IDENTIFIED BY 'CHANGE_ME_LONG_RANDOM_DATABASE_PASSWORD';

GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

Use a long, random password. Store it securely; it will be placed in the server-only Moodle `config.php`.

Verify the credentials before continuing:

```bash
mariadb -u moodleuser -p moodle -e 'SELECT 1;'
```

## 5. Install Moodle Source

Clone the GitHub repository into `/var/www/moodle`:

```bash
sudo git clone https://github.com/CHANGE_ME_GITHUB_USER/CHANGE_ME_REPOSITORY.git /var/www/moodle
```

Set ownership of the source tree to the web-service account:

```bash
sudo chown -R www-data:www-data /var/www/moodle
```

Create the Moodle data directory outside the Apache document root:

```bash
sudo install -d -o www-data -g www-data -m 2770 /var/moodledata
```

## 6. Create Moodle Configuration

The repository includes `config-dist.php` as a template. Create the real configuration file at the repository root on the VPS:

```bash
sudo cp /var/www/moodle/config-dist.php /var/www/moodle/config.php
sudo nano /var/www/moodle/config.php
```

Set the database and site values. The important values should look like this:

```php
$CFG->dbtype    = 'mariadb';
$CFG->dblibrary  = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'CHANGE_ME_LONG_RANDOM_DATABASE_PASSWORD';
$CFG->prefix    = 'mdl_';

$CFG->dboptions = [
    'dbpersist' => false,
    'dbsocket' => false,
    'dbport' => '',
    'dbcollation' => 'utf8mb4_unicode_ci',
];

$CFG->wwwroot = 'https://elearning.giovanni.net';
$CFG->dataroot = '/var/moodledata';
$CFG->directorypermissions = 02770;
```

The file must end with Moodle's existing bootstrap line:

```php
require_once(__DIR__ . '/lib/setup.php');
```

Protect the configuration file:

```bash
sudo chown root:www-data /var/www/moodle/config.php
sudo chmod 640 /var/www/moodle/config.php
```

The root `.gitignore` excludes this file, so database credentials are not pushed to GitHub.

## 7. BIND9 DNS

Use `giovanni.net` for the Apache default site and `elearning.giovanni.net` for Moodle. Both names must resolve to the VPS. For a public site, create these records at your domain registrar or DNS provider. Running BIND9 on the VPS does not automatically publish records to the Internet unless the domain is delegated to your nameserver.

The following is an example BIND9 zone. Replace the VPS address:

Create `/etc/bind/db.giovanni.net`:

```dns
$TTL 86400
@   IN  SOA ns1.giovanni.net. admin.giovanni.net. (
        2026091501
        3600
        900
        604800
        86400
)

    IN  NS  ns1.giovanni.net.

ns1       IN  A   192.168.1.50
@         IN  A   192.168.1.50
elearning IN  A   192.168.1.50
```

Add the zone to `/etc/bind/named.conf.local`:

```bind
zone "giovanni.net" {
    type master;
    file "/etc/bind/db.giovanni.net";
};
```

Validate and reload BIND9:

```bash
sudo named-checkzone giovanni.net /etc/bind/db.giovanni.net
sudo named-checkconf
sudo systemctl reload bind9
```

Configure the client computers or local DHCP server to use this BIND9 server for DNS. Test resolution:

```bash
dig @192.168.1.50 giovanni.net
dig @192.168.1.50 elearning.giovanni.net
```

Use `giovanni.net` for the default Apache site and `elearning.giovanni.net` in the Moodle Apache virtual host, certificate SAN, and `$CFG->wwwroot`.

## 8. Self-Signed TLS Certificate

Generate one certificate covering both hostnames. This allows HTTPS access to the Apache default page at `giovanni.net` and Moodle at `elearning.giovanni.net`:

```bash
sudo openssl req -x509 -nodes -newkey rsa:4096 \
    -keyout /etc/ssl/private/moodle.key \
    -out /etc/ssl/certs/moodle.crt \
    -days 825 \
    -subj "/C=US/ST=State/L=City/O=Giovanni/CN=giovanni.net" \
    -addext "subjectAltName=DNS:giovanni.net,DNS:elearning.giovanni.net"
```

Protect the private key:

```bash
sudo chmod 600 /etc/ssl/private/moodle.key
```

Browsers will warn about a self-signed certificate until the certificate is installed as trusted on each client device. For a public Internet site, Let's Encrypt is usually preferable because normal browsers already trust it.

## 9. Apache2 Virtual Host

Enable the required Apache modules:

```bash
sudo a2enmod rewrite ssl headers proxy_fcgi setenvif
```

Leave Apache's default site enabled so `giovanni.net` continues to serve `/var/www/html/index.html`. Create `/etc/apache2/sites-available/moodle.conf` for the Moodle subdomain:

```apache
<VirtualHost *:80>
    ServerName elearning.giovanni.net
    Redirect permanent / https://elearning.giovanni.net/
</VirtualHost>

<VirtualHost *:443>
    ServerName elearning.giovanni.net

    DocumentRoot /var/www/moodle/public

    <Directory /var/www/moodle/public>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
        DirectoryIndex index.php
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/moodle.crt
    SSLCertificateKeyFile /etc/ssl/private/moodle.key

    Header always set X-Content-Type-Options "nosniff"

    ErrorLog ${APACHE_LOG_DIR}/moodle-error.log
    CustomLog ${APACHE_LOG_DIR}/moodle-access.log combined
</VirtualHost>
```

If you also want `https://giovanni.net` to serve the default `/var/www/html/index.html`, create `/etc/apache2/sites-available/giovanni-ssl.conf`:

```apache
<VirtualHost *:443>
    ServerName giovanni.net

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.html
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/moodle.crt
    SSLCertificateKeyFile /etc/ssl/private/moodle.key

    ErrorLog ${APACHE_LOG_DIR}/giovanni-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/giovanni-ssl-access.log combined
</VirtualHost>
```

Replace only the PHP-FPM socket if needed. Enable the Moodle site and the root HTTPS site. Do not disable `000-default.conf`:

```bash
sudo a2ensite moodle.conf
sudo a2ensite giovanni-ssl.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## 10. Complete Installation

### Troubleshooting: `dataroot` Error

If Moodle reports:

```text
$CFG->dataroot is not configured properly, directory does not exist or is not accessible
```

create the data directory and make it accessible to the web-service account:

```bash
sudo install -d -o www-data -g www-data -m 2770 /var/moodledata
sudo chmod 2770 /var/moodledata
```

Confirm that the `www-data` user can enter the directory and write to it:

```bash
sudo -u www-data test -r /var/moodledata && echo "readable"
sudo -u www-data test -w /var/moodledata && echo "writable"
sudo -u www-data touch /var/moodledata/.moodle-write-test
sudo -u www-data rm /var/moodledata/.moodle-write-test
```

All three checks must succeed. Also verify that the parent directory permits traversal:

```bash
namei -l /var/moodledata
```

The server-only `/var/www/moodle/config.php` must contain the exact absolute path:

```php
$CFG->dataroot = '/var/moodledata';
```

Check the configured value without printing the database password:

```bash
sudo grep -n "dataroot" /var/www/moodle/config.php
```

If the directory was created under the wrong path, remove only the empty incorrect directory and use `/var/moodledata`. Never put `moodledata` inside `/var/www/moodle/public`, because Apache could expose uploaded files.

If a database or administrator password has been exposed in a screenshot, terminal recording, chat, or log, do not reuse it. Change the MariaDB password before retrying:

```bash
sudo mariadb
```

```sql
ALTER USER 'moodleuser'@'localhost'
    IDENTIFIED BY 'CHANGE_ME_NEW_LONG_RANDOM_DATABASE_PASSWORD';
FLUSH PRIVILEGES;
EXIT;
```

Update `$CFG->dbpass` in `/var/www/moodle/config.php` to the new database password. Use a new, unrelated value for `--adminpass` when rerunning the installer. If installation failed before the Moodle tables and administrator were created, there is no existing Moodle administrator password to change.

Open the HTTPS hostname in a browser:

```text
https://elearning.giovanni.net
```

Accept or install the self-signed certificate warning, then complete Moodle's web installer. Select MariaDB when prompted and use:

- Database server: `localhost`
- Database name: `moodle`
- Database user: `moodleuser`
- Database password: the password created above
- Database tables prefix: `mdl_`
- Data directory: `/var/moodledata`

Alternatively, install from the command line:

```bash
sudo -u www-data php /var/www/moodle/admin/cli/install.php \
    --wwwroot=https://elearning.giovanni.net \
    --dataroot=/var/moodledata \
    --dbtype=mariadb \
    --dbhost=localhost \
    --dbname=moodle \
    --dbuser=moodleuser \
    --dbpass='CHANGE_ME_LONG_RANDOM_DATABASE_PASSWORD' \
    --fullname='My Moodle Site' \
    --shortname='Moodle' \
    --adminuser=admin \
    --adminpass='CHANGE_ME_STRONG_ADMIN_PASSWORD' \
    --adminemail=admin@giovanni.net \
    --agree-license
```

## 11. Cron

Moodle requires its cron task to run regularly. Add a cron entry for the web-service user:

```bash
sudo crontab -u www-data -e
```

Add:

```cron
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1
```

## 12. Firewall and Checks

If UFW is enabled, allow SSH and web traffic:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Apache Full'
sudo ufw enable
```

Check the main services:

```bash
sudo systemctl status mariadb apache2 bind9 php8.3-fpm
sudo apache2ctl configtest
sudo journalctl -u apache2 -n 50 --no-pager
sudo tail -n 50 /var/log/apache2/moodle-error.log
```

## 13. Updating Moodle

Back up the database and `moodledata` before upgrades. Put Moodle into maintenance mode, pull the intended Git commit, then run the upgrade:

```bash
sudo -u www-data php /var/www/moodle/admin/cli/maintenance.php --enable
sudo -u www-data git -C /var/www/moodle fetch origin
sudo -u www-data git -C /var/www/moodle checkout main
sudo -u www-data git -C /var/www/moodle pull --ff-only origin main
sudo -u www-data php /var/www/moodle/admin/cli/upgrade.php --non-interactive
sudo -u www-data php /var/www/moodle/admin/cli/maintenance.php --disable
```

Do not update production by blindly tracking an untested branch. Test Moodle upgrades on a copy first.
