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

**Install Curl, IPTables, net-tools, ssh**
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
1. User-Login Auth
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
2. Anonymous Auth
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
3. FTP Secure
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
TLSRSACertificateFile                   /etc/ssl/private/proftpd.pem
TLSRSACertificateKeyFile                /etc/ssl/private/proftpd.pem
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
1. HTTP
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

2. HTTPS
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
        SSLCertificateFile      /etc/ssl/certs/ssl-cert-snakeoil.pem
        SSLCertificateKeyFile   /etc/ssl/private/ssl-cert-snakeoil.key
    </VirtualHost>
</IfModule>
...
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
1. Linux
```zsh
curl http://www.sija.sch.id
curl https://www.sija.sch.id
```
2. Windows
```
[HTTP] Access Web with URL http://www.sija.sch.id
[HTTPS] Download CA File from URL http://www.sija.sch.id/cacert.pem
Install/Import the Certificate
Access Web with URL https://www.sija.sch.id
```

