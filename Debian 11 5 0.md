# Debian 11.5.0 Server

## Basic Config
**Change root password (Optional)**
```sh
passwd root
```

**Change login banner**
```sh
nano /etc/issue
```
```
...
Welcome to Server SIJA
...
```

**IP Configuration**
```sh
nano /etc/network/interfaces
```
```
...
auto enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
    address 20.20.20.16/24
    dns-nameservers 20.20.20.16
...
```
```zsh
apt install resolvconf -y
systemctl restart networking
```

**Change Hostname**
```zsh
hostnamectl set-hostname lab-srv.azem.id
```

**Map host on /etc/hosts**
```zsh
nano /etc/hosts
```
```
...
20.20.20.16 lab-srv.azem.id lab-srv
...
```

**Install Curl, IPTables, net-tools, SSH**
```zsh
apt install curl iptables net-tools ssh -y
```

**Configure SSH**
```zsh
nano /etc/ssh/sshd_config
```
```
...
Port 1608
PermitRootLogin yes
...
```

## SSL Certificate
**Create Directory for Certificates**
```zsh
mkdir /etc/ssl/CA
mkdir /etc/ssl/newcerts
sh -c "echo '01' > /etc/ssl/CA/serial"
touch /etc/ssl/CA/index.txt
```

**Configure OpenSSL**
```zsh
nano /etc/ssl/openssl.cnf
```
```
...
# Line 83
dir             = /etc/ssl              # Where everything is kept
database        = $dir/CA/index.txt     # database index file.
certificate     = $dir/certs/ca.pem     # The CA certificate
serial          = $dir/CA/serial        # The current serial number
private_key     = $dir/private/cakey.pem# The private key
x509_extensions = v3_ca                 # The extensions to add to the cert

# Line 121
policy          = policy_anything
...
```

**Create an Extension File**
```zsh
nano domains.ext
```
```
...
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = www.azem.id
DNS.2 = mail.azem.id
...
```

**Create Self-signed CA Certificate**
```zsh
openssl req -new -x509 -keyout /etc/ssl/private/cakey.pem -out /etc/ssl/certs/ca.pem
```

**Generate a Certificate signed by the CA**
```zsh
openssl req -new -keyout /etc/ssl/private/srv.key -out srv.csr
openssl ca -in server.csr -config /etc/ssl/openssl.cnf -extfile domains.ext
nano /etc/ssl/certs/srv.crt
```
```
Copy and paste everything beginning with the line: -----BEGIN CERTIFICATE----- and continuing through the line: ----END CERTIFICATE----- lines to a file named srv.crt
``` 

## NTP Server
**1. Install Chrony**
```zsh
apt install chrony -y
```

**2. Configure Chrony**
```zsh
nano /etc/chrony/chrony.conf
```
```
...
allow 20.20.20.0/24
...
```

**3. Restart Chrony**
```zsh
systemctl restart chrony
```

**4. Verify and see Clients**
```zsh
chronyc sources
chronyc clients
```

**NTP Client (Windows)**
```
Open Control Panel\Clock and Region\Date and Time then open tab internet time > change settings > enter server ip, then update now
```

## DNS Server
**1. Install bind9**
```zsh
apt install bind9 bind9utils -y
```

**2. Configure BIND**
```zsh
cd /etc/bind
nano named.conf.options
```
```
...
allow-query { localhost; 20.20.20.0/24; };
allow-transfer { localhost; };
recursion yes;
...
```
```zsh
nano named.conf.local
```
```
...
zone "azem.id" {
        type master;
        file "/etc/bind/db.azem.id";
};

zone "20.20.20.in-addr.arpa" {
        type master;
        file "/etc/bind/db.20";
};
...
```

**3. Configure Zone Files**
```zsh
cp db.local db.azem.id
nano db.azem.id
```
```
...
;
; BIND data file for azem.id
;
$TTL    604800
@       IN      SOA     lab-srv.azem.id. root.azem.id. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      azem.id.
@       IN      A       20.20.20.16
@       IN      MX  10  mail.azem.id.

mail    IN      A       20.20.20.16
www     IN      A       20.20.20.16   
...
```

```zsh
cp db.127 db.20
nano db.20
```
```
...
;
; BIND reverse data file for azem.id
;
$TTL    604800
@       IN      SOA     lab-srv.azem.id. root.azem.id. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      azem.id.

16      IN      PTR     mail.azem.id.
16      IN      PTR     www.azem.id.
...
```

**4. Restart BIND**
```zsh
systemctl restart named
```

**5. Testing DNS**
```zsh
nslookup azem.id
nslookup www.azem.id
nslookup mail.azem.id
```

**DNS Client (Windows)**
```
1. (WIN+R) Run ncpa.cpl.
2. Change DNS on adapter that connect to VirtualBox.
3. Use NSLOOKUP and PING.
```

## FTP Server
**1. Install ProFTPD**
```zsh
apt install proftpd -y
```

**2. Configure ProFTPD**
- User-Login Auth
```zsh
nano /etc/proftpd/proftpd.conf
```
```xml
...
UseIPv6 off
ServerName "lab-srv.azem.id"
DefaultRoot /home/Public
...
```
- Anonymous Auth
```zsh
nano /etc/proftpd/proftpd.conf
```
```xml
...
UseIPv6 off
ServerName "lab-srv.azem.id"
DefaultRoot /home/Public
<Anonymous /home/Public>
    User azem
    UserAlias anonymous azem 
</Anonymous>
...
```
- FTP Secure
```zsh
nano /etc/proftpd/proftpd.conf
```
```
...
Include /etc/proftpd/tls.conf
...
```
```zsh
nano /etc/proftpd/tls.conf
```
```xml
...
TLSEngine               on
TLSLog                  /var/log/proftpd/tls.log
TLSProtocol             TLSv1.3
TLSRSACertificateFile                   /etc/ssl/certs/srv.crt
TLSRSACertificateKeyFile                /etc/ssl/private/srv.key
...
```

**3. Create FTP Directory**
```zsh
mkdir /home/Public
chmod 777 /home/Public
echo "TEST FILE" > /home/Public/test.txt
```

## Web Server
**1. Install Apache2**
```zsh
apt install apache2 -y
```

**2. Configure VirtualHost**
- HTTP
```zsh
cd /etc/apache2/sites-available
cp 000-default.conf www.azem.id.conf
nano www.azem.id.conf
```
```xml
...
<VirtualHost *:80>
        ServerName www.azem.id
        ServerAlias azem.id
        ServerAdmin webmaster@azem.id
        DocumentRoot /var/www/azem.id/
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
...
```

- HTTPS
```zsh
a2enmod ssl
cd /etc/apache2/sites-available
cp 000-default.conf www.azem.id.conf
nano www.azem.id.conf
```
```xml
...
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName www.azem.id
        ServerAlias azem.id
        ServerAdmin webmaster@azem.id
        DocumentRoot /var/www/azem.id/
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        SSLEngine on
        SSLCertificateFile      /etc/ssl/certs/srv.crt
        SSLCertificateKeyFile   /etc/ssl/private/srv.key
    </VirtualHost>
</IfModule>
...
```
```zsh
cp /etc/ssl/certs/ca.pem /var/www/azem.id
```

**3. Activate VirtualHost**
```zsh
a2ensite www.azem.id.conf
systemctl reload apache2
```

**4. Create a page**
```zsh
mkdir /var/www/azem.id/
```
```zsh
nano /var/www/azem.id/index.html
```
```html
...
<html>
    <head>
        <title>LMS SIJA</title>
    </head>
    <body>
        <div style="width: 100%; font-size: 40px; font-weight: bold; text-align: center;">
            Selamat datang di Web Seleksi Internal LKS ITNSA 2023
        </div>
    </body>
</html>
...
```

**5. Restart Apache2**
```zsh
systemctl restart apache2
```

**Testing Web**
- Linux
```zsh
curl http://www.azem.id
curl https://www.azem.id
```
- Windows
```
[HTTP] Access Web with URL http://www.azem.id
[HTTPS] Download CA File from URL http://www.azem.id/ca.pem
Install/Import the Certificate
Access Web with URL https://www.azem.id
```

## Database Server
**1. Install MariaDB Server**
```zsh
apt install mariadb-server -y
systemctl start mariadb
systemctl enable mariadb
```

**2. Initial Setting**
```zsh
mysql_secure_installation
```
```
n, n, y all
```

**3. Connect to MariaDB**
```zsh
mysql -u root -p
```
```
show grants for root@localhost;
```
```
show databases;
```
```
select user,host,password from mysql.user;
```

**4. Install phpMyAdmin**
```zsh
apt install php phpmyadmin -y
dpkg-reconfigure phpmyadmin
```
```
use user root@localhost
db administrator : root
```

## Mail Server
**1. Install Postfix and Dovecot**
```zsh
apt install postfix dovecot-imapd dovecot-pop3d -y
```

**2. Configure SMTP and IMAP Server**
```zsh
dpkg-reconfigure postfix
```
```
domain : azem.id
network add 10.20.20.0/24
```
```zsh
nano /etc/postfix/main.cf
```
```
home_mailbox = Maildir/ # Add
```
```zsh
nano /etc/dovecot/dovecot.conf
```
```xml
...
listen = *, ::  # Line 30
...
```
```zsh
nano /etc/dovecot/conf.d/10-auth.conf
```
```xml
...
disable_plaintext_auth = no     # Line 10
...
```
```zsh
nano /etc/dovecot/conf.d/10-mail.conf
```
```xml
...
mail_location = maildir:~/Maildir
...
```
```zsh
maildirmake.dovecot /etc/skel/Maildir
```
```zsh
systemctl restart postfix dovecot
```

**3. Add Mail User Accounts**
```zsh
adduser zaidansatu
adduser zaidandua
```

**4. Create database for Roundcube**
```
Open azem.id/phpMyAdmin on browser
Login with user root
Create new User accounts
- Username : roundcube
- Hostname : localhost
- Password : mlzdn16
Database for user accounts
- check all
Global Privileges
- check all
```

**5. Install Roundcube**
```zsh
apt install roundcube -y
nano /etc/roundcube/config.inc.php 
```
```php
...
$config['default_host'] = 'lab-srv.azem.id';      # Line 36
$config['smtp_server'] = 'lab-srv.azem.id';       # Line 50
$config['smtp_port'] = 25;                    # Line 53
$config['smtp_user'] = '';                    # Line 57
$config['smtp_pass'] = '';                    # Line 61
$config['product_name'] = 'Webmail SIJA';     # Line 68
...
```
```zsh
cd /etc/apache2/sites-available
cp 000-default.conf roundcube.conf
nano roundcube.conf
```
```xml
...
<VirtualHost *:80>
        ServerName mail.azem.id
        ServerAdmin webmaster@azem.id
        DocumentRoot /var/lib/roundcube/public_html
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName mail.azem.id
        ServerAdmin webmaster@azem.id
        DocumentRoot /var/lib/roundcube/public_html
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        SSLEngine on
        SSLCertificateFile      /etc/ssl/certs/srv.crt
        SSLCertificateKeyFile   /etc/ssl/private/srv.key
    </VirtualHost>
</IfModule>
...
```
```zsh
a2ensite roundcube.conf
systemctl restart apache2 postfix dovecot
```

**Testing Webmail**
```
Access https://mail.azem.id
Login with User that has been created before
Testing Sending Mail
```
