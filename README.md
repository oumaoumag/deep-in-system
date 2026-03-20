# deep-in-system

A hands-on Linux server administration project involving the setup and configuration of an Ubuntu Server 24.04 LTS virtual machine. The server is configured with custom disk partitioning, static networking, SSH security, user management, FTP, MySQL, WordPress, and automated backups.

---

## Table of Contents

- [Virtual Machine Setup](#virtual-machine-setup)
- [Disk Partitioning](#disk-partitioning)
- [Network Configuration](#network-configuration)
- [SSH Security](#ssh-security)
- [Firewall Configuration](#firewall-configuration)
- [User Management](#user-management)
- [FTP Server](#ftp-server)
- [MySQL Database](#mysql-database)
- [WordPress](#wordpress)
- [Backup](#backup)
- [Open Ports Justification](#open-ports-justification)

---

## Virtual Machine Setup

**Hypervisor:** Oracle VirtualBox

**OS:** Ubuntu Server 24.04.4 LTS (Noble Numbat)

**Hardware configuration:**
- RAM: 2048 MB
- CPUs: 2
- Disk: 30 GB (dynamically allocated)
- Network: Bridged Adapter

**Hostname:** `oumaouma-host`

Set with:
```bash
hostnamectl set-hostname oumaouma-host
```

**Username:** `oumaouma`

---

## Disk Partitioning

The 30GB disk was divided into 4 partitions during installation using the Ubuntu Server custom storage layout:

| Partition | Size | Format | Mount Point | Purpose |
|---|---|---|---|---|
| partition 2 | 4GB | swap | none | Virtual memory overflow when RAM is full |
| partition 3 | 15GB | ext4 | / | Root partition — OS and all programs |
| partition 4 | 5GB | ext4 | /home | User home directories |
| partition 5 | 6GB | ext4 | /backup | Dedicated backup storage |

**ext4** (Fourth Extended Filesystem) is the standard Linux filesystem format — reliable, journaled, and widely supported.

**swap** is a special partition that acts as overflow memory. When physical RAM runs out the OS temporarily moves less-used data to swap space.

Verify partitions with:
```bash
lsblk
df -h
```

---

## Network Configuration

A static private IP address is configured so the server always has the same address on the network.

**Interface:** `enp0s3`

**Static IP:** `192.168.1.168/24`

**Gateway:** `192.168.1.1`

**DNS:** `8.8.8.8`, `8.8.4.4`

Configuration file: `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.1.168/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Apply configuration:
```bash
sudo netplan apply
```

Test internet connectivity:
```bash
ping -c 5 google.com
```

**Why static IP:** A server must have a fixed address so other devices and services can reliably connect to it. DHCP (Dynamic Host Configuration Protocol) assigns addresses automatically but they can change on reboot — unacceptable for a server.

---

## SSH Security

SSH (Secure Shell) allows remote command-line access to the server over an encrypted connection.

Configuration file: `/etc/ssh/sshd_config`

Changes made:

```
Port 2222
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
```

Apply changes:
```bash
sudo systemctl restart ssh
```

**Why port 2222:** The default SSH port 22 is constantly targeted by automated attacks. Changing to a non-standard port reduces exposure to automated brute force attempts.

**Why PermitRootLogin no:** The root account has unrestricted access to everything. Allowing remote root login means a single compromised password gives an attacker complete control. Using sudo instead provides the same capabilities with better auditing and access control.

Connect via SSH:
```bash
ssh oumouma@192.168.1.168 -p 2222
```

---

## Firewall Configuration

UFW (Uncomplicated Firewall) controls incoming and outgoing network traffic. By default all incoming connections are blocked and only explicitly allowed ports are open.

```bash
sudo ufw enable
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 21/tcp
```

Check status:
```bash
sudo ufw status
```

---

## Open Ports Justification

| Port | Protocol | Service | Justification |
|---|---|---|---|
| 2222 | TCP | SSH | Remote server administration — changed from default 22 for security |
| 80 | TCP | HTTP | WordPress website access |
| 443 | TCP | HTTPS | WordPress secure website access |
| 21 | TCP | FTP | File transfer for nami user to access /backup |

All other ports are closed. No unnecessary services are exposed.

---

## User Management

### User: luffy

- **Home directory:** `/home/luffy`
- **Sudo access:** yes
- **Authentication:** SSH public key

Create user and grant sudo:
```bash
sudo adduser luffy
sudo usermod -aG sudo luffy
```

Generate SSH key pair as luffy:
```bash
ssh-keygen -t ed25519
```

Set up authorized keys:
```bash
cp ~/.ssh/id_ed25519.pub ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

**Why public key authentication:** SSH keys are cryptographically stronger than passwords. A key pair consists of a private key (kept secret on the client machine) and a public key (placed on the server). Authentication succeeds only when the private key matches the public key — no password to guess or brute force.

### User: zoro

- **Home directory:** `/home/zoro`
- **Sudo access:** no
- **Authentication:** password

Create user:
```bash
sudo adduser zoro
```

Zoro has no sudo access — a standard unprivileged user account with password authentication only.

**Why no sudo for zoro:** The principle of least privilege — users should only have the permissions they need. Zoro does not require administrative access so none is granted.

Verify users:
```bash
id luffy
id zoro
```

---

## FTP Server

**Software:** vsftpd (Very Secure FTP Daemon)

vsftpd is a lightweight, secure FTP server. FTP (File Transfer Protocol) allows file transfer between a client and server over a network.

Install:
```bash
sudo apt install vsftpd -y
```

Configuration file: `/etc/vsftpd.conf`

```
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=NO
chroot_local_user=YES
allow_writeable_chroot=YES
user_sub_token=$USER
local_root=/backup
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
```

### User: nami

- **FTP access:** restricted to `/backup` directory
- **Permissions:** read-only
- **Anonymous access:** disabled

Create user:
```bash
sudo adduser nami
```

Add to userlist:
```bash
echo "nami" | sudo tee /etc/vsftpd.userlist
sudo chmod 644 /etc/vsftpd.userlist
```

Set backup directory permissions:
```bash
sudo chown root:root /backup
sudo chmod 755 /backup
```

Restart service:
```bash
sudo systemctl restart vsftpd
```

Test FTP connection:
```bash
ftp 192.168.1.168
```

**Why chroot_local_user:** Chroot jails the FTP user inside their designated directory — they cannot navigate above /backup and cannot access any other part of the filesystem. This is a critical security measure.

**Why no anonymous access:** Anonymous FTP allows anyone to connect without credentials — a significant security risk that could allow unauthorised file access.

---

## MySQL Database

**Version:** MySQL 8.0

MySQL is a relational database management system. WordPress uses MySQL to store all site content — posts, users, settings, comments, and configuration.

Install:
```bash
sudo apt install mysql-server -y
```

Run security setup:
```bash
sudo mysql_secure_installation
```

Security choices made:
- Validate password component: No
- Remove anonymous users: Yes
- Disallow root login remotely: Yes
- Remove test database: Yes
- Reload privilege tables: Yes

### WordPress Database Setup

```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
GRANT PROCESS ON *.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
```

**Why @'localhost':** The `@'localhost'` restriction means wp_user can only connect from within the server itself — not from any external machine. This keeps the database inaccessible from outside the server.

**Why not root:** The root MySQL account has full access to all databases. Using a dedicated wp_user with access only to wordpress_db limits damage if credentials are ever compromised.

Verify connection:
```bash
mysql -u wp_user -p wordpress_db
```

---

## WordPress

**Version:** 6.9.4

**URL:** `http://192.168.1.168/`

WordPress is an open-source content management system written in PHP that uses MySQL for data storage.

**Web server:** Apache2

**PHP modules installed:** php, php-mysql, php-curl, php-gd, php-mbstring, php-xml, php-xmlrpc, php-soap, php-intl, php-zip

Install Apache and PHP:
```bash
sudo apt install apache2 php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y
```

Download and install WordPress:
```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mv wordpress/* /var/www/html
sudo rm /var/www/html/index.html
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

WordPress database configuration in `wp-config.php`:
```php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'StrongPassword123!' );
define( 'DB_HOST', 'localhost' );
```

Secure the config file:
```bash
sudo chmod 600 /var/www/html/wp-config.php
```

**Why chmod 600 on wp-config.php:** wp-config.php contains database credentials. Setting permissions to 600 means only the file owner can read it — the web server serves WordPress but the config file itself is never exposed publicly. Visiting `http://192.168.1.168/wp-config.php` returns 403 Forbidden.

---

## Backup

A cron job runs every day at midnight (00:00) to create a compressed backup of the WordPress database.

**Backup script:** `/usr/local/bin/backup.sh`

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="/backup/wordpress_db_$DATE.tar.gz"
START_TIME=$(date +%T)

mysqldump -u wp_user -p'StrongPassword123!' wordpress_db | gzip > $BACKUP_FILE

echo "[$START_TIME] Backup successful: $BACKUP_FILE" >> /var/log/backup.log
```

Make executable:
```bash
sudo chmod +x /usr/local/bin/backup.sh
```

Cron job (`sudo crontab -e`):
```
0 0 * * * /usr/local/bin/backup.sh
```

**Cron schedule explained:**
```
0 0 * * *
│ │ │ │ │
│ │ │ │ └── day of week (any)
│ │ │ └──── month (any)
│ │ └────── day of month (any)
│ └──────── hour (0 = midnight)
└────────── minute (0)
```

**What each part does:**
- `mysqldump` — exports the WordPress database to SQL format
- `gzip` — compresses the output to save space
- `date +%Y-%m-%d_%H-%M-%S` — generates a timestamp for the filename so each backup is uniquely named
- `>> /var/log/backup.log` — appends a success line to the log file without overwriting previous entries

Backup files are stored in `/backup` which is accessible to the nami FTP user for download.

Test manually:
```bash
sudo /usr/local/bin/backup.sh
ls /backup
cat /var/log/backup.log
```

---

## Submission

Export VM:
```
VirtualBox → File → Export Appliance → deep-in-system.ova
```

Generate sha1sum:
```bash
sha1sum deep-in-system.ova > deep-in-system.sha1
```

Files submitted:
```
deep-in-system.sha1
README.md
```

---

## Author

Ouma Ouma — Zone01 Kisumu
