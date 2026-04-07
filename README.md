# Samba AD DC Lab – Full Commands for Windows Domain Join (Fedora)

---

# 🐧 A. SERVER (Fedora Linux – Samba AD DC)

## 0. Install Required Packages
```bash
dnf install -y samba samba-client samba-dc krb5-workstation krb5-libs krb5-auth-dialog bind-utils oddjob oddjob-mkhomedir policycoreutils-python-utils samba-winbind samba-winbind-clients
```

(Optional, for firewall management)
```bash
dnf install -y firewalld
systemctl enable --now firewalld
```

---

## 0.1 Set IP and Hostname
```bash
hostnamectl set-hostname server1.tp.local
nmcli con mod "System eth0" ipv4.addresses 192.168.100.10/24
nmcli con mod "System eth0" ipv4.gateway 192.168.100.1
nmcli con mod "System eth0" ipv4.dns 192.168.100.10
nmcli con mod "System eth0" ipv4.method manual
nmcli con up "System eth0"
ip addr show
hostname
```

---

## 1. Stop everything (clean state)
```bash
systemctl stop samba
```

## 2. Remove broken configuration
```bash
rm -rf /var/lib/samba/*
rm -f /etc/samba/smb.conf
```

## 3. Disable DNS conflicts (CRITICAL)
```bash
systemctl stop systemd-resolved
systemctl stop systemd-resolved.socket
systemctl stop systemd-resolved.service
systemctl stop systemd-resolved-varlink.socket
systemctl stop systemd-resolved-monitor.socket

systemctl disable systemd-resolved
systemctl disable systemd-resolved.socket
systemctl disable systemd-resolved.service
systemctl disable systemd-resolved-varlink.socket
systemctl disable systemd-resolved-monitor.socket
```

## 4. Fix DNS resolver
```bash
rm -f /etc/resolv.conf
echo "nameserver 127.0.0.1" > /etc/resolv.conf
chattr +i /etc/resolv.conf
```

## 5. Verify port 53 is FREE
```bash
ss -tulpn | grep :53
```

## 6. Provision the domain
```bash
samba-tool domain provision \
  --use-rfc2307 \
  --realm=TP.LOCAL \
  --domain=TP \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='Admin@123'
```

## 7. Fix Kerberos
```bash
\cp -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

## 8. Verify Kerberos config
```bash
cat /etc/krb5.conf
```

## 9. Start Samba
```bash
systemctl start samba
systemctl enable samba
systemctl status samba
```

---

## Firewall – Open all necessary AD ports
```bash
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-service=kerberos
firewall-cmd --permanent --add-service=ldap
firewall-cmd --permanent --add-service=samba
firewall-cmd --permanent --add-service=samba-client
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-service=cockpit

firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --permanent --add-port=88/tcp
firewall-cmd --permanent --add-port=88/udp
firewall-cmd --permanent --add-port=135/tcp
firewall-cmd --permanent --add-port=139/tcp
firewall-cmd --permanent --add-port=445/tcp
firewall-cmd --permanent --add-port=389/tcp
firewall-cmd --permanent --add-port=389/udp
firewall-cmd --permanent --add-port=636/tcp
firewall-cmd --permanent --add-port=137/udp
firewall-cmd --permanent --add-port=138/udp

firewall-cmd --reload
firewall-cmd --list-services
firewall-cmd --list-ports
```

---

## Samba AD DC – DNS & Kerberos Checks
```bash
samba-tool dns query 127.0.0.1 tp.local _ldap._tcp SRV -U "Administrator"
samba-tool dns query 127.0.0.1 tp.local _kerberos._tcp SRV -U "Administrator"

kinit Administrator
klist

samba-tool computer list
samba-tool computer delete WINCLIENT$

host tp.local 127.0.0.1
host server1.tp.local 127.0.0.1
```

---

# 🐧 B. LINUX CLIENT

## 0. Set IP and Hostname
```bash
hostnamectl set-hostname client1.tp.local
nmcli con mod "System eth0" ipv4.addresses 192.168.100.20/24
nmcli con mod "System eth0" ipv4.gateway 192.168.100.1
nmcli con mod "System eth0" ipv4.dns 192.168.100.10
nmcli con mod "System eth0" ipv4.method manual
nmcli con up "System eth0"
ip addr show
hostname
```

## Configure DNS
```bash
echo "nameserver 192.168.100.10" > /etc/resolv.conf
```

## Test DNS Resolution
```bash
host tp.local
host server1.tp.local
```

## Test Kerberos
```bash
kinit Administrator
klist
```

## Test LDAP / AD Connectivity
```bash
ldapsearch -x -H ldap://192.168.100.10 -b "dc=tp,dc=local"
```

---

# 🪟 C. WINDOWS CLIENT

## 0. Set IP and Hostname
```powershell
Rename-Computer -NewName WINCLIENT1 -Force
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.30 -PrefixLength 24 -DefaultGateway 192.168.100.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.100.10
Get-NetIPAddress
Get-DnsClientServerAddress
```

## 1. Time Sync – Windows Time Service
```powershell
Start-Service w32time
Set-Service w32time -StartupType Automatic

w32tm /config /manualpeerlist:"192.168.100.10" /syncfromflags:manual /update
w32tm /resync
w32tm /query /status

klist purge
```

## 2. Firewall – Allow AD/DC ports
```powershell
New-NetFirewallRule -DisplayName "Allow Kerberos TCP 88" -Direction Inbound -Protocol TCP -LocalPort 88 -Action Allow
New-NetFirewallRule -DisplayName "Allow Kerberos UDP 88" -Direction Inbound -Protocol UDP -LocalPort 88 -Action Allow

New-NetFirewallRule -DisplayName "Allow LDAP TCP 389" -Direction Inbound -Protocol TCP -LocalPort 389 -Action Allow
New-NetFirewallRule -DisplayName "Allow LDAP UDP 389" -Direction Inbound -Protocol UDP -LocalPort 389 -Action Allow

New-NetFirewallRule -DisplayName "Allow SMB TCP 445" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Allow
New-NetFirewallRule -DisplayName "Allow NetBIOS TCP 139" -Direction Inbound -Protocol TCP -LocalPort 139 -Action Allow
New-NetFirewallRule -DisplayName "Allow NetBIOS UDP 137" -Direction Inbound -Protocol UDP -LocalPort 137 -Action Allow
New-NetFirewallRule -DisplayName "Allow NetBIOS UDP 138" -Direction Inbound -Protocol UDP -LocalPort 138 -Action Allow

New-NetFirewallRule -DisplayName "Allow RPC TCP 135" -Direction Inbound -Protocol TCP -LocalPort 135 -Action Allow

New-NetFirewallRule -DisplayName "Allow DNS TCP/UDP 53" -Direction Inbound -Protocol TCP -LocalPort 53 -Action Allow
New-NetFirewallRule -DisplayName "Allow DNS UDP 53" -Direction Inbound -Protocol UDP -LocalPort 53 -Action Allow
```

## 3. Test AD/DC Connectivity
```powershell
Test-NetConnection 192.168.100.10 -Port 88
Test-NetConnection 192.168.100.10 -Port 389
Test-NetConnection 192.168.100.10 -Port 445
Test-NetConnection 192.168.100.10 -Port 135
Test-NetConnection 192.168.100.10 -Port 53
```

## 4. Join Domain
```powershell
Add-Computer -DomainName tp.local -Credential TP\Administrator -Restart
```

## Verify Domain
```powershell
nltest /dsgetdc:tp.local
nltest /sc_query:tp.local
```

---

# 👥 D. ADD USERS TO GROUPS (SERVER SIDE)
```bash
sudo bash << 'EOF'
samba-tool group addmembers IT alice,bob
samba-tool group addmembers HR carol,dave
samba-tool group addmembers Finance eve,frank

echo "=== IT ===" && samba-tool group listmembers IT
echo "=== HR ===" && samba-tool group listmembers HR
echo "=== Finance ===" && samba-tool group listmembers Finance
EOF
```
