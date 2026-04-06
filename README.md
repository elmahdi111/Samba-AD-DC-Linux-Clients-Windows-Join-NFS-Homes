# Samba AD DC + File Sharing Lab — Complete Guide

**Domain:** TP.LOCAL  
**Server:** SERVER1 — 192.168.100.10 (Fedora)  
**Client:** CLIENT1 — 192.168.100.129 (CentOS Stream 10)

---

## Architecture

```
SERVER1 (192.168.100.10) — Fedora
├── Samba AD DC (TP.LOCAL)
├── Users: carol, dave (IT) | bob, alice (HR) | eve, frank (Finance)
└── Shares:
    ├── /srv/share/IT        → IT group only
    ├── /srv/share/HR        → HR group only
    ├── /srv/share/Finance   → Finance group only
    └── /srv/share/Documents → All Domain Users

CLIENT1 (192.168.100.129) — CentOS Stream 10
└── SSSD joined to TP.LOCAL → AD authentication working
```

---

## SERVER SIDE

### 1. Create AD Groups

```bash
samba-tool group add IT
samba-tool group add HR
samba-tool group add Finance
```

### 2. Create AD Users and Add to Groups

```bash
# Create users
samba-tool user create carol
samba-tool user create dave
samba-tool user create bob
samba-tool user create alice
samba-tool user create eve
samba-tool user create frank

# Set passwords
for user in carol dave bob alice eve frank; do
  samba-tool user setpassword $user --newpassword="Admin@123"
done

# Add users to groups
samba-tool group addmembers IT carol dave
samba-tool group addmembers HR bob alice
samba-tool group addmembers Finance eve frank
```

### 3. Create Share Directories

```bash
mkdir -p /srv/share/IT /srv/share/HR /srv/share/Finance /srv/share/Documents
chmod 777 /srv/share/IT /srv/share/HR /srv/share/Finance /srv/share/Documents
```

### 4. Configure /etc/samba/smb.conf

```ini
# Global parameters
[global]
        dns forwarder = 127.0.0.53
        netbios name = SERVER1
        realm = TP.LOCAL
        server role = active directory domain controller
        workgroup = TP
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

### 5. Add Kerberos SPN for IP Address

```bash
samba-tool spn add ldap/192.168.100.10 SERVER1$
```

### 6. Restart and Verify Samba

```bash
testparm -s
systemctl restart samba
systemctl status samba
```

---

## CLIENT SIDE

### 1. Install Required Packages

```bash
sudo dnf install sssd sssd-ad adcli krb5-workstation \
  oddjob-mkhomedir samba-client cifs-utils -y
```

### 2. Configure /etc/hosts

```bash
sudo nano /etc/hosts
```

Add:
```
192.168.100.10   server1.tp.local server1
192.168.100.129  client1.tp.local client1
```

### 3. Configure Kerberos (/etc/krb5.conf)

```ini
[libdefaults]
    default_realm = TP.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = false
    rdns = false
    forwardable = true
    ticket_lifetime = 24h
    renew_lifetime = 7d

[realms]
    TP.LOCAL = {
        kdc = server1.tp.local
        admin_server = server1.tp.local
    }

[domain_realm]
    .tp.local = TP.LOCAL
    tp.local = TP.LOCAL
```

### 4. Join the Domain

```bash
sudo realm join -U Administrator TP.LOCAL
```

Enter Administrator's password when prompted.

### 5. Configure SSSD (/etc/sssd/sssd.conf)

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
sudo chmod 600 /etc/sssd/sssd.conf
```

### 6. Enable Home Directory Auto-Creation

```bash
sudo authselect select sssd with-mkhomedir --force
sudo systemctl enable --now oddjobd
```

### 7. Restart SSSD and Clear Cache

```bash
sudo systemctl stop sssd
sudo rm -rf /var/lib/sss/db/*
sudo rm -rf /var/lib/sss/mc/*
sudo systemctl start sssd
sudo systemctl enable sssd
```

### 8. Create Home Directories for AD Users

```bash
for user in carol dave bob alice eve frank; do
  sudo mkdir -p /home/$user
  sudo chown $user /home/$user
  sudo chmod 700 /home/$user
done
```

---

## Verification

### Verify AD Integration on Client

```bash
# Verify users
getent passwd carol
id carol

# Verify groups
getent group "IT"
getent group "HR"
getent group "Finance"
getent group "Domain Users"
```

Expected output:
```
carol:*:472001108:472000513:Carol White:/home/carol:/bin/bash
it:*:472001104:carol,dave
hr:*:472001103:bob,alice
finance:*:472001105:eve,frank
```

### Test Share Access from Linux Client

```bash
# carol (IT member) → ACCESS GRANTED
smbclient //192.168.100.10/IT -U "TP\carol%Admin@123" -c "ls"

# bob (HR member) → ACCESS DENIED on IT share
smbclient //192.168.100.10/IT -U "TP\bob%Admin@123" -c "ls"

# bob (HR member) → ACCESS GRANTED on HR share
smbclient //192.168.100.10/HR -U "TP\bob%Admin@123" -c "ls"

# Upload a file
echo "Hello from carol" > /tmp/test.txt
smbclient //192.168.100.10/IT -U "TP\carol%Admin@123" -c "put /tmp/test.txt test.txt"

# Download a file
smbclient //192.168.100.10/IT -U "TP\carol%Admin@123" -c "get test.txt /tmp/downloaded.txt"
cat /tmp/downloaded.txt
```

### Mount Share on Linux Client

```bash
sudo mkdir -p /mnt/IT
sudo mount -t cifs //192.168.100.10/IT /mnt/IT \
  -o username=carol,password=Admin@123,domain=TP,noperm
```

### Test Share Access from Windows Client

```cmd
REM Connect to share
net use \\192.168.100.10\IT /user:TP\carol Admin@123

REM Map as network drive
net use Z: \\192.168.100.10\IT /user:TP\carol Admin@123

REM Disconnect all
net use * /delete
```

---

## Access Control Summary

| Share | Path | Allowed Users |
|---|---|---|
| IT | /srv/share/IT | carol, dave (IT group) |
| HR | /srv/share/HR | bob, alice (HR group) |
| Finance | /srv/share/Finance | eve, frank (Finance group) |
| Documents | /srv/share/Documents | All Domain Users |

---

## Troubleshooting

```bash
# Check Samba service
systemctl status samba

# Check AD users and groups
samba-tool user list
samba-tool group list
samba-tool group listmembers IT

# Check SSSD logs
sudo journalctl -u sssd --no-pager | tail -30
sudo tail -30 /var/log/sssd/sssd_tp.local.log

# Check firewall (server)
firewall-cmd --list-services   # should include: samba

# Check Samba is listening
ss -tlnp | grep 445

# Reset a user password
samba-tool user setpassword carol --newpassword="Admin@123"

# Clear SSSD cache
sudo systemctl stop sssd
sudo rm -rf /var/lib/sss/db/*
sudo rm -rf /var/lib/sss/mc/*
sudo systemctl start sssd
```

---

## Key Notes

- `ad_enable_gc = False` — disables Global Catalog (port 3268) which Samba AD DC does not support
- `access_provider = permit` — allows all AD users to login via GUI (PAM)
- `use_fully_qualified_names = False` — allows login as `carol` instead of `carol@tp.local`
- Group names in smb.conf must use domain prefix: `@"TP\IT"` not `@IT`
- Windows allows only one credential set per server — run `net use * /delete` before switching users
