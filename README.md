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

```bash
echo "nameserver 192.168.100.10" > /etc/resolv.conf
host tp.local
host server1.tp.local

kinit Administrator
klist

ldapsearch -x -H ldap://192.168.100.10 -b "dc=tp,dc=local"
```

---

# 🪟 C. WINDOWS CLIENT

DNS, Time Sync, Firewall, AD Test, Domain Join commands (as before)...

---

# 👥 D. ADD USERS TO GROUPS
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
