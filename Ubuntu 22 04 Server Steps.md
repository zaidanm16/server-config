# Ubuntu 22.04 Server

## Basic Config
**Configure Sudo**
```zsh
sudo usermod -aG sudo azem
sudo visudo
```
```
...
azem	ALL=(ALL) NOPASSWD:ALL
...
```

**Activate and change root password**
```zsh
sudo passwd root
```

**Change login banner**
```zsh
sudo nano /etc/issue
```
```
...
Welcome to SIJA Server
...
```

**IP Configuration**
```zsh
cd /etc/netplan
sudo mv 00-installer-config.yaml 01-netcfg.yaml
sudo nano 01-netcfg.yaml
```
```
...
network:
  ethernets:
    ens3:
      match:
        macaddress: 08:00:27:53:11:f6
      dhcp4: true
      set-name: ens3
    ens4:
      match:
        macaddress: 08:00:27:3e:1e:b1
      dhcp4: false
      addresses:
        - 10.20.20.16/24
      nameservers:
        addresses: [10.20.20.16]
      set-name: ens4
  version: 2
...
```
```zsh
sudo netplan apply
```

**Configure Resolvconf**
```zsh
sudo apt install resolvconf -y
sudo unlink /etc/resolv.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

**Change Hostname**
```zsh
sudo hostnamectl hostname srv.sija.sch.id
```

**Map host on /etc/hosts**
```zsh
sudo nano /etc/hosts
```
```
...
10.20.20.16 srv.sija.sch.id srv
...
```

**Install Curl, IPTables, net-tools, SSH**
```zsh
sudo apt install curl iptables net-tools ssh -y
```

**Configure SSH**
```zsh
sudo nano /etc/ssh/sshd_config
```
```
...
Port 1608
...
```

## SSL Certificate
**Create Directory for Certificates**
```zsh
sudo mkdir /etc/ssl/CA
sudo mkdir /etc/ssl/newcerts
sudo sh -c "echo '01' > /etc/ssl/CA/serial"
sudo touch /etc/ssl/CA/index.txt
```

**Configure OpenSSL**
```zsh
sudo nano /etc/ssl/openssl.cnf
```
```
...
# Line 83
dir             = /etc/ssl              # Where everything is kept
database        = $dir/CA/index.txt     # database index file.
certificate     = $dir/certs/cacert.pem # The CA certificate
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
DNS.1 = www.sija.sch.id
DNS.2 = lms.sija.sch.id
...
```

**Create Self-signed CA Certificate**
```zsh
sudo openssl req -new -x509 -keyout cakey.pem -out cacert.pem
```

**Move Certificate**
```zsh
sudo mv cakey.pem /etc/ssl/private/
sudo mv cacert.pem /etc/ssl/certs/
```

**Generate a Certificate signed by the CA**
```zsh
sudo openssl req -new -keyout server.key -out server.csr
sudo openssl ca -in server.csr -config /etc/ssl/openssl.cnf -extfile domains.ext
```
```
Copy and paste everything beginning with the line: -----BEGIN CERTIFICATE----- and continuing through the line: ----END CERTIFICATE----- lines to a file named server.crt
```
```zsh
sudo mv server.crt /etc/ssl/certs/
sudo mv server.key /etc/ssl/private/
```

## NTP Server
**1. Install Chrony**
```zsh
sudo apt install chrony -y
```

**2. Configure Chrony**
```zsh
sudo nano /etc/chrony/chrony.conf
```
```
...
allow 10.20.20.0/24
...
```

**3. Restart Chrony**
```zsh
sudo systemctl restart chrony
```

**4. Verify and see Clients**
```zsh
sudo chronyc sources
sudo chronyc clients
```

**NTP Client (Windows)**
```
Open Control Panel\Clock and Region\Date and Time then open tab internet time > change settings > enter server ip, then update now
```

## DNS Server
**1. Install bind9**
```zsh
sudo apt install bind9 bind9utils -y
```

**2. Configure BIND**
```zsh
cd /etc/bind
sudo nano named.conf.options
```
```
...
allow-query { localhost; 10.20.20.0/24; };
allow-transfer { localhost; };
recursion yes;
...
```
```zsh
sudo nano named.conf.local
```
```
...
zone "sija.sch.id" {
        type master;
        file "/etc/bind/db.sija.sch.id";
};

zone "20.20.10.in-addr.arpa" {
        type master;
        file "/etc/bind/db.10";
};
...
```

**3. Configure Zone Files**
```zsh
sudo cp db.local db.sija.sch.id
sudo nano db.sija.sch.id
```
```
...
;
; BIND data file for sija.sch.id
;
$TTL    604800
@       IN      SOA     srv.sija.sch.id. root.sija.sch.id. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      sija.sch.id.
@       IN      A       10.20.20.16
@       IN      MX  10  mail.sija.sch.id.

srv     IN      A       10.20.20.16
mail    IN      A       10.20.20.16

www     IN      CNAME   srv.sija.sch.id.
...
```

```zsh
sudo cp db.127 db.10
sudo nano db.10
```
```
...
;
; BIND reverse data file for sija.sch.id
;
$TTL    604800
@       IN      SOA     srv.sija.sch.id. root.sija.sch.id. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      sija.sch.id.

16      IN      PTR     srv.sija.sch.id.
16      IN      PTR     mail.sija.sch.id.
...
```

**4. Restart BIND**
```zsh
sudo systemctl restart named
```

**5. Testing DNS**
```zsh
nslookup sija.sch.id
nslookup srv.sija.sch.id
nslookup www.sija.sch.id
nslookup mail.sija.sch.id
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
sudo apt install vsftpd -y
```

**2. Configure ProFTPD**
- User-Login Auth
```zsh
sudo nano /etc/proftpd/proftpd.conf
```
```
...
UseIPv6 off
ServerName "srv.sija.sch.id"
DefaultRoot /home/Public
...
```
- Anonymous Auth
```zsh
sudo nano /etc/proftpd/proftpd.conf
```
```
...
UseIPv6 off
ServerName "srv.sija.sch.id"
DefaultRoot /home/Public
<Anonymous /home/Public>
    User azem
    UserAlias anonymous azem 
</Anonymous>
...
```
- FTP Secure
```zsh
sudo nano /etc/proftpd/proftpd.conf
```
```
...
Include /etc/proftpd/tls.conf
...
```
```zsh
sudo nano /etc/proftpd/tls.conf
```
```
...
TLSEngine               on
TLSLog                  /var/log/proftpd/tls.log
TLSProtocol             TLSv1.3
TLSRSACertificateFile                   /etc/ssl/certs/server.crt
TLSRSACertificateKeyFile                /etc/ssl/private/server.key
...
```

**3. Create FTP Directory**
```zsh
sudo mkdir /home/Public
sudo chmod 777 /home/Public
echo "TEST FILE" > /home/Public/test.txt
```

## Web Server
**1. Install Apache2**
```zsh
sudo apt install apache2 -y
```

**2. Configure VirtualHost**
- HTTP
```zsh
cd /etc/apache2/sites-available
sudo cp 000-default.conf www.sija.sch.id.conf
sudo nano www.sija.sch.id.conf
```
```
...
<VirtualHost *:80>
        ServerName www.sija.sch.id
        ServerAlias sija.sch.id
        ServerAdmin webmaster@sija.sch.id
        DocumentRoot /var/www/sija.sch.id/
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
...
```

- HTTPS
```zsh
sudo a2enmod ssl
cd /etc/apache2/sites-available
sudo cp 000-default.conf www.sija.sch.id.conf
sudo nano www.sija.sch.id.conf
```
```
...
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName www.sija.sch.id
        ServerAlias sija.sch.id
        ServerAdmin webmaster@sija.sch.id
        DocumentRoot /var/www/sija.sch.id/
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        SSLEngine on
        SSLCertificateFile      /etc/ssl/certs/server.crt
        SSLCertificateKeyFile   /etc/ssl/private/server.key
    </VirtualHost>
</IfModule>
...
```
```zsh
sudo cp /etc/ssl/certs/cacert.pem /var/www/sija.sch.id
```

**3. Activate VirtualHost**
```zsh
sudo a2ensite www.sija.sch.id.conf
sudo systemctl reload apache2
```

**4. Create a page**
```zsh
sudo mkdir /var/www/sija.sch.id/
```
```zsh
sudo nano /var/www/sija.sch.id/index.html
```
```
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
sudo systemctl restart apache2
```

**Testing Web**
- Linux
```zsh
curl http://www.sija.sch.id
curl https://www.sija.sch.id
```
- Windows
```
[HTTP] Access Web with URL http://www.sija.sch.id
[HTTPS] Download CA File from URL http://www.sija.sch.id/cacert.pem
Install/Import the Certificate
Access Web with URL https://www.sija.sch.id
```

## Database Server
**1. Install MariaDB Server**
```zsh
sudo apt install mariadb-server -y
```

**2. Initial Setting**
```zsh
sudo mysql_secure_installation
```
```
n, n, y all
```

**3. Connect to MariaDB**
```zsh
sudo mysql -u root -p
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
sudo apt install php8.1 php8.1-cgi libapache2-mod-php8.1 php8.1-common php-pear php8.1-mbstring phpmyadmin -y
```

**5. Delete All data from MariaDB dan Initialize it**
```zsh
sudo systemctl stop mariadb
sudo rm -rf /var/lib/mysql/*
sudo mysql_install_db --datadir=/var/lib/mysql --user=mysql
sudo systemctl start mariadb
```

**6. Create new SuperUser account for phpMyadmin**
```
CREATE USER 'mele'@localhost IDENTIFIED BY 'mele16';
SELECT User FROM mysql.user;
GRANT ALL PRIVILEGES ON *.* TO 'mele'@localhost IDENTIFIED BY 'mele16';
FLUSH PRIVILEGES;
```

## Mail Server dan Webmail
**1. Install Postfix to configure SMTP Server**
```zsh
sudo apt install postfix sasl2-bin -y
```
```
select [No Configuration]
```
```zsh
sudo cp /usr/share/postfix/main.cf.dist /etc/postfix/main.cf
sudo nano /etc/postfix/main.cf
```
```
...
mail_owner = postfix            # Line 78
myhostname = srv.sija.sch.id    # Line 94
mydomain = sija.sch.id          # Line 102
myorigin = $mydomain            # Line 123
inet_interfaces = all           # Line 137
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain  # Line 185
local_recipient_maps = unix:passwd.byname $alias_maps    # Line 228
mynetworks_style = subnet       # Line 270
mynetworks = 127.0.0.0/8, 10.20.20.0/24  # Line 287
alias_maps = hash:/etc/aliases  # Line 407
alias_database = hash:/etc/aliases  # Line 418
home_mailbox = Maildir/         # Line 440
sendmail_path = /usr/sbin/postfix   # Line 650
newaliases_path = /usr/bin/newaliases  # Line 655
mailq_path = /usr/bin/mailq     # Line 660
setgid_group = postdrop         # Line 666
#html_directory =
#manpage_directory =
#sample_directory =
#readme_directory =

disable_vrfy_command = yes

smtpd_helo_required = yes

message_size_limit = 10240000

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination, permit_sasl_authenticated, reject
smtpd_client_restrictions = permit_mynetworks, reject_unknown_client_hostname, permit
smtpd_sender_restrictions = permit_mynetworks, reject_unknown_sender_domain, reject_non_fqdn_sender
smtpd_helo_restrictions = permit_mynetworks, reject_unknown_hostname, reject_non_fqdn_hostname, reject_invalid_hostname, permit

smtpd_use_tls = yes
smtp_tls_mandatory_protocols = !SSLv2, !SSLv3
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3
smtpd_tls_cert_file = /etc/ssl/certs/server.crt
smtpd_tls_key_file = /etc/ssl/private/server.key
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
...
```

```zsh
sudo nano /etc/postfix/master.cf
```
```
...
# line 19, 20, 22 : uncomment
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes

# line 33-36 : uncomment
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
...
```

```zsh
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```
```
ssl = yes    # Line 6

ssl_cert = </etc/ssl/certs/server.crt   # Line 12
ssl_key = </etc/ssl/private/server.key  # Line 13
```

```zsh
sudo systemctl restart postfix
```

**2. Install Dovecot to configure POP/IMAP server**
```zsh
sudo apt install dovecot-core dovecot-pop3d dovecot-imapd -y
sudo nano /etc/dovecot/dovecot.conf
```
```
...
listen = *, ::  # Line 30
...
```
```zsh
sudo nano /etc/dovecot/conf.d/10-auth.conf
```
```
...
disable_plaintext_auth = no     # Line 10
auth_mechanisms = plain login   # Line 100
...
```
```zsh
sudo nano /etc/dovecot/conf.d/10-mail.conf
```
```
...
mail_location = maildir:~/Maildir
...
```
```zsh
sudo nano /etc/dovecot/conf.d/10-master.conf
```
```
...
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {   # Line 107-109
    mode = 0666
    user = postfix
    group = postfix
  }
...
```
```zsh
sudo systemctl restart dovecot
```

**3. Add Mail User Accounts**
```zsh
sudo apt install mailutils -y
sudo echo 'export MAIL=$HOME/Maildir/' >> /etc/profile.d/mail.sh
sudo useradd -m zaidansatu
sudo useradd -m zaidandua
sudo passwd zaidansatu
sudo passwd zaidandua
```

**4. Install Roundcube**
```zsh
sudo mysql -u root p
```
```
create database roundcube;
grant all privileges on roundcube.* to roundcube@'localhost' identified by 'pass16';
flush privileges; 
exit 
```
```zsh
sudo apt install roundcube roundcube-mysql -y
```
```
select [No] to set manually later
```
```zsh
cd /usr/share/dbconfig-common/data/roundcube/install
sudo mysql -u roundcube -D roundcube -p < mysql
cd
sudo nano /etc/roundcube/debian-db.php
```
```
...
$dbuser='roundcube';
$dbpass='pass16';
$basepath='';
$dbname='roundcube';
$dbserver='localhost';
$dbport='3306';
$dbtype='mysql';
...
```
```zsh
sudo nano /etc/roundcube/config.inc.php
```
```
...
$config['default_host'] = 'tls://mail.srv.world';       # Line 36
$config['smtp_server'] = 'tls://mail.srv.world';        # Line 50
$config['smtp_port'] = 587;                             # Line 53
$config['smtp_user'] = '%u';                            # Line 57
$config['smtp_pass'] = '%p';                            # Line 61
$config['product_name'] = 'SIJA Webmail';               # Line 68
$config['default_port'] = 143;
$config['smtp_auth_type'] = 'LOGIN';
$config['smtp_helo_host'] = 'srv.sija.sch.id';
$config['mail_domain'] = 'sija.sch.id';
$config['useragent'] = 'SIJA Webmail';
$config['imap_conn_options'] = array(
  'ssl'         => array(
    'verify_peer' => true,
    'CN_match' => 'sija.sch.id',
    'allow_self_signed' => true,
    'ciphers' => 'HIGH:!SSLv2:!SSLv3',
  ),
);
$config['smtp_conn_options'] = array(
  'ssl'         => array(
    'verify_peer' => true,
    'CN_match' => 'sija.sch.id',
    'allow_self_signed' => true,
    'ciphers' => 'HIGH:!SSLv2:!SSLv3',
  ),
);
...
```
```zsh
sudo nano /etc/apache2/conf-enabled/roundcube.conf
```
```
...
Alias /roundcube /var/lib/roundcube/public_html
...
```

**5. Restart Apache2**
```zsh
sudo systemctl restart apache2
```

**Testing**
```
Access to [https://sija.sch.id/roundcube/]
```

