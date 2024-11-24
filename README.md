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