<div align="center">

```
██████╗  ██████╗ ██████╗ ███╗   ██╗██████╗ ██████╗ ███████╗██████╗  ██████╗  ██████╗ ████████╗
██╔══██╗██╔═══██╗██╔══██╗████╗  ██║╚════██╗██╔══██╗██╔════╝██╔══██╗██╔═══██╗██╔═══██╗╚══██╔══╝
██████╔╝██║   ██║██████╔╝██╔██╗ ██║ █████╔╝██████╔╝█████╗  ██████╔╝██║   ██║██║   ██║   ██║   
██╔══██╗██║   ██║██╔══██╗██║╚██╗██║██╔═══╝ ██╔══██╗██╔══╝  ██╔══██╗██║   ██║██║   ██║   ██║   
██████╔╝╚██████╔╝██║  ██║██║ ╚████║███████╗██████╔╝███████╗██║  ██║╚██████╔╝╚██████╔╝   ██║   
╚═════╝  ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═══╝╚══════╝╚═════╝ ╚══════╝╚═╝  ╚═╝ ╚═════╝  ╚═════╝    ╚═╝   
```

### 42 School — System Administration Project

![Score](https://img.shields.io/badge/Score-125%2F100-brightgreen?style=for-the-badge)
![Bonus](https://img.shields.io/badge/Bonus-Completed-blue?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Debian-red?style=for-the-badge&logo=debian)
![VirtualBox](https://img.shields.io/badge/VM-VirtualBox-183A61?style=for-the-badge&logo=virtualbox)

</div>

---

## What Is This Project?

Born2beroot is a system administration project from 42 School. The goal is to set up a fully functional Linux server inside a virtual machine, following strict security rules and configuration requirements — no graphical interface, no shortcuts, just the terminal and your hands.

You go from a blank VM to a hardened, monitored, and production-like Debian server. Every decision is intentional. Every config file matters.

---

## Table of Contents

- [Virtual Machine Setup](#virtual-machine-setup)
- [Disk Partitioning (Bonus)](#disk-partitioning-bonus)
- [Operating System](#operating-system)
- [User & Group Management](#user--group-management)
- [Password Policy](#password-policy)
- [Sudo Configuration](#sudo-configuration)
- [SSH Configuration](#ssh-configuration)
- [Firewall — UFW](#firewall--ufw)
- [Monitoring Script](#monitoring-script)
- [Bonus — WordPress Stack](#bonus--wordpress-stack)
- [Key Concepts Learned](#key-concepts-learned)
- [Useful Commands](#useful-commands)

---

## Virtual Machine Setup

- **Hypervisor:** VirtualBox
- **Guest OS:** Debian (latest stable at time of evaluation)
- **RAM:** 1024 MB
- **Storage:** Dynamically allocated VDI disk
- No desktop environment installed — pure CLI server

The VM was configured to run headless and mimic how a real server would behave in a production environment.

---

## Disk Partitioning (Bonus)

One of the bonus requirements was to set up the partition structure correctly using **LVM (Logical Volume Manager)** with encryption via **LUKS**.

The partition layout follows the bonus structure:

```
sda
├── sda1      →  /boot         (primary, ~500MB, not encrypted)
└── sda5      →  LVM (LUKS encrypted)
                 ├── root      →  /
                 ├── swap      →  [swap]
                 ├── home      →  /home
                 ├── var       →  /var
                 ├── srv       →  /srv
                 ├── tmp       →  /tmp
                 └── var--log  →  /var/log
```

Why LVM? Because it gives you flexibility — you can resize volumes, add new ones, and manage storage without repartitioning the disk. Combined with LUKS encryption, all data on the logical volumes is encrypted at rest.

To verify partitioning:
```bash
lsblk
```

---

## Operating System

**Debian** was chosen over Rocky Linux for this project because:
- APT package management is straightforward and well-documented
- Stability is the core philosophy of Debian — perfect for a server
- Wide community support and long-term release cycles
- More relevant for the kind of sysadmin work introduced at this level

---

## User & Group Management

The project requires a specific user and group structure:

- A user with your login exists and belongs to both `sudo` and `user42` groups
- The `user42` group was created manually
- The `root` account direct login is disabled via SSH (only key-based or sudo escalation allowed)

```bash
# Check groups your user belongs to
groups <your_login>

# Verify user42 group exists
getent group user42

# Add a new user and assign group
sudo adduser <username>
sudo usermod -aG user42 <username>
```

---

## Password Policy

Strong password rules were enforced through two files:

### `/etc/login.defs`
```
PASS_MAX_DAYS   30
PASS_MIN_DAYS   2
PASS_WARN_AGE   7
```

### `/etc/pam.d/common-password` — using `libpam-pwquality`
```
minlen=10
ucredit=-1
dcredit=-1
maxrepeat=3
reject_username
difok=7
enforce_for_root
```

**What this means in plain terms:**
- Passwords expire every 30 days
- You must wait at least 2 days before changing a password again
- 7-day warning before expiry
- Minimum 10 characters, at least one uppercase, one digit
- Cannot repeat the same character more than 3 times in a row
- Cannot contain the username
- Must differ from the old password by at least 7 characters
- Root is not exempt from these rules

---

## Sudo Configuration

Sudo access was configured carefully using `visudo` and a custom file under `/etc/sudoers.d/`.

Rules applied:

```
Defaults     passwd_tries=3
Defaults     badpass_message="Wrong password. Try again."
Defaults     logfile="/var/log/sudo/sudo.log"
Defaults     log_input
Defaults     log_output
Defaults     requiretty
Defaults     secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

**Breakdown:**
- Max 3 password attempts before sudo access is denied
- Custom error message on wrong password
- Every sudo command — inputs and outputs — is logged to `/var/log/sudo/sudo.log`
- `requiretty` means sudo can only be run from a real terminal, not a script or background process
- `secure_path` restricts the PATH used during sudo execution to prevent hijacking

---

## SSH Configuration

SSH is the only way to access this server remotely. The configuration lives in `/etc/ssh/sshd_config`.

Key changes made:

```
Port 4242
PermitRootLogin no
```

- Default port 22 was changed to **4242** to reduce automated attacks
- Root login over SSH is completely disabled — you must log in as a regular user and escalate with sudo
- Password authentication is permitted for evaluation purposes

To connect to the VM:
```bash
ssh <your_login>@<vm_ip> -p 4242
```

To check SSH service status:
```bash
sudo systemctl status ssh
```

---

## Firewall — UFW

**UFW (Uncomplicated Firewall)** was installed and configured to control incoming traffic.

Only one port is open:

```bash
sudo ufw allow 4242
sudo ufw enable
```

For the WordPress bonus, port 80 was also opened:
```bash
sudo ufw allow 80
```

To see the current rules:
```bash
sudo ufw status verbose
```

The default policy is to **deny all incoming** traffic and **allow all outgoing**. Only explicitly allowed ports get through.

---

## Monitoring Script

One of the most hands-on parts of this project was writing `monitoring.sh` from scratch — a bash script that collects system information and broadcasts it to all logged-in users via `wall` every 10 minutes, automated through **cron**.

### `monitoring.sh`

```bash
#!/bin/bash

# Architecture
arch=$(uname -a)

# Physical CPUs
cpu_p=$(grep "physical id" /proc/cpuinfo | sort -u | wc -l)

# Virtual CPUs
cpu_v=$(grep "^processor" /proc/cpuinfo | wc -l)

# RAM
ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
ram_used=$(free --mega | awk '$1 == "Mem:" {print $3}')
ram_pct=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')

# Disk
disk_total=$(df -Bg | grep '^/dev/' | grep -v '/boot$' | awk '{ft += $2} END {print ft}')
disk_used=$(df -Bm | grep '^/dev/' | grep -v '/boot$' | awk '{ut += $3} END {print ut}')
disk_pct=$(df -Bm | grep '^/dev/' | grep -v '/boot$' | awk '{ut += $3} {ft+= $2} END {printf("%d"), ut/ft*100}')

# CPU Load
cpul=$(top -bn1 | grep '^%Cpu' | cut -c 9- | xargs | awk '{printf "%.1f%%", $1 + $3}')

# Last Boot
lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')

# LVM Check
lvmu=$(if [ $(lsblk | grep "lvm" | wc -l) -gt 0 ]; then echo yes; else echo no; fi)

# TCP Connections
tcpc=$(ss -neopt state established | grep -c tcp)

# Logged-in Users
ulog=$(users | wc -w)

# Network
ip=$(hostname -I | awk '{print $1}')
mac=$(ip link show | grep "link/ether" | awk '{print $2}')

# Sudo Commands
cmds=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

wall "  #Architecture: $arch
  #CPU physical: $cpu_p
  #vCPU: $cpu_v
  #Memory Usage: $ram_used/${ram_total}MB ($ram_pct%)
  #Disk Usage: $disk_used/${disk_total}Gb ($disk_pct%)
  #CPU load: $cpul
  #Last boot: $lb
  #LVM use: $lvmu
  #Connections TCP: $tcpc ESTABLISHED
  #User log: $ulog
  #Network: IP $ip ($mac)
  #Sudo: $cmds cmd"
```

### Cron Setup

The script runs automatically every 10 minutes using cron:

```bash
sudo crontab -u root -e
```

Entry added:
```
*/10 * * * * /usr/local/bin/monitoring.sh
```

To stop it from broadcasting without deleting it:
```bash
sudo crontab -u root -e
# Comment out or remove the line
```

---

## Bonus — WordPress Stack

The bonus required setting up a functional **WordPress** website using a specific stack:

| Component | Role |
|-----------|------|
| **lighttpd** | Web server (lightweight, fast) |
| **MariaDB** | Database |
| **PHP** | Backend scripting |
| **WordPress** | CMS |

### lighttpd
```bash
sudo apt install lighttpd
sudo ufw allow 80
sudo systemctl enable lighttpd
```

PHP support was enabled through the fastcgi module:
```bash
sudo lighty-enable-mod fastcgi
sudo lighty-enable-mod fastcgi-php
sudo service lighttpd force-reload
```

### MariaDB
```bash
sudo apt install mariadb-server
sudo mysql_secure_installation
```

A dedicated database and user were created for WordPress:
```sql
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
```

### PHP
```bash
sudo apt install php-cgi php-mysql
```

### WordPress
Downloaded and extracted into `/var/www/html/`, then configured `wp-config.php` with the database credentials created above.

---

## Key Concepts Learned

This project wasn't just about following steps. It forced you to actually understand what you're doing at each layer:

- **How virtualization works** — hypervisors, VDI disks, network adapters
- **Linux filesystem hierarchy** — what goes where and why
- **LVM and LUKS** — logical volume management and disk encryption from scratch
- **PAM (Pluggable Authentication Modules)** — how Linux enforces password rules
- **sudoers file syntax** — fine-grained privilege management
- **SSH hardening** — why changing the default port and disabling root login matters
- **UFW and iptables** — what a firewall actually does under the hood
- **Cron scheduling** — how to automate tasks reliably on a server
- **LAMP/LLMP stack** — how a web server, database, and scripting language connect to serve a website
- **Shell scripting** — parsing `/proc`, using `awk`, `grep`, `cut`, piping commands together meaningfully

---

## Useful Commands

```bash
# Check OS and kernel
uname -a
cat /etc/os-release

# Partition layout
lsblk

# LVM volumes
sudo lvdisplay

# Users and groups
cat /etc/passwd
cat /etc/group
groups <username>

# Password policy check
sudo chage -l <username>

# UFW rules
sudo ufw status verbose

# SSH service
sudo systemctl status ssh

# Cron jobs
sudo crontab -u root -l

# Active TCP connections
ss -neopt state established

# Sudo logs
sudo cat /var/log/sudo/sudo.log

# System resource overview
top
free -m
df -h
```

---

<div align="center">

*Built entirely through the terminal. No GUI. No shortcuts.*

**42 School — Born2beroot**

</div>
