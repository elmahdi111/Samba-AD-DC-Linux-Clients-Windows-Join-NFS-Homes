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

## smb.conf

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

# 🌐 DNS FIX (CRITICAL)

```bash
samba-tool dns add localhost _msdcs.tp.local _ldap._tcp.dc SRV "0 100 389 server1.tp.local" -U Administrator
samba-tool dns add localhost _msdcs.tp.local _kerberos._tcp.dc SRV "0 100 88 server1.tp.local" -U Administrator
samba-tool dns add localhost _msdcs.tp.local _gc._tcp.dc SRV "0 100 3268 server1.tp.local" -U Administrator
```

---

# 🪟 WINDOWS JOIN (NORMAL)

---

## Set DNS

```
192.168.100.10
```

---

## Verify DNS

```cmd
nslookup tp.local
nslookup -type=SRV _ldap._tcp.dc._msdcs.tp.local
```

---

## Join

```
System → Change settings → Domain: TP.LOCAL
```

---

# 🚨 WINDOWS JOIN TROUBLESHOOTING (VERY IMPORTANT)

---

## ❌ ERROR: "Domain not found"

### FIX

```cmd
ipconfig /all
```

👉 DNS must be:
```
192.168.100.10 ONLY
```

---

## ❌ ERROR: "DNS name does not exist"

### FIX on SERVER

```bash
samba-tool dns query localhost tp.local @ A -U Administrator
```

If missing → re-add SRV (again)

---

## ❌ ERROR: "Cannot contact domain"

### TEST FROM WINDOWS

```cmd
ping 192.168.100.10
ping server1.tp.local
```

If fails:
- Network problem
- /etc/hosts missing
- firewall blocking

---

## ❌ ERROR: "Clock skew too great"

### FIX TIME (VERY COMMON)

On Windows (Admin CMD):

```cmd
w32tm /resync
```

On Linux:

```bash
timedatectl set-ntp true
```

---

## ❌ ERROR: "Access denied"

Use:

```
Administrator
Password: Admin@123
```

NOT:
```
TP\carol
```

---

## ❌ ERROR: "The specified domain either does not exist or could not be contacted"

### FULL FIX CHECKLIST

✔ DNS → correct  
✔ SRV records → exist  
✔ Samba running  
✔ Firewall disabled  

```bash
systemctl status samba
systemctl disable --now firewalld
setenforce 0
```

---

## 🔍 ADVANCED DEBUG

### On Windows

```cmd
nltest /dsgetdc:tp.local
```

---

### On Server

```bash
journalctl -xe | grep samba
```

---

# 📡 NFS

---

## SERVER

```bash
dnf install nfs-utils -y
chown -R nfsnobody:nfsnobody /home
chmod 755 /home
```

```
/etc/exports
/home 192.168.100.0/24(rw,sync,no_root_squash)
```

```bash
systemctl enable --now nfs-server
exportfs -rav
```

---

## CLIENT

```bash
mkdir -p /home
mount -t nfs 192.168.100.10:/home /home
```

---

# ✅ FINAL RESULT

You now have:

- Samba Active Directory
- Windows domain join working
- Linux domain join working
- Department shares
- NFS home directories

---

# 🧠 REAL LESSON

If Windows join fails → **99% DNS issue**

Remember:

👉 AD = DNS + Kerberos + LDAP  
👉 If DNS broken → EVERYTHING fails
