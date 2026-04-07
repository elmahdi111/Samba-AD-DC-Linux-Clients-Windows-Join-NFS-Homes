# 🧪 FULL LAB — Samba AD DC + Linux Client + Windows Join + NFS (FROM ZERO)

---

# 📌 LAB INFO

- Domain: TP.LOCAL
- NetBIOS: TP
- Server: SERVER1
- IP Server: 192.168.100.10
- Client Linux: CLIENT1
- Network: 192.168.100.0/24

---

# 🖥️ SERVER1 (ROCKY / RHEL / CENTOS)

---

## 1️⃣ Set Hostname

```bash
hostnamectl set-hostname server1.tp.local
```

---

## 2️⃣ Configure IP Address

```bash
nmcli con show
nmcli con mod "System eth0" ipv4.addresses 192.168.100.10/24
nmcli con mod "System eth0" ipv4.gateway 192.168.100.1
nmcli con mod "System eth0" ipv4.dns 127.0.0.1
nmcli con mod "System eth0" ipv4.method manual
nmcli con up "System eth0"
```

---

## 3️⃣ Update /etc/hosts

```bash
nano /etc/hosts
```

```
192.168.100.10 server1.tp.local server1
```

---

## 4️⃣ Install Packages

```bash
dnf update -y
dnf install samba samba-dc samba-client krb5-workstation -y
```

---

## 5️⃣ Stop Default Services

```bash
systemctl disable --now smb nmb winbind
```

---

## 6️⃣ Provision Domain

```bash
samba-tool domain provision --use-rfc2307 --interactive
```

```
Realm: TP.LOCAL
Domain: TP
Server Role: DC
DNS Backend: SAMBA_INTERNAL
DNS Forwarder: 8.8.8.8
Administrator Password: Admin@123
```

---

## 7️⃣ Start Samba AD DC

```bash
systemctl enable --now samba
```

---

## 8️⃣ Verify

```bash
samba-tool user list
samba-tool group list
```

---

# 👥 USERS & GROUPS

---

## Create Groups

```bash
samba-tool group add IT
samba-tool group add HR
samba-tool group add Finance
```

---

## Create Users

```bash
for u in carol dave bob alice eve frank; do
  samba-tool user create $u Admin@123
done
```

---

## Add Users to Groups

```bash
samba-tool group addmembers IT carol dave
samba-tool group addmembers HR bob alice
samba-tool group addmembers Finance eve frank
```

---

# 📂 SAMBA SHARES

---

## Create Directories

```bash
mkdir -p /srv/share/{IT,HR,Finance,Documents}
chmod 777 /srv/share/*
```

---

## Configure Samba

```bash
nano /etc/samba/smb.conf
```

```
[global]
   workgroup = TP
   realm = TP.LOCAL
   netbios name = SERVER1
   server role = active directory domain controller
   dns forwarder = 8.8.8.8
   idmap_ldb:use rfc2307 = yes

[sysvol]
   path = /var/lib/samba/sysvol
   read only = no

[netlogon]
   path = /var/lib/samba/sysvol/tp.local/scripts
   read only = no

[IT]
   path = /srv/share/IT
   browseable = yes
   writable = yes
   valid users = @"TP\IT"

[HR]
   path = /srv/share/HR
   browseable = yes
   writable = yes
   valid users = @"TP\HR"

[Finance]
   path = /srv/share/Finance
   browseable = yes
   writable = yes
   valid users = @"TP\Finance"

[Documents]
   path = /srv/share/Documents
   browseable = yes
   writable = yes
   valid users = @"TP\Domain Users"
```

---

## Restart

```bash
systemctl restart samba
```

---

# 🌐 DNS FIX

---

```bash
samba-tool dns add localhost _msdcs.tp.local _ldap._tcp.dc SRV "0 100 389 server1.tp.local" -U Administrator
samba-tool dns add localhost _msdcs.tp.local _kerberos._tcp.dc SRV "0 100 88 server1.tp.local" -U Administrator
samba-tool dns add localhost _msdcs.tp.local _gc._tcp.dc SRV "0 100 3268 server1.tp.local" -U Administrator
```

---

# 📡 NFS SERVER

---

## Install

```bash
dnf install nfs-utils -y
```

---

## Configure

```bash
chown -R nfsnobody:nfsnobody /home
chmod 755 /home
```

---

```bash
nano /etc/exports
```

```
/home 192.168.100.0/24(rw,sync,no_root_squash)
```

---

## Start

```bash
systemctl enable --now nfs-server
exportfs -rav
```

---

# 🖥️ CLIENT1 (LINUX)

---

## 1️⃣ Hostname

```bash
hostnamectl set-hostname client1.tp.local
```

---

## 2️⃣ IP CONFIG

```bash
nmcli con mod "System eth0" ipv4.addresses 192.168.100.20/24
nmcli con mod "System eth0" ipv4.gateway 192.168.100.1
nmcli con mod "System eth0" ipv4.dns 192.168.100.10
nmcli con mod "System eth0" ipv4.method manual
nmcli con up "System eth0"
```

---

## 3️⃣ /etc/hosts

```bash
nano /etc/hosts
```

```
192.168.100.10 server1.tp.local
```

---

## 4️⃣ Install Packages

```bash
dnf install sssd sssd-ad adcli krb5-workstation oddjob-mkhomedir samba-client -y
```

---

## 5️⃣ Join Domain

```bash
realm join -U Administrator TP.LOCAL
```

---

## 6️⃣ Configure SSSD

```bash
nano /etc/sssd/sssd.conf
```

```
[sssd]
domains = tp.local
services = nss, pam

[domain/tp.local]
id_provider = ad
access_provider = permit
use_fully_qualified_names = False
fallback_homedir = /home/%u
default_shell = /bin/bash
```

---

## 7️⃣ Enable Home Creation

```bash
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
systemctl restart sssd
```

---

## 8️⃣ Test

```bash
su - carol
id carol
```

---

# 📡 NFS CLIENT

---

```bash
mkdir -p /home
mount -t nfs 192.168.100.10:/home /home
```

---

# 🪟 WINDOWS 10

---

## DNS

```
192.168.100.10
```

---

## TEST

```cmd
nslookup tp.local
nslookup -type=SRV _ldap._tcp.dc._msdcs.tp.local
```

---

## JOIN DOMAIN

```
System → Change settings → Domain: TP.LOCAL
```

---

## LOGIN

```
TP\carol
Password: Admin@123
```

---

# ✅ FINAL TESTS

---

## Samba

```bash
smbclient //localhost/IT -U "TP\carol%Admin@123"
```

---

## Access

| User | Access |
|------|--------|
| carol | IT ✅ |
| bob | HR ✅ |
| bob | IT ❌ |

---

# ⚠️ IMPORTANT

- Always use: @"TP\GROUP"
- DNS must point to SERVER1
- Restart services after config
- Firewall may need to be disabled:

```bash
systemctl disable --now firewalld
setenforce 0
```

---

# 🎯 RESULT

You now have:

- Samba AD Domain Controller
- Linux joined to AD
- Windows joined to AD
- Group-based shares
- NFS home directories

---
