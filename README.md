# 🧪 Full Lab — Samba AD DC + Linux Client + Windows Join + NFS

---

## 📌 Lab Information

| Parameter | Value |
|---|---|
| Domain (FQDN) | TP.LOCAL |
| NetBIOS Name | TP |
| Server Hostname | server1.tp.local |
| Server IP | 192.168.100.10 |
| Linux Client Hostname | client1.tp.local |
| Linux Client IP | 192.168.100.20 |
| Network | 192.168.100.0/24 |

---

## 🖥️ Part 1 — SERVER1 Configuration (Samba AD DC)

### 1.1 Set Hostname

```bash
hostnamectl set-hostname server1.tp.local
```

### 1.2 Configure Network

```bash
nmcli con mod "System eth0" ipv4.addresses 192.168.100.10/24
nmcli con mod "System eth0" ipv4.gateway 192.168.100.1
nmcli con mod "System eth0" ipv4.dns 127.0.0.1
nmcli con mod "System eth0" ipv4.method manual
nmcli con up "System eth0"
```

> **Note:** The server must point DNS to itself (`127.0.0.1`) because it will run the Samba internal DNS server.

### 1.3 Configure /etc/hosts

```bash
nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.100.10  server1.tp.local server1
```

### 1.4 Install Required Packages

```bash
dnf update -y
dnf install samba samba-dc samba-client krb5-workstation -y
```

### 1.5 Disable Conflicting Services

> These services conflict with the Samba AD DC. They must be stopped and disabled before provisioning.

```bash
systemctl disable --now smb nmb winbind 2>/dev/null
```

### 1.6 Remove Default Samba Config

```bash
mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

### 1.7 Provision the Domain

```bash
samba-tool domain provision \
  --use-rfc2307 \
  --realm=TP.LOCAL \
  --domain=TP \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL \
  --adminpass='Admin@123'
```

> **Note:** Use `--interactive` if you prefer to be prompted for each value instead.

### 1.8 Configure Kerberos

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 1.9 Enable and Start Samba

```bash
systemctl enable --now samba
systemctl status samba
```

### 1.10 Verify Domain Provisioning

```bash
samba-tool domain info 127.0.0.1
kinit administrator@TP.LOCAL
klist
```

### 1.11 Configure Firewall

```bash
firewall-cmd --permanent --add-service=samba
firewall-cmd --permanent --add-service=samba-dc
firewall-cmd --permanent --add-service=dns
firewall-cmd --permanent --add-service=kerberos
firewall-cmd --permanent --add-service=ldap
firewall-cmd --permanent --add-service=nfs
firewall-cmd --reload
```

> **Important:** Do not disable the firewall. Open only the required ports instead.

---

## 👥 Part 2 — Users and Groups

### 2.1 Create Groups

```bash
samba-tool group add IT
samba-tool group add HR
samba-tool group add Finance
```

### 2.2 Create Users

```bash
for u in carol dave bob alice eve frank; do
  samba-tool user create $u Admin@123
done
```

### 2.3 Add Users to Groups

```bash
samba-tool group addmembers IT carol dave
samba-tool group addmembers HR bob alice
samba-tool group addmembers Finance eve frank
```

### 2.4 Verify

```bash
samba-tool user list
samba-tool group listmembers IT
samba-tool group listmembers HR
samba-tool group listmembers Finance
```

---

## 📂 Part 3 — Samba File Shares

### 3.1 Create Share Directories

```bash
mkdir -p /srv/share/{IT,HR,Finance,Documents}
chmod 777 /srv/share/IT /srv/share/HR /srv/share/Finance /srv/share/Documents
```

### 3.2 Configure smb.conf

```bash
nano /etc/samba/smb.conf
```

```ini
# Global parameters
[global]
        netbios name = SERVER1
        realm = TP.LOCAL
        workgroup = TP
        server role = active directory domain controller
        dns forwarder = 8.8.8.8
        idmap_ldb:use rfc2307 = yes

[sysvol]
        path = /var/lib/samba/sysvol
        read only = No

[netlogon]
        path = /var/lib/samba/sysvol/tp.local/scripts
        read only = No

[IT]
        path = /srv/share/IT
        browseable = yes
        writable = yes
        valid users = @"TP\IT"
        create mask = 0666
        directory mask = 0777

[HR]
        path = /srv/share/HR
        browseable = yes
        writable = yes
        valid users = @"TP\HR"
        create mask = 0666
        directory mask = 0777

[Finance]
        path = /srv/share/Finance
        browseable = yes
        writable = yes
        valid users = @"TP\Finance"
        create mask = 0666
        directory mask = 0777

[Documents]
        path = /srv/share/Documents
        browseable = yes
        writable = yes
        valid users = @"TP\Domain Users"
        create mask = 0666
        directory mask = 0777
```

### 3.3 Validate and Restart

```bash
testparm -s
systemctl restart samba
```

### 3.4 Add Kerberos SPN for IP-based Access

```bash
samba-tool spn add ldap/192.168.100.10 SERVER1$
```

### 3.5 Test Shares Locally

```bash
smbclient //localhost/IT -U "TP\carol%Admin@123" -c "ls"
smbclient //localhost/HR -U "TP\bob%Admin@123" -c "ls"
smbclient //localhost/Finance -U "TP\eve%Admin@123" -c "ls"
```

---

## 📡 Part 4 — NFS Server

> NFS is configured to share home directories so that AD users have the same `/home` on both server and client.

### 4.1 Install NFS

```bash
dnf install nfs-utils -y
```

### 4.2 Set Correct Ownership on /home

```bash
chown root:root /home
chmod 755 /home
```

> **Note:** Do not use `nfsnobody` ownership on `/home` — this breaks AD user home directories. Keep `root:root` and let individual user directories have correct ownership.

### 4.3 Configure Exports

```bash
nano /etc/exports
```

```
/home  192.168.100.0/24(rw,sync,root_squash)
```

> **Note:** Use `root_squash` (not `no_root_squash`) unless you specifically need root access from clients. `no_root_squash` is a security risk.

### 4.4 Enable and Start NFS

```bash
systemctl enable --now nfs-server
exportfs -rav
exportfs -v
```

---

## 🖥️ Part 5 — CLIENT1 Configuration (Linux)

### 5.1 Set Hostname

```bash
hostnamectl set-hostname client1.tp.local
```

### 5.2 Configure Network

```bash
nmcli con mod "System eth0" ipv4.addresses 192.168.100.20/24
nmcli con mod "System eth0" ipv4.gateway 192.168.100.1
nmcli con mod "System eth0" ipv4.dns 192.168.100.10
nmcli con mod "System eth0" ipv4.method manual
nmcli con up "System eth0"
```

> **Critical:** The client DNS must point to the server (`192.168.100.10`) so that Kerberos and domain join work correctly.

### 5.3 Configure /etc/hosts

```bash
nano /etc/hosts
```

```
127.0.0.1       localhost
192.168.100.10  server1.tp.local server1
192.168.100.20  client1.tp.local client1
```

### 5.4 Install Required Packages

```bash
dnf install sssd sssd-ad adcli krb5-workstation \
  oddjob-mkhomedir samba-client cifs-utils -y
```

### 5.5 Verify DNS Resolution Before Joining

```bash
ping server1.tp.local
nslookup tp.local
```

Both must succeed before proceeding.

### 5.6 Join the Domain

```bash
realm join -U Administrator TP.LOCAL
```

Enter the Administrator password when prompted.

### 5.7 Configure SSSD

```bash
nano /etc/sssd/sssd.conf
```

```ini
[sssd]
domains = tp.local
config_file_version = 2
services = nss, pam

[domain/tp.local]
id_provider = ad
access_provider = permit
auth_provider = ad
chpass_provider = ad
ad_domain = tp.local
ad_server = server1.tp.local
override_homedir = /home/%u
default_shell = /bin/bash
use_fully_qualified_names = False
fallback_homedir = /home/%u
ldap_id_mapping = True
enumerate = True
cache_credentials = True
ad_enable_gc = False
krb5_auth_timeout = 15
```

```bash
chmod 600 /etc/sssd/sssd.conf
```

> **Key settings explained:**
> - `ad_enable_gc = False` — disables Global Catalog port 3268 which Samba AD DC does not support
> - `access_provider = permit` — allows all AD users to authenticate via PAM (GUI login)
> - `use_fully_qualified_names = False` — allows login as `carol` instead of `carol@tp.local`

### 5.8 Enable Home Directory Auto-Creation

```bash
authselect select sssd with-mkhomedir --force
systemctl enable --now oddjobd
```

### 5.9 Restart SSSD and Clear Cache

```bash
systemctl stop sssd
rm -rf /var/lib/sss/db/*
rm -rf /var/lib/sss/mc/*
systemctl enable --now sssd
```

### 5.10 Create Home Directories for AD Users

```bash
for user in carol dave bob alice eve frank; do
  mkdir -p /home/$user
  chown $user /home/$user
  chmod 700 /home/$user
done
```

### 5.11 Mount NFS Home Directory

```bash
mount -t nfs 192.168.100.10:/home /home
```

For permanent mount, add to `/etc/fstab`:

```
192.168.100.10:/home  /home  nfs  defaults,_netdev  0 0
```

### 5.12 Verify AD Integration

```bash
# Verify users
getent passwd carol
id carol

# Verify groups
getent group "IT"
getent group "HR"
getent group "Finance"
```

Expected output:
```
carol:*:472001108:472000513:Carol White:/home/carol:/bin/bash
it:*:472001104:carol,dave
hr:*:472001103:bob,alice
finance:*:472001105:eve,frank
```

### 5.13 Test Share Access from Linux Client

```bash
# carol (IT) → GRANTED
smbclient //192.168.100.10/IT -U "TP\carol%Admin@123" -c "ls"

# bob (HR) → DENIED on IT share
smbclient //192.168.100.10/IT -U "TP\bob%Admin@123" -c "ls"

# bob (HR) → GRANTED on HR share
smbclient //192.168.100.10/HR -U "TP\bob%Admin@123" -c "ls"

# Upload a file
echo "Hello from carol" > /tmp/test.txt
smbclient //192.168.100.10/IT -U "TP\carol%Admin@123" -c "put /tmp/test.txt test.txt"
```

### 5.14 Test Terminal Login

```bash
su - carol
id
```

---

## 🪟 Part 6 — Windows Client Join

### 6.1 Set DNS on Windows

Go to: **Control Panel → Network → Adapter Settings → IPv4 Properties**

```
DNS Server: 192.168.100.10
```

### 6.2 Verify DNS Resolution

```cmd
nslookup tp.local
nslookup server1.tp.local
```

Both must resolve to `192.168.100.10` before joining.

### 6.3 Synchronize Time

```cmd
w32tm /resync
```

> **Important:** Kerberos requires time difference to be less than 5 minutes between client and DC.

### 6.4 Join the Domain

Go to: **Settings → System → About → Domain or Workgroup**

- Select **Domain**
- Enter: `TP.LOCAL`
- Authenticate with: `Administrator` / `Admin@123`
- Restart when prompted

### 6.5 Test Domain Controller Connectivity

```cmd
nltest /dsgetdc:tp.local
```

### 6.6 Login as AD User

At the Windows login screen:

```
Username: TP\carol
Password: Admin@123
```

### 6.7 Access Shares from Windows

```cmd
REM Connect to share
net use \\192.168.100.10\IT /user:TP\carol Admin@123

REM Map as a network drive
net use Z: \\192.168.100.10\IT /user:TP\carol Admin@123

REM Disconnect all connections
net use * /delete
```

Or open File Explorer and navigate to:
```
\\192.168.100.10\IT
```

---

## 🔍 Part 7 — Troubleshooting

### DNS Issues

```bash
# On server — check Samba DNS is responding
samba-tool dns query 127.0.0.1 tp.local @ ALL -U Administrator

# On client — check DNS resolution
nslookup server1.tp.local 192.168.100.10
```

### Kerberos Issues

```bash
# Get a ticket
kinit administrator@TP.LOCAL
klist

# Time sync check (must be < 5 min difference)
date
```

### SSSD Issues

```bash
# Check logs
journalctl -u sssd --no-pager | tail -30
tail -30 /var/log/sssd/sssd_tp.local.log

# Clear cache and restart
systemctl stop sssd
rm -rf /var/lib/sss/db/* /var/lib/sss/mc/*
systemctl start sssd
```

### Samba Issues

```bash
# Validate config
testparm -s

# Check service
systemctl status samba

# Test shares
smbclient -L //localhost -U Administrator
```

### Windows Domain Join Issues

```cmd
REM Check DC connectivity
nltest /dsgetdc:tp.local

REM Check DNS
ipconfig /all

REM Flush DNS cache
ipconfig /flushdns

REM Resync time
w32tm /resync
```

---

## 📋 Access Control Summary

| Share | Path | Authorized Group | Members |
|---|---|---|---|
| IT | /srv/share/IT | TP\IT | carol, dave |
| HR | /srv/share/HR | TP\HR | bob, alice |
| Finance | /srv/share/Finance | TP\Finance | eve, frank |
| Documents | /srv/share/Documents | TP\Domain Users | all users |

---

## ⚠️ Security Notes

- Do **not** disable the firewall (`firewalld`). Use `firewall-cmd` to open specific ports only.
- Do **not** set `no_root_squash` in NFS exports unless strictly required.
- Do **not** use `setenforce 0` permanently. If SELinux causes issues, investigate with `audit2allow` instead.
- Passwords in production should follow complexity policies longer than `Admin@123`.

---

## 🧠 Key Rule

> **If anything fails — check DNS first.**
> 
> 90% of Samba AD DC, domain join, and Kerberos issues are caused by incorrect DNS configuration.
