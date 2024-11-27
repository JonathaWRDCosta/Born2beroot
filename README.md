# **Born2beroot**

## **Project Overview**
The Born2beroot project is a foundational introduction to server configuration and management, focusing on setting up a secure Debian-based virtual machine (VM). This guide walks through user management, password policies, SSH configuration, firewall setup, and server monitoring using a custom script.

### **Key Objectives**
- Install and configure a Debian-based virtual machine.
- Secure the environment using user management, sudo privileges, and SSH access.
- Implement password policies and firewall rules to safeguard the server.
- Monitor server performance through an automated shell script.

---

## **Why Use Virtual Machines?**
Virtual machines (VMs) allow multiple operating systems to run on a single physical machine, providing:
- **Isolation:** A secure environment for testing and development.
- **Flexibility:** The ability to experiment with configurations safely.
- **Convenience:** Test software without affecting the host system.

---

## **Virtual Machine Configuration**
### **Specifications**
- **Base Memory:** 1024 MB  
- **CPUs:** 4  
- **Hard Disk:** 8 GB  
- **Network Adapter:** NAT (initial), Bridged Adapter (final)

### **System Details**
- **Hostname:** `yourlogin42`
- **Username:** `yourlogin`
- **Password:** Set during installation

---

## **Step-by-Step Setup**

### **1. Installing sudo & Configuring Users**
Switch to superuser:
```bash
su
```
Install sudo:
```bash
apt install sudo
sudo reboot
```
Create a user and assign groups:
```bash
sudo adduser yourlogin
sudo addgroup user42
sudo adduser yourlogin user42
sudo adduser yourlogin sudo
```

### **2. SSH Configuration**
Install and verify OpenSSH server:
```bash
sudo apt update
sudo apt install openssh-server
sudo service ssh status
```
Edit the SSH configuration file:
```bash
sudo vim /etc/ssh/sshd_config
```
Update:
- `Port 22` → `Port 4242`
- `PermitRootLogin prohibit-password` → `PermitRootLogin no`

Restart and verify:
```bash
sudo service ssh restart
sudo service ssh status
```

### **3. Configuring Firewall**
Install UFW and enable:
```bash
sudo apt install ufw
sudo ufw enable
sudo ufw allow 4242
```
Verify firewall rules:
```bash
sudo ufw status
```

### **4. Enforcing Security**
#### **Password Policies**
Edit `/etc/login.defs`:
```bash
vim /etc/login.defs
```
Change:
- `PASS_MAX_DAYS 99999` → `PASS_MAX_DAYS 30`
- `PASS_MIN_DAYS 0` → `PASS_MIN_DAYS 2`

Install password quality library:
```bash
sudo apt install libpam-pwquality
```
Edit `/etc/pam.d/common-password`:
```bash
vim /etc/pam.d/common-password
```
Add:
```
minlen=10
ucredit=-1
dcredit=-1
lcredit=-1
maxrepeat=3
reject_username
difok=7
enforce_for_root
```

#### **Sudo Logging**
Create a custom sudoers file:
```bash
sudo touch /etc/sudoers.d/sudo_config
sudo vim /etc/sudoers.d/sudo_config
```
Add:
```
Defaults  passwd_tries=3
Defaults  logfile="/var/log/sudo/sudo_config"
Defaults  requiretty
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```
Create the log directory:
```bash
sudo mkdir /var/log/sudo
```

### **5. Networking**
Switch the VM network adapter to "Bridged" in settings.

Configure static IP:
```bash
sudo vim /etc/network/interfaces
```
Add:
```
auto enp0s8
iface enp0s8 inet static
address 192.168.1.251
netmask 255.255.0.0
gateway 10.11.254.254
dns-nameservers 10.11.254.254
```
Restart networking:
```bash
sudo systemctl restart networking
```

Test SSH connection:
```bash
ssh yourlogin@192.168.1.251 -p 4242
```

---

## **Monitoring Script**
### **Purpose**
A custom shell script to monitor and display system metrics, including:
- CPU usage and load
- RAM and disk usage
- Network information
- Active TCP connections
- Number of sudo commands executed

### **Setup**
1. Create the script:
```bash
cd /usr/local/bin
sudo touch monitoring.sh
sudo chmod 744 monitoring.sh
sudo vim monitoring.sh
```
2. Add the following code:
```bash
#!/bin/bash

arch=$(uname -a)
cpuf=$(grep "physical id" /proc/cpuinfo | wc -l)
cpuv=$(grep "processor" /proc/cpuinfo | wc -l)
ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
ram_use=$(free --mega | awk '$1 == "Mem:" {print $3}')
ram_percent=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')
disk_total=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} END {printf("%.1fGb\n"), disk_t/1024}')
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
3. Automate execution using cron:
```bash
sudo crontab -e
```
Add:
```
@reboot sleep 600 && while true; do /usr/local/bin/monitoring.sh; sleep 600; done
```

---

## **Conclusion**
By completing these steps, you have set up a secure, efficient server environment on a Debian-based VM. The system is configured for user management, SSH access, and firewall protection, with automated monitoring to ensure optimal performance.

Happy coding! 🎉