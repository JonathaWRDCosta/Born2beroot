# **Born2beroot**

## **Project Overview**
The **Born2beroot** project is an introduction to server setup, focusing on the configuration and securing of a Linux virtual machine. This document outlines step-by-step instructions for setting up a Debian-based VM, implementing security measures, and creating a monitoring script for server management.

---

## **VM Configuration**

### **Hardware Requirements**
- **Base Memory:** 1024 MB
- **CPUs:** 4
- **Hard Disk:** 8 GB
- **Network Adapter:** NAT (initial), Bridged Adapter (final).

### **VM Details**
- **Hostname:** `yourlogin42`
- **Username:** `yourlogin`
- **Password:** set during OS installation

---

## **Step-by-Step Setup**

### **1. Installing sudo & Configuring Users and Groups**
1. Switch to superuser:
   ```bash
   su
   ```
2. Install `sudo`:
   ```bash
   apt install sudo
   sudo reboot
   ```
3. Create a new user and group:
   ```bash
   sudo adduser jonathro
   sudo addgroup user42
   getent group user42
   sudo adduser jonathro user42
   sudo adduser jonathro sudo
   ```

---

### **2. Setting Up SSH**
1. Install OpenSSH server:
   ```bash
   sudo apt update
   sudo apt install openssh-server
   sudo service ssh status
   ```
2. Update SSH configuration:
   ```bash
   sudo vim /etc/ssh/sshd_config
   ```
   - Change `Port 22` → `Port 4242`
   - Update `PermitRootLogin prohibit-password` → `PermitRootLogin no`

3. Restart SSH and verify:
   ```bash
   sudo service ssh restart
   sudo service ssh status
   ```

---

### **3. Configuring Firewall**
1. Install UFW:
   ```bash
   sudo apt install ufw
   sudo ufw enable
   sudo ufw allow 4242
   ```
2. Verify firewall rules:
   ```bash
   sudo ufw status
   ```

---

### **4. Enhancing Security**
1. Configure sudo logging:
   - Create a custom sudoers configuration:
     ```bash
     sudo touch /etc/sudoers.d/sudo_config
     sudo vim /etc/sudoers.d/sudo_config
     ```
     Add:
     ```plaintext
     Defaults  passwd_tries=3
     Defaults  badpass_message="Mensaje de error personalizado"
     Defaults  logfile="/var/log/sudo/sudo_config"
     Defaults  log_input, log_output
     Defaults  iolog_dir="/var/log/sudo"
     Defaults  requiretty
     Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
     ```
   - Create a directory for logs:
     ```bash
     mkdir /var/log/sudo
     ```

2. Enforce password policies:
   - Update `/etc/login.defs`:
     ```bash
     vim /etc/login.defs
     ```
     Change:
     ```plaintext
     PASS_MAX_DAYS 99999 → PASS_MAX_DAYS 30
     PASS_MIN_DAYS 99999 → PASS_MIN_DAYS 2
     ```

   - Install `libpam-pwquality`:
     ```bash
     sudo apt install libpam-pwquality
     ```
   - Edit PAM configuration:
     ```bash
     vim /etc/pam.d/common-password
     ```
     Add:
     ```plaintext
     minlen=10
     ucredit=-1
     dcredit=-1
     lcredit=-1
     maxrepeat=3
     reject_username
     difok=7
     enforce_for_root
     ```

3. Apply password changes to users:
   ```bash
   sudo chage -m 2 -M 30 root
   sudo chage -m 2 -M 30 jonathro
   ```

---

### **5. Networking**
1. Switch network adapter to **Bridged** in VM settings.

2. Configure static IP:
   ```bash
   sudo vim /etc/network/interfaces
   ```
   Add:
   ```plaintext
   auto enp0s8
   iface enp0s8 inet static
   address 192.168.1.251
   netmask 255.255.0.0
   gateway 10.11.254.254
   dns 10.11.254.254
   ```

3. Test SSH connection:
   ```bash
   ssh VMlogin@VM_ip -p 4242
   ```

---

### **6. Monitoring Script**
1. Create the monitoring script:
   - Navigate to `/usr/local/bin`:
     ```bash
     cd /usr/local/bin
     sudo touch monitoring.sh
     sudo chmod 777 monitoring.sh
     ```
   - Edit `monitoring.sh`:
     ```bash
     vim monitoring.sh
     ```
     Add the script:
     ```bash
     #!/bin/bash

     arch=$(uname -a)
     cpuf=$(grep "physical id" /proc/cpuinfo | wc -l)
     cpuv=$(grep "processor" /proc/cpuinfo | wc -l)
     ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
     ram_use=$(free --mega | awk '$1 == "Mem:" {print $3}')
     ram_percent=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')
     disk_total=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} END {printf ("%.1fGb\n"), disk_t/1024}')
     disk_use=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} END {print disk_u}')
     disk_percent=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} {disk_t+= $2} END {printf("%d"), disk_u/disk_t*100}')
     cpul=$(vmstat 1 2 | tail -1 | awk '{printf $15}')
     cpu_fin=$(printf "%.1f" $(expr 100 - $cpul))
     lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')
     lvmu=$(if [ $(lsblk | grep "lvm" | wc -l) -gt 0 ]; then echo yes; else echo no; fi)
     tcpc=$(ss -ta | grep ESTAB | wc -l)
     ulog=$(users | wc -w)
     ip=$(hostname -I)
     mac=$(ip link | grep "link/ether" | awk '{print $2}')
     cmnd=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

     wall "	Architecture: $arch
     	CPU physical: $cpuf
     	vCPU: $cpuv
     	Memory Usage: $ram_use/${ram_total}MB ($ram_percent%)
     	Disk Usage: $disk_use/${disk_total} ($disk_percent%)
     	CPU load: $cpu_fin%
     	Last boot: $lb
     	LVM use: $lvmu
     	Connections TCP: $tcpc ESTABLISHED
     	User log: $ulog
     	Network: IP $ip ($mac)
     	Sudo: $cmnd cmd"
     ```

2. Set permissions:
   ```bash
   sudo chmod 744 monitoring.sh
   ```

3. Grant user access to execute the script:
   ```bash
   sudo visudo
   ```
   Add:
   ```plaintext
   jonathro ALL=(ALL) NOPASSWD: /usr/local/bin/monitoring.sh
   ```

4. Automate execution:
   - Add to `crontab`:
     ```bash
     sudo crontab -u root -e
     ```
     Add:
     ```plaintext
     @reboot sleep 600 && while true; do /usr/local/bin/monitoring.sh; sleep 600; done
     ```

---


# Born2beroot - Virtual Machine Setup and Configuration

## Project Overview

In this project, a virtual machine (VM) environment was set up with the following primary goals:
- Installing and configuring a Debian-based virtual machine (VM) with essential security settings.
- Configuring user management, sudo privileges, and SSH access.
- Applying password policies, firewall settings, and monitoring scripts to ensure a secure, functional environment.

## Virtual Machine Setup

### VM Specifications:
- **VM Name**: Born2beroot
- **Username and Password**: Default set at the time of VM installation.
- **Hardware Configuration**:
  - **Base Memory**: 1024MB
  - **CPUs**: 4
  - **Hard Disk**: 8GB
- **Hostname**: `jonathro42`

### Operating System:
The chosen operating system is **Debian** due to its stability, security, and ease of use. Debian is known for being a solid foundation for both servers and desktops.

### Differences Between Debian and Rocky Linux:
- **Debian** is community-driven, general-purpose, and known for its flexibility and stability. It works well in both desktop and server environments.
- **Rocky Linux** is designed as an enterprise-class OS, offering binary compatibility with Red Hat Enterprise Linux (RHEL), focusing on long-term stability and enterprise support.

## Benefits of Virtual Machines
Virtual machines provide isolation, allowing multiple operating systems to run on a single physical machine. This environment:
- Provides enhanced security.
- Offers the ability to run and test different OS configurations.
- Allows for controlled environments to test software without affecting the host system.

## Configuration

### 1. System Configuration

1. **Verify VM Has No Graphical Environment at Startup**:
   - Command: `systemctl get-default` (should return `multi-user.target`)

2. **Password Request Upon Connection**:
   - Ensure the system requests a password before allowing connections.

3. **Login as Non-Root User**:
   - User `jonathro` has been created with `sudo` and `user42` group membership.

4. **Verify Sudo Installation**:
   - Command: `which sudo`

5. **Verify UFW (Firewall) Status**:
   - Command: `sudo ufw status`

6. **SSH Service Status**:
   - Command: `sudo service ssh status`

7. **Change Hostname**:
   - Command to change hostname: `sudo hostnamectl set-hostname jonathro42`
   - After reboot, verify the hostname change with `hostname`.

8. **Display Partitions**:
   - Command: `lsblk`

### 2. User Management

1. **Create a New User**:
   - Command: `sudo adduser nome_usuario`
   - Assign a password respecting the password policy.

2. **Assign User to Groups**:
   - Command to create `evaluating` group: `sudo addgroup evaluating`
   - Command to add the user to the `evaluating` group: `sudo usermod -aG evaluating nome_usuario`

3. **Verify User's Groups**:
   - Command: `groups nome_usuario`

### 3. Sudo Configuration

1. **Verify Sudo Installation**:
   - Command: `which sudo`

2. **Add User to the `sudo` Group**:
   - Command: `sudo usermod -aG sudo nome_usuario`

3. **Sudo Log Configuration**:
   - Create a directory for sudo logs: `sudo mkdir /var/log/sudo`
   - Configure the sudoers file: `sudo visudo`
     - Set options like `passwd_tries=3` and `logfile="/var/log/sudo/sudo_config"`
   
4. **Verify Sudo Logs**:
   - Command: `sudo cat /var/log/sudo/sudo_log`

### 4. UFW / Firewall Configuration

1. **Install and Check UFW**:
   - Command: `sudo ufw status`

2. **Add Firewall Rules**:
   - Open port `4242`: `sudo ufw allow 4242`
   - Add new rule for port `8080`: `sudo ufw allow 8080`
   - Delete the port 8080 rule: `sudo ufw delete allow 8080`

### 5. SSH Configuration

1. **Install and Verify SSH Service**:
   - Command: `sudo service ssh status`

2. **SSH Configuration**:
   - Edit the SSH configuration file to change the port to `4242`:
     - `sudo vim /etc/ssh/sshd_config` and update `Port` and `PermitRootLogin` settings.
   - Restart SSH service: `sudo service ssh restart`

3. **Verify SSH Connectivity**:
   - SSH into the VM using `ssh jonathro@192.168.1.251 -p 4242`

### 6. Script Monitoring Setup

1. **Create a Monitoring Script**:
   - Path: `/usr/local/bin/monitoring.sh`
   - The script collects system information like CPU usage, memory, disk, and network status, and displays it on the terminal every 10 minutes.

2. **Ensure Cron Job Runs**:
   - Add a cron job to execute the monitoring script every 10 minutes at boot: `sudo crontab -e`
   - Example entry: `@reboot sleep 600 && while true; do /usr/local/bin/monitoring.sh; sleep 600; done`

3. **Disable Cron After Script Setup**:
   - To stop the cron job from running after the system restart: `sudo systemctl stop cron` and `sudo systemctl disable cron`.

### 7. Conclusion

This setup covers:
- A secure and controlled environment using Debian as the chosen operating system.
- Proper user management, sudo privileges, and group assignments.
- Firewall setup with UFW and SSH service management.
- Monitoring system performance using a custom shell script executed via cron.

By completing these steps, the system is well-configured, secure, and ready for various tasks, including remote management and monitoring.