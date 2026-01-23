# LAB05 - Complete Samba 4 Active Directory Implementation

**Enterprise Active Directory Deployment with Samba 4 on Ubuntu Server 24.04**

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Network Architecture](#network-architecture)
- [Sprint 1: Domain Controller Setup](#sprint-1-domain-controller-setup)
- [Sprint 2: Users, Groups, OUs, and Password Policy](#sprint-2-users-groups-organizational-units)
- [Sprint 3: Shared Folders, Permissions, and Automatic Mounting](#sprint-3-shared-folders-and-permissions)
- [Sprint 4: Forest Trust Between Domains](#sprint-4-forest-trust-between-domains)
- [Windows 11 Client Configuration](#windows-11-client-configuration)
- [Ubuntu Desktop Client Configuration](#ubuntu-desktop-client-configuration)
- [Cross-Domain Authentication Testing](#cross-domain-authentication-testing)
- [Complete Command Reference](#complete-command-reference)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Project Overview

### Executive Summary

This project demonstrates a complete enterprise Active Directory implementation using Samba 4 on Ubuntu Server 24.04, including:

- ✅ Two Active Directory domain controllers (LAB05 and LAB06)
- ✅ Bidirectional forest trust between domains
- ✅ 7 domain users across multiple security groups
- ✅ 3 organizational units with delegation of control
- ✅ Group Policy Objects for password security
- ✅ Shared folders with granular ACL permissions
- ✅ Automatic folder mapping (logon scripts for Windows, PAM for Linux)
- ✅ Windows 11 and Ubuntu Desktop clients joined to domain
- ✅ Cross-domain authentication and resource access

### Infrastructure Details

| Component | LAB05 | LAB06 |
|-----------|-------|-------|
| **Domain** | lab05.lan | lab06.lan |
| **Realm** | LAB05.LAN | LAB06.LAN |
| **NetBIOS** | LAB05 | LAB06 |
| **DC Hostname** | ls05.lab05.lan | ls06.lab06.lan |
| **Internal IP** | 192.168.1.1/24 | 192.168.1.2/24 |
| **Bridge IP** | 172.30.20.41/25 | 172.30.20.42/25 |
| **OS** | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |
| **Samba Version** | 4.19.5 | 4.19.5 |

### Client Systems

| Client | Hostname | IP Address | Domain | OS |
|--------|----------|------------|--------|-----|
| Windows 11 | wc-05 | 192.168.1.100 | lab05.lan | Windows 11 |
| Ubuntu Desktop | lslc | 192.168.1.10 | lab05.lan | Ubuntu 24.04 |

### Credentials

**Linux System User:**
- Username: `administrador`
- Password: `admin_21`

**Domain Administrator (LAB05):**
- Username: `Administrator` or `administrator@LAB05.LAN`
- Password: `Admin_21`
- Domain: `LAB05\Administrator`

**Domain Administrator (LAB06):**
- Username: `Administrator` or `administrator@LAB06.LAN`
- Password: `Admin_21`
- Domain: `LAB06\Administrator`

---

## Network Architecture

```
                    Internet / External Network
                        DNS: 10.239.3.7/8
                              │
                        Gateway: 172.30.20.1
                              │
                  ┌───────────┴───────────┐
                  │   Bridge Network      │
                  │   172.30.20.0/25      │
                  └───────┬───────┬───────┘
                          │       │
                    .41   │       │   .42
                  ┌───────┴───┐ ┌─┴───────┐
                  │   ls05    │ │   ls06  │
                  │ LAB05 DC  │ │ LAB06 DC│
                  │172.30.20  │ │172.30.20│
                  │    .41    │ │    .42  │
                  └───────┬───┘ └─┬───────┘
                          │       │
                     .1   │       │   .2
                  ┌───────┴───────┴───────┐
                  │   Internal Network    │
                  │   192.168.1.0/24      │
                  │                       │
                  │   ls05: 192.168.1.1   │
                  │   ls06: 192.168.1.2   │
                  └───────┬───────────────┘
                          │
              ┌───────────┴───────────┐
              │   Domain Clients      │
              │                       │
              │  wc-05: .100 (Win11)  │
              │  lslc:  .10  (Ubuntu) │
              └───────────────────────┘

                  Forest Trust (Bidirectional)
              LAB05.LAN ←──────────→ LAB06.LAN
```

### Trust Relationship

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│   Forest: lab05.lan             │     │   Forest: lab06.lan             │
│   ├─ DC: ls05.lab05.lan         │◄───►│   ├─ DC: ls06.lab06.lan         │
│   ├─ IP: 192.168.1.1            │     │   ├─ IP: 192.168.1.2            │
│   ├─ Users: 7                   │     │   ├─ Users: testuser, testuser2 │
│   └─ Groups: 4                  │     │   └─ Trust Type: Forest         │
│                                 │     │                                 │
│   Trust Direction: BOTH         │     │   Trust Direction: BOTH         │
│   Trust Type: FOREST            │     │   Trust Type: FOREST            │
│   Transitive: YES               │     │   Transitive: YES               │
└─────────────────────────────────┘     └─────────────────────────────────┘
```

---

## Sprint 1: Domain Controller Setup

**Duration:** 6 hours  
**Objective:** Install and configure Samba 4 Active Directory Domain Controller

### Step 1: Initial System Configuration

**Set hostname:**
```bash
sudo hostnamectl set-hostname ls05.lab05.lan
hostname -f
```

**Update system:**
```bash
sudo apt update
sudo apt upgrade -y
```

**Configure `/etc/hosts`:**
```bash
sudo nano /etc/hosts
```

Add:
```
127.0.0.1 localhost
192.168.1.1 ls05.lab05.lan ls05

::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### Step 2: Network Configuration

**Edit Netplan:**
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Configuration:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.41/25
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses: [127.0.0.1, 10.239.3.7]
        search: [lab05.lan]
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.1.1/24
```

**Apply configuration:**
```bash
sudo netplan apply
ip a  # Verify IPs
ip route  # Verify routes
```

### Step 3: Disable systemd-resolved

⚠️ **CRITICAL:** Samba requires full control of port 53.

```bash
# Stop and disable systemd-resolved
sudo systemctl disable --now systemd-resolved

# Remove symbolic link
sudo unlink /etc/resolv.conf

# Create manual resolv.conf
sudo nano /etc/resolv.conf
```

**Content:**
```
nameserver 127.0.0.1
nameserver 10.239.3.7
nameserver 10.239.3.8
search lab05.lan
```

**Verify port 53 is free:**
```bash
sudo ss -tulnp | grep :53
# Should be empty
```

### Step 4: Install Samba

**Install packages:**
```bash
sudo apt install -y acl attr samba samba-dsdb-modules samba-vfs-modules \
  winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user \
  dnsutils ldap-utils
```

**Kerberos configuration prompts:**
- **Realm:** LAB05.LAN
- **Kerberos Server:** ls05.lab05.lan
- **Admin Server:** ls05.lab05.lan

**Stop default services:**
```bash
sudo systemctl disable --now smbd nmbd winbind
sudo systemctl stop smbd nmbd winbind
```

**Verify services are stopped:**
```bash
sudo systemctl status smbd
sudo systemctl status nmbd
sudo systemctl status winbind
```

### Step 5: Provision the Domain

**Remove default configuration:**
```bash
sudo rm -f /etc/samba/smb.conf
```

**Provision domain:**
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Provisioning answers:**

| Question | Answer |
|----------|--------|
| Realm | LAB05.LAN |
| Domain | LAB05 |
| Server Role | dc |
| DNS backend | SAMBA_INTERNAL |
| DNS forwarder | 10.239.3.7 |
| Administrator password | Admin_21 |

**Verify generated smb.conf:**
```bash
cat /etc/samba/smb.conf
```

Expected content:
```ini
[global]
    dns forwarder = 10.239.3.7
    netbios name = LS05
    realm = LAB05.LAN
    server role = active directory domain controller
    workgroup = LAB05

[sysvol]
    path = /var/lib/samba/sysvol
    read only = No

[netlogon]
    path = /var/lib/samba/sysvol/lab05.lan/scripts
    read only = No
```

### Step 6: Configure Kerberos

```bash
# Copy Kerberos configuration
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Verify content
cat /etc/krb5.conf
```

### Step 7: Start Samba AD DC

```bash
# Unmask, enable and start service
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc

# Verify status
sudo systemctl status samba-ad-dc
```

### Step 8: Verification

**Check domain level:**
```bash
sudo samba-tool domain level show
```

Expected output:
```
Forest function level: (Windows) 2008 R2
Domain function level: (Windows) 2008 R2
Lowest function level of a DC: (Windows) 2008 R2
```

**Check domain information:**
```bash
sudo samba-tool domain info 127.0.0.1
```

Expected output:
```
Forest           : lab05.lan
Domain           : lab05.lan
Netbios domain   : LAB05
DC name          : ls05.lab05.lan
DC netbios name  : LS05
Server site      : Default-First-Site-Name
Client site      : Default-First-Site-Name
```

**Verify DNS:**
```bash
# A record
host -t A ls05.lab05.lan

# SRV records
host -t SRV _ldap._tcp.lab05.lan
host -t SRV _kerberos._tcp.lab05.lan

# Reverse lookup
host 192.168.1.1
```

**Verify Kerberos:**
```bash
# Get ticket
kinit administrator@LAB05.LAN
# Password: Admin_21

# List tickets
klist

# Destroy ticket
kdestroy
```

**Verify LDAP:**
```bash
# With Kerberos authentication
kinit administrator@LAB05.LAN
ldapsearch -Y GSSAPI -H ldap://ls05.lab05.lan -b "DC=lab05,DC=lan" "(objectClass=user)" cn

# Clean up
kdestroy
```

**List users:**
```bash
sudo samba-tool user list
```

Expected output:
```
Administrator
Guest
krbtgt
```

**Verify listening ports:**
```bash
sudo ss -tulnp | grep -E ':(53|88|389|445|636|3268)'
```

Should show Samba listening on:
- Port 53 (DNS)
- Port 88 (Kerberos)
- Port 389 (LDAP)
- Port 445 (SMB)
- Port 636 (LDAPS)
- Port 3268 (Global Catalog)

### Sprint 1 Summary

✅ System updated and configured  
✅ Dual network (Bridge + Internal) operational  
✅ systemd-resolved disabled  
✅ Samba 4 installed and provisioned  
✅ Domain lab05.lan operational  
✅ DNS functioning correctly  
✅ Kerberos authenticating  
✅ LDAP responding  
✅ Services started and enabled  

---

## Sprint 2: Users, Groups, and Organizational Units

**Duration:** 6 hours  
**Objective:** Create organizational structure with OUs, security groups, and domain users

### Understanding OUs vs Groups

| Feature | OU (Organizational Unit) | Group |
|---------|-------------------------|-------|
| **Purpose** | Organization and GPO application | Assign permissions |
| **Contains** | Users, Groups, Computers, OUs | Only members (users) |
| **Permissions** | ❌ Cannot assign to resources | ✅ Assigned to resources |
| **GPOs** | ✅ Applied to OUs | ❌ Not applied to groups |

### Group Types and Scopes

**Types:**
- **Security Group:** Assigns permissions (most common)
- **Distribution Group:** Email only

**Scopes:**
- **Domain Local:** Permissions in local domain
- **Global:** Members from same domain (recommended)
- **Universal:** Across entire forest

### Step 1: Create Organizational Units

```bash
# Create IT_Department OU
sudo samba-tool ou create "OU=IT_Department,DC=lab05,DC=lan"

# Create HR_Department OU
sudo samba-tool ou create "OU=HR_Department,DC=lab05,DC=lan"

# Create Students OU
sudo samba-tool ou create "OU=Students,DC=lab05,DC=lan"
```

**Verify OUs:**
```bash
sudo samba-tool ou list
```

Expected output:
```
OU=IT_Department,DC=lab05,DC=lan
OU=HR_Department,DC=lab05,DC=lan
OU=Students,DC=lab05,DC=lan
OU=Domain Controllers,DC=lab05,DC=lan
```

### Step 2: Create Security Groups

```bash
# IT administrators group
sudo samba-tool group add IT_Admins

# HR staff group
sudo samba-tool group add HR_Staff

# Students group
sudo samba-tool group add Students

# Finance group (initially empty)
sudo samba-tool group add Finance

# Technical support group (for delegation)
sudo samba-tool group add Tech_Support
```

**Verify groups:**
```bash
sudo samba-tool group list | grep -E "(IT_Admins|HR_Staff|Students|Finance|Tech_Support)"
```

Expected output:
```
Finance
HR_Staff
IT_Admins
Students
Tech_Support
```

### Step 3: Create Domain Users

**Students group users:**
```bash
sudo samba-tool user create alice admin_21 \
  --given-name=Alice --surname=Wonderland

sudo samba-tool user create bob admin_21 \
  --given-name=Bob --surname=Marley

sudo samba-tool user create charlie admin_21 \
  --given-name=Charlie --surname=Sheen
```

**IT_Admins group users:**
```bash
sudo samba-tool user create iosif admin_21 \
  --given-name=Stalin --surname=Thegreat

sudo samba-tool user create karl admin_21 \
  --given-name=Karl --surname=Marx

sudo samba-tool user create lenin admin_21 \
  --given-name=Vladimir --surname=Lenin
```

**HR_Staff group users:**
```bash
sudo samba-tool user create vladimir admin_21 \
  --given-name=Vladimir --surname=Malakovsky
```

**Technical support user:**
```bash
sudo samba-tool user create techsupport admin_21 \
  --given-name=Tech --surname=Support
```

**Verify all users:**
```bash
sudo samba-tool user list
```

Expected output:
```
Administrator
alice
bob
charlie
Guest
iosif
karl
krbtgt
lenin
techsupport
vladimir
```

### Step 4: Assign Users to Groups

**Add members to Students group:**
```bash
sudo samba-tool group addmembers Students alice,bob,charlie
```

**Add members to IT_Admins group:**
```bash
sudo samba-tool group addmembers IT_Admins iosif,karl,lenin
```

**Add members to HR_Staff group:**
```bash
sudo samba-tool group addmembers HR_Staff vladimir
```

**Add techsupport to Tech_Support group:**
```bash
sudo samba-tool group addmembers Tech_Support techsupport
```

**Note:** Users are separated by commas with no spaces.

**Verify memberships:**
```bash
# Students group members
sudo samba-tool group listmembers Students

# IT_Admins group members
sudo samba-tool group listmembers IT_Admins

# HR_Staff group members
sudo samba-tool group listmembers HR_Staff

# Tech_Support group members
sudo samba-tool group listmembers Tech_Support

# Finance group (should be empty)
sudo samba-tool group listmembers Finance
```

**View user's group memberships:**
```bash
sudo samba-tool user show alice | grep memberOf
```

Expected output:
```
memberOf: CN=Students,CN=Users,DC=lab05,DC=lan
memberOf: CN=Domain Users,CN=Users,DC=lab05,DC=lan
```

### Step 5: Test User Authentication

**Test Kerberos authentication as alice:**
```bash
# Get ticket
kinit alice@LAB05.LAN
# Password: admin_21

# Verify ticket
klist

# Clean up
kdestroy
```

### Organizational Structure

```
lab05.lan
├── Domain Controllers (OU)
│   └── LS05 (Computer)
│
├── IT_Department (OU)
├── HR_Department (OU)
├── Students (OU)
│
├── Users (Container - default)
│   ├── alice → Students Group
│   ├── bob → Students Group
│   ├── charlie → Students Group
│   ├── iosif → IT_Admins Group
│   ├── karl → IT_Admins Group
│   ├── lenin → IT_Admins Group
│   ├── vladimir → HR_Staff Group
│   ├── techsupport → Tech_Support Group
│   │
│   ├── IT_Admins (Group)
│   ├── HR_Staff (Group)
│   ├── Students (Group)
│   ├── Finance (Group - empty)
│   └── Tech_Support (Group)
```

### Step 6: Configure Password Security Policy (GPO)

**Set strong password requirements for the domain:**

**View current password policy:**
```bash
sudo samba-tool domain passwordsettings show
```

**Configure secure password policy:**

```bash
# Minimum password length: 12 characters
sudo samba-tool domain passwordsettings set --min-pwd-length=12

# Enable password complexity (uppercase, lowercase, numbers, symbols)
sudo samba-tool domain passwordsettings set --complexity=on

# Password history: remember last 24 passwords
sudo samba-tool domain passwordsettings set --history-length=24

# Minimum password age: 1 day (prevents immediate changes)
sudo samba-tool domain passwordsettings set --min-pwd-age=1

# Maximum password age: 42 days (force periodic changes)
sudo samba-tool domain passwordsettings set --max-pwd-age=42

# Account lockout duration: 30 minutes
sudo samba-tool domain passwordsettings set --account-lockout-duration=30

# Account lockout threshold: 0 (disabled for lab)
sudo samba-tool domain passwordsettings set --account-lockout-threshold=0

# Reset lockout counter after: 30 minutes
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=30
```

**Verify password policy:**
```bash
sudo samba-tool domain passwordsettings show
```

Expected output:
```
Password information for domain 'DC=lab05,DC=lan'
Password complexity: on
Store plaintext passwords: off
Password history length: 24
Minimum password length: 12
Minimum password age (days): 1
Maximum password age (days): 42
Account lockout duration (mins): 30
Account lockout threshold (attempts): 0
Reset account lockout after (mins): 30
```

**Test password policy:**

Attempt to create user with weak password:
```bash
sudo samba-tool user create testgpo weak123
```

Should fail with:
```
ERROR: the password is too short. It should be equal or longer than 12 characters!
```

Create user with compliant password:
```bash
sudo samba-tool user create testgpo 'SecureP@ss2026!'
```

Should succeed. Clean up:
```bash
sudo samba-tool user delete testgpo
```

**Important notes:**
- Password policies apply domain-wide immediately
- Existing passwords are not affected until users change them
- Administrator account should be set to never expire:
  ```bash
  sudo samba-tool user setexpiry Administrator --noexpiry
  ```

### Sprint 2 Summary

✅ 3 Organizational Units created  
✅ 5 Security Groups created  
✅ 8 Domain users created  
✅ Users assigned to appropriate groups  
✅ **Password security policy configured (GPO)**  
✅ Structure verified with samba-tool  
✅ Kerberos authentication tested  

---

## Sprint 3: Shared Folders and Permissions

**Duration:** 6 hours  
**Objective:** Configure shared folders with granular ACL permissions

### Understanding Permission Layers

In Samba (like Windows Server), there are **TWO permission levels:**

1. **Share Permissions** (configured in smb.conf)
2. **File System Permissions** (POSIX + ACLs)

**Golden Rule:** The most restrictive permission wins.

### POSIX vs ACLs

**POSIX Basic Permissions (Limited):**
```
rwxrw-r--
│││││││││
││││││└└└─ Others (o)
│││└└└───  Group (g)
└└└─────   Owner (u)
```

**ACLs (Granular, similar to NTFS):**
- Multiple users and groups per file
- Permission inheritance
- Default permissions

### Planned Structure

```
/srv/samba/
├── finance/    → Finance group (R/W without delete)
├── hr/         → HR_Staff group (R/W)
└── public/     → Domain Users (Read-only)
```

### Permission Matrix

| Share | Finance | HR_Staff | Students | Domain Admins |
|-------|---------|----------|----------|---------------|
| **FinanceDocs** | R/W (no delete) | ❌ | ❌ | Full Control |
| **HRDocs** | ❌ | R/W | ❌ | Full Control |
| **Public** | R | R | R | Full Control |

### Step 1: Create Directories

```bash
# Create directory structure
sudo mkdir -p /srv/samba/{finance,hr,public}

# Verify
ls -la /srv/samba/
```

**Set owners and basic permissions:**
```bash
# Owner: root, Group: Domain Users
sudo chown -R root:"Domain Users" /srv/samba

# Base permissions
sudo chmod -R 770 /srv/samba

# Verify
ls -la /srv/samba/
```

Expected output:
```
drwxrwx--- 5 root Domain Users 4096 ... .
drwxrwx--- 2 root Domain Users 4096 ... finance
drwxrwx--- 2 root Domain Users 4096 ... hr
drwxrwx--- 2 root Domain Users 4096 ... public
```

### Step 2: Configure Shares in Samba

**Edit smb.conf:**
```bash
sudo nano /etc/samba/smb.conf
```

**Add after the [netlogon] section:**

```ini
#===================== Share Definitions =====================

[FinanceDocs]
    comment = Finance Department Documents
    path = /srv/samba/finance
    valid users = @Finance, @"Domain Admins"
    read only = no
    browseable = yes
    create mask = 0660
    directory mask = 0770

[HRDocs]
    comment = HR Department Documents
    path = /srv/samba/hr
    valid users = @HR_Staff, @"Domain Admins"
    read only = no
    browseable = yes
    create mask = 0660
    directory mask = 0770

[Public]
    comment = Public Shared Documents (Read-Only)
    path = /srv/samba/public
    valid users = @"Domain Users"
    read only = yes
    browseable = yes
    write list = @"Domain Admins"
```

**Verify syntax:**
```bash
testparm
```

Expected output:
```
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
```

**Reload Samba:**
```bash
sudo systemctl reload samba-ad-dc
```

**List shares:**
```bash
smbclient -L localhost -U administrator
```

Expected output:
```
FinanceDocs     Disk      Finance Department Documents
HRDocs          Disk      HR Department Documents
Public          Disk      Public Shared Documents (Read-Only)
```

### Step 3: Configure ACLs

**Install ACL tools:**
```bash
sudo apt install -y acl
```

**Configure FinanceDocs (R/W without delete):**
```bash
# Set ACL for Finance group
sudo setfacl -m g:Finance:rwx /srv/samba/finance

# Default ACL (for new files)
sudo setfacl -d -m g:Finance:rwx /srv/samba/finance

# Apply sticky bit (prevent deletion)
sudo chmod +t /srv/samba/finance

# Verify
getfacl /srv/samba/finance
ls -la /srv/samba/
```

Expected output showing sticky bit:
```
drwxrwx--T 2 root Domain Users ... finance
         ^-- T = sticky bit
```

**What does the sticky bit do?**
- Users can create files
- Users can only delete their own files
- Users cannot delete files created by others

**Configure HRDocs (Normal R/W):**
```bash
# Set ACL for HR_Staff group
sudo setfacl -m g:HR_Staff:rwx /srv/samba/hr

# Default ACL
sudo setfacl -d -m g:HR_Staff:rwx /srv/samba/hr

# Verify
getfacl /srv/samba/hr
```

**Configure Public (Read-only):**
```bash
# Read-only permissions for Domain Users
sudo setfacl -m g:"Domain Users":rx /srv/samba/public

# Default ACL
sudo setfacl -d -m g:"Domain Users":rx /srv/samba/public

# Verify
getfacl /srv/samba/public
```

**View all ACLs:**
```bash
for dir in finance hr public; do
    echo "=== /srv/samba/$dir ==="
    getfacl /srv/samba/$dir
    echo
done
```

### Step 4: Test from Server

**Create test files:**
```bash
# Authenticate as administrator
kinit administrator@LAB05.LAN

# Create test files
sudo -u administrator touch /srv/samba/finance/test_finance.txt
sudo -u administrator touch /srv/samba/hr/test_hr.txt
sudo -u administrator touch /srv/samba/public/test_public.txt

# Verify
ls -la /srv/samba/finance/
ls -la /srv/samba/hr/
ls -la /srv/samba/public/

# Clean up
kdestroy
```

**Test access with smbclient:**
```bash
# Connect to FinanceDocs as administrator
smbclient //ls05.lab05.lan/FinanceDocs -U administrator

# Inside SMB session:
smb: \> ls
smb: \> exit
```

### Share UNC Paths

Access shares from clients using:
- `\\ls05.lab05.lan\FinanceDocs`
- `\\ls05.lab05.lan\HRDocs`
- `\\ls05.lab05.lan\Public`

### Step 5: Configure Automatic Folder Mapping

**Objective:** Automatically mount shared folders when users log in

#### For Windows Clients - Logon Scripts

**Create logon script on DC:**
```bash
sudo mkdir -p /var/lib/samba/sysvol/lab05.lan/scripts
sudo nano /var/lib/samba/sysvol/lab05.lan/scripts/mapdrives.bat
```

**Script content:**
```batch
@echo off
REM Map network drives automatically
net use Z: \\ls05.lab05.lan\Public /persistent:yes >nul 2>&1
net use H: \\ls05.lab05.lan\HRDocs /persistent:yes >nul 2>&1
net use F: \\ls05.lab05.lan\FinanceDocs /persistent:yes >nul 2>&1
exit
```

**Set permissions:**
```bash
sudo chmod 755 /var/lib/samba/sysvol/lab05.lan/scripts/mapdrives.bat
```

**Assign script to users using LDAP:**

For individual users:
```bash
# For alice
sudo ldbmodify -H /var/lib/samba/private/sam.ldb <<EOF
dn: CN=Alice Wonderland,CN=Users,DC=lab05,DC=lan
changetype: modify
replace: scriptPath
scriptPath: mapdrives.bat
EOF
```

**Verify script assignment:**
```bash
sudo samba-tool user show alice | grep scriptPath
```

Expected output:
```
scriptPath: mapdrives.bat
```

**Assign to all users (script):**
```bash
# Create bulk assignment script
for user in alice bob charlie iosif karl lenin vladimir; do
  CN_NAME=$(sudo samba-tool user show $user | grep "^dn:" | cut -d',' -f1 | cut -d'=' -f2)
  sudo ldbmodify -H /var/lib/samba/private/sam.ldb <<EOF
dn: CN=$CN_NAME,CN=Users,DC=lab05,DC=lan
changetype: modify
replace: scriptPath
scriptPath: mapdrives.bat
EOF
done
```

#### For Linux Clients - PAM Session Scripts

**Create mount script on server:**
```bash
sudo mkdir -p /var/lib/samba/netlogon/linux
sudo nano /var/lib/samba/netlogon/linux/mount-shares.sh
```

**Script content:**
```bash
#!/bin/bash
# Automatic share mounting for domain users

USER=$PAM_USER
DOMAIN="LAB05"

# Create mount points
mkdir -p ~/Shared/{Public,HRDocs,FinanceDocs} 2>/dev/null

# Mount Public (all users)
if ! mountpoint -q ~/Shared/Public; then
    mount -t cifs //ls05.lab05.lan/Public ~/Shared/Public \
      -o username=$USER,domain=$DOMAIN,uid=$(id -u),gid=$(id -g),_netdev 2>/dev/null
fi

# Mount HRDocs (HR_Staff group only)
if groups | grep -q "HR_Staff"; then
    if ! mountpoint -q ~/Shared/HRDocs; then
        mount -t cifs //ls05.lab05.lan/HRDocs ~/Shared/HRDocs \
          -o username=$USER,domain=$DOMAIN,uid=$(id -u),gid=$(id -g),_netdev 2>/dev/null
    fi
fi

# Mount FinanceDocs (Finance group only)
if groups | grep -q "Finance"; then
    if ! mountpoint -q ~/Shared/FinanceDocs; then
        mount -t cifs //ls05.lab05.lan/FinanceDocs ~/Shared/FinanceDocs \
          -o username=$USER,domain=$DOMAIN,uid=$(id -u),gid=$(id -g),_netdev 2>/dev/null
    fi
fi

exit 0
```

**Set permissions:**
```bash
sudo chmod 755 /var/lib/samba/netlogon/linux/mount-shares.sh
```

**Configure on Linux clients:**

On each Linux client that joins the domain:

1. Copy script from server:
```bash
sudo scp administrador@ls05.lab05.lan:/var/lib/samba/netlogon/linux/mount-shares.sh \
  /usr/local/bin/
sudo chmod 755 /usr/local/bin/mount-shares.sh
```

2. Configure PAM to run script at login:
```bash
sudo nano /etc/pam.d/common-session
```

Add at the end:
```
session optional pam_exec.so /usr/local/bin/mount-shares.sh
```

3. Verify on next login:
```bash
# After logging in as domain user
ls ~/Shared/
# Should show: Public HRDocs FinanceDocs (depending on permissions)
```

#### Alternative: AutoFS (Advanced)

**For automatic on-demand mounting:**

```bash
# Install AutoFS
sudo apt install -y autofs

# Configure automount
sudo nano /etc/auto.master.d/samba.autofs
```

Content:
```
/home/LAB05 /etc/auto.samba --timeout=60
```

Create map file:
```bash
sudo nano /etc/auto.samba
```

Content:
```
* -fstype=cifs,rw,credentials=/etc/samba-credentials,uid=&,sec=krb5 ://ls05.lab05.lan/&
```

Restart autofs:
```bash
sudo systemctl restart autofs
sudo systemctl enable autofs
```

#### Testing Automatic Mapping

**Windows Client:**
1. Log in as domain user (e.g., alice@lab05.lan)
2. Open File Explorer
3. Verify mapped drives:
   - Z: → Public
   - H: → HRDocs (if in HR_Staff)
   - F: → FinanceDocs (if in Finance)

**Linux Client:**
1. Log in as domain user
2. Check mounted shares:
```bash
ls ~/Shared/
df -h | grep cifs
```

**Troubleshooting Automatic Mounting:**

**Windows - Script not running:**
```bash
# Verify script path on server
ls -la /var/lib/samba/sysvol/lab05.lan/scripts/

# Check user attribute
sudo samba-tool user show alice | grep scriptPath
```

**Linux - PAM script fails:**
```bash
# Check script exists
ls -la /usr/local/bin/mount-shares.sh

# Check PAM configuration
cat /etc/pam.d/common-session | grep pam_exec

# Test manual mount
/usr/local/bin/mount-shares.sh
```

### Sprint 3 Summary

✅ 3 shared directories created  
✅ Shares configured in smb.conf  
✅ Granular ACLs established  
✅ Sticky bit applied to Finance (prevent deletion)  
✅ Permissions verified  
✅ Server-side access tested  
✅ **Automatic folder mapping configured (Windows & Linux)**  
✅ **Logon scripts tested and working**  

---

## Sprint 4: Forest Trust Between Domains

**Duration:** 6 hours  
**Objective:** Create second AD domain and establish bidirectional forest trust

### Architecture

```
Forest 1: lab05.lan               Forest 2: lab06.lan
├── DC: ls05.lab05.lan            ├── DC: ls06.lab06.lan
├── IP: 192.168.1.1               ├── IP: 192.168.1.2
└── Users: 8                      └── Users: 2
            │
            │ ◄──── Forest Trust (Bidirectional) ────►
            │
```

### Trust Types

| Type | Scope | Direction | Use |
|------|-------|-----------|-----|
| **Forest Trust** | Complete forests | Bidirectional | Full integration between organizations |
| **External Trust** | Specific domains | Uni/Bidirectional | Limited access between domains |

### Domain Information

| Parameter | lab05.lan | lab06.lan |
|-----------|-----------|-----------|
| **Hostname** | ls05 | ls06 |
| **FQDN** | ls05.lab05.lan | ls06.lab06.lan |
| **Realm** | LAB05.LAN | LAB06.LAN |
| **NetBIOS** | LAB05 | LAB06 |
| **Internal IP** | 192.168.1.1/24 | 192.168.1.2/24 |
| **Bridge IP** | 172.30.20.41/25 | 172.30.20.42/25 |
| **DNS Primary** | 127.0.0.1 | 127.0.0.1 |
| **DNS Secondary** | 192.168.1.2 | 192.168.1.1 |

### Step 1: Create Second Domain Controller

**Follow Sprint 1 steps for ls06 with these differences:**

- **Hostname:** ls06.lab06.lan
- **Realm:** LAB06.LAN
- **Domain:** LAB06
- **Internal IP:** 192.168.1.2/24
- **Bridge IP:** 172.30.20.42/25

**Netplan configuration for ls06:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 172.30.20.42/25
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses: [127.0.0.1, 10.239.3.7]
        search: [lab06.lan]
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.1.2/24
      nameservers:
        addresses: [127.0.0.1, 192.168.1.1]
        search: [lab06.lan]
```

**Update `/etc/hosts` on ls06:**
```
127.0.0.1 localhost
192.168.1.2 ls06.lab06.lan ls06
192.168.1.1 ls05.lab05.lan ls05
```

**Provision LAB06 domain:**
```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

Answers:
- Realm: LAB06.LAN
- Domain: LAB06
- Server Role: dc
- DNS backend: SAMBA_INTERNAL
- DNS forwarder: 10.239.3.7
- Administrator password: Admin_21

### Step 2: Configure DNS Forwarding

**On ls05 (lab05.lan):**
```bash
sudo nano /etc/samba/smb.conf
```

Modify:
```ini
[global]
    dns forwarder = 192.168.1.2 10.239.3.7
```

Reload:
```bash
sudo systemctl reload samba-ad-dc
```

**On ls06 (lab06.lan):**
```bash
sudo nano /etc/samba/smb.conf
```

Modify:
```ini
[global]
    dns forwarder = 192.168.1.1 10.239.3.7
```

Reload:
```bash
sudo systemctl reload samba-ad-dc
```

**Test DNS resolution:**

From ls05:
```bash
host -t A ls06.lab06.lan
host -t SRV _ldap._tcp.lab06.lan
```

From ls06:
```bash
host -t A ls05.lab05.lan
host -t SRV _ldap._tcp.lab05.lan
```

Both should resolve correctly.

### Step 3: Create Forest Trust

**From ls05 to lab06.lan:**
```bash
sudo samba-tool domain trust create lab06.lan \
  --type=forest \
  --direction=both \
  -U administrator@lab06.lan
```

Enter password: `Admin_21`

Expected output:
```
Trust created successfully
```

**Verify trust:**
```bash
sudo samba-tool domain trust list
```

Expected output:
```
Type[Forest]   Transitive[Yes] Direction[BOTH]     Name[lab06.lan]
```

**Show trust details:**
```bash
sudo samba-tool domain trust show lab06.lan
```

Expected output:
```
LocalDomain Netbios[LAB05] DNS[lab05.lan] SID[S-1-5-21-...]
TrustedDomain:
NetbiosName:    LAB06
DnsName:        lab06.lan
SID:            S-1-5-21-...
Type:           0x2 (UPLEVEL)
Direction:      0x3 (BOTH)
Attributes:     0x8 (FOREST_TRANSITIVE)
```

**Validate trust:**
```bash
sudo samba-tool domain trust validate lab06.lan
```

Expected output:
```
OK: LocalValidation: DC[\\ls06.lab06.lan] CONNECTION[WERR_OK] TRUST[WERR_OK]
OK: LocalRediscover: DC[\\ls06.lab06.lan] CONNECTION[WERR_OK]
```

Note: Error about netlogon server is normal in Samba - the trust is working correctly.

### Step 4: Create Test Users in LAB06

**On ls06:**
```bash
# Create test users
sudo samba-tool user create testuser admin_21 \
  --given-name=Test --surname=User

sudo samba-tool user create testuser2 admin_21 \
  --given-name=Test2 --surname=User

# Verify
sudo samba-tool user list
```

### Step 5: Test Cross-Domain Authentication

**From ls05, authenticate as LAB06 user:**
```bash
kinit testuser@LAB06.LAN
# Password: admin_21

# Verify ticket
klist

# Clean up
kdestroy
```

**From ls06, authenticate as LAB05 user:**
```bash
kinit alice@LAB05.LAN
# Password: admin_21

# Verify ticket
klist

# Clean up
kdestroy
```

### Sprint 4 Summary

✅ Second DC (ls06.lab06.lan) created  
✅ Domain lab06.lan provisioned  
✅ DNS forwarding configured  
✅ Bidirectional forest trust established  
✅ Trust validated successfully  
✅ Cross-domain authentication tested  

---

## Windows 11 Client Configuration

**Hostname:** wc-05  
**IP Address:** 192.168.1.100  
**Domain:** lab05.lan  
**OS:** Windows 11 Professional

### Prerequisites

- Windows 11 Pro or Enterprise (Home edition cannot join domains)
- Network connectivity to domain controller
- DNS configured to point to DC

### Step 1: Configure Network Settings

**Option A: Static IP (Recommended for lab)**

1. Open **Settings** → **Network & Internet** → **Ethernet/Wi-Fi**
2. Click **Edit** next to IP assignment
3. Select **Manual**
4. Configure:
   - **IP address:** 192.168.1.100
   - **Subnet mask:** 255.255.255.0
   - **Gateway:** 192.168.1.1 (optional)
   - **Preferred DNS:** 192.168.1.1
   - **Alternate DNS:** (leave empty or 10.239.3.7)

**Option B: DHCP (if configured)**

Ensure DHCP server provides:
- IP in 192.168.1.0/24 range
- DNS server: 192.168.1.1

### Step 2: Verify Connectivity

**Open PowerShell or Command Prompt:**

```cmd
# Test connectivity to DC
ping 192.168.1.1

# Verify DNS resolution
nslookup lab05.lan
nslookup ls05.lab05.lan
nslookup _ldap._tcp.lab05.lan
```

Expected results:
```
lab05.lan resolves to 192.168.1.1
ls05.lab05.lan resolves to 192.168.1.1
_ldap._tcp.lab05.lan SRV record found
```

### Step 3: Join Domain

**Method 1: Settings App (Windows 11)**

1. Open **Settings** → **System** → **About**
2. Click **Domain or workgroup**
3. Under "Domain or workgroup", click **Domain**
4. Enter domain name: `lab05.lan`
5. Click **OK**
6. Enter credentials:
   - Username: `Administrator` or `administrator@lab05.lan`
   - Password: `Admin_21`
7. Click **OK**

**Welcome message should appear:**
```
Welcome to the lab05.lan domain
```

8. Click **OK**
9. Restart when prompted

**Method 2: System Properties (Classic)**

1. Press **Win + Pause** or search for "System Properties"
2. Click **Change settings** next to computer name
3. Click **Change...**
4. Select **Domain** radio button
5. Enter: `lab05.lan`
6. Click **OK**
7. Enter credentials:
   - Username: `Administrator`
   - Password: `Admin_21`
8. Welcome message appears
9. Click **OK** and restart

### Step 4: Verify Domain Join

**After reboot:**

1. On login screen, click **Other user**
2. Enter domain credentials:
   - Username: `alice` or `alice@lab05.lan` or `LAB05\alice`
   - Password: `admin_21`
3. Should successfully log in

**Verify from PowerShell:**
```powershell
# Check computer domain
(Get-WmiObject Win32_ComputerSystem).Domain

# Should output: lab05.lan

# View domain controller
nltest /dclist:lab05.lan

# Output should show: ls05.lab05.lan
```

### Step 5: Access Shared Folders

**From File Explorer:**

1. Press **Win + E**
2. In address bar, type: `\\ls05.lab05.lan`
3. Should see:
   - FinanceDocs (if user in Finance group)
   - HRDocs (if user in HR_Staff group)
   - Public (all domain users)

**Map network drive:**

1. Right-click **This PC** → **Map network drive**
2. Choose drive letter (e.g., Z:)
3. Enter path: `\\ls05.lab05.lan\Public`
4. Check **Reconnect at sign-in**
5. Click **Finish**

### Troubleshooting Windows Join

**Issue: "The specified domain either does not exist or could not be contacted"**

Solutions:
1. Verify DNS is set to 192.168.1.1
2. Test: `nslookup lab05.lan`
3. Ping DC: `ping ls05.lab05.lan`
4. Flush DNS: `ipconfig /flushdns`

**Issue: "The specified network password is not correct"**

Solutions:
1. Verify password is `Admin_21` (capital A)
2. Try username: `administrator@lab05.lan`
3. Check caps lock is off

**Issue: "Password has expired"**

Solutions:
On DC, reset Administrator password:
```bash
sudo samba-tool user setpassword Administrator --newpassword='Admin_21'
sudo samba-tool user setexpiry Administrator --noexpiry
```

### Windows 11 Client Summary

✅ Windows 11 Pro joined to lab05.lan  
✅ Domain users can log in  
✅ Network shares accessible  
✅ DNS resolution working  
✅ Authentication functioning correctly  

---

## Ubuntu Desktop Client Configuration

**Hostname:** lslc  
**IP Address:** 192.168.1.10  
**Domain:** lab05.lan  
**OS:** Ubuntu Desktop 24.04

### Prerequisites

- Ubuntu Desktop 22.04 or 24.04
- Network connectivity to DC
- Static IP or DHCP reservation

### Step 1: Configure Network

**For Ubuntu Desktop (GUI):**

1. Open **Settings** → **Network**
2. Click gear icon on connection
3. Go to **IPv4** tab
4. Select **Manual**
5. Configure:
   - **Address:** 192.168.1.10
   - **Netmask:** 255.255.255.0
   - **Gateway:** 192.168.1.1 (or leave empty)
   - **DNS:** 192.168.1.1
   - **Search domains:** lab05.lan
6. Click **Apply**

**For Ubuntu Server (CLI):**

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Configuration:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:  # Your interface name
      dhcp4: no
      addresses:
        - 192.168.1.10/24
      nameservers:
        addresses: [192.168.1.1]
        search: [lab05.lan]
```

Apply:
```bash
sudo netplan apply
```

### Step 2: Verify Connectivity

```bash
# Test DNS resolution
nslookup lab05.lan
nslookup ls05.lab05.lan

# Test ping
ping -c 4 192.168.1.1
ping -c 4 ls05.lab05.lan

# Test SRV records (critical for domain join)
host -t SRV _ldap._tcp.lab05.lan
```

Expected results:
```
lab05.lan has address 192.168.1.1
ls05.lab05.lan has address 192.168.1.1
_ldap._tcp.lab05.lan has SRV record 0 100 389 ls05.lab05.lan.
```

### Step 3: Set Hostname

```bash
# Set meaningful hostname
sudo hostnamectl set-hostname lslc

# Verify
hostname
hostname -f
```

### Step 4: Install Required Packages

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss \
  adcli samba-common-bin packagekit krb5-user
```

**Kerberos configuration prompts:**
- **Default Kerberos realm:** LAB05.LAN
- **Kerberos servers:** ls05.lab05.lan
- **Administrative server:** ls05.lab05.lan

### Step 5: Discover Domain

```bash
sudo realm discover lab05.lan
```

Expected output:
```
lab05.lan
  type: kerberos
  realm-name: LAB05.LAN
  domain-name: lab05.lan
  configured: no
  server-software: active-directory
  client-software: sssd
```

✅ Key indicator: `type: kerberos` and `server-software: active-directory`

### Step 6: Join Domain

```bash
sudo realm join --verbose --user=administrator lab05.lan
```

Enter password: `Admin_21`

Expected output:
```
 * Resolving: _ldap._tcp.lab05.lan
 * Performing LDAP DSE lookup on: 192.168.1.1
 * Successfully discovered: lab05.lan
 * Enrolling machine in realm
 * Calculated computer account name: LSLC
 * Using domain name: lab05.lan
 * Joining machine to realm
 * Successfully enrolled machine in realm
```

### Step 7: Verify Domain Join

```bash
sudo realm list
```

Expected output:
```
lab05.lan
  type: kerberos
  realm-name: LAB05.LAN
  domain-name: lab05.lan
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
  login-formats: %U@lab05.lan
  login-policy: allow-realm-logins
```

✅ Key indicator: `configured: kerberos-member`

**Verify computer account on DC:**
```bash
# On ls05
sudo samba-tool computer list
```

Should show:
```
LSLC$
WC-05$
```

### Step 8: Configure SSSD

**Edit SSSD configuration:**
```bash
sudo nano /etc/sssd/sssd.conf
```

**Configuration:**
```ini
[sssd]
domains = lab05.lan
config_file_version = 2
services = nss, pam

[domain/lab05.lan]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = LAB05.LAN
realmd_tags = manages-system joined-with-adcli
id_provider = ad
fallback_homedir = /home/%u@%d
ad_domain = lab05.lan
use_fully_qualified_names = True
ldap_id_mapping = True
access_provider = ad
```

**For short usernames (optional):**

Change these lines:
```ini
use_fully_qualified_names = False
fallback_homedir = /home/%u
```

**Restart SSSD:**
```bash
sudo systemctl restart sssd
sudo systemctl enable sssd
```

### Step 9: Configure Automatic Home Directory Creation

**Edit PAM session:**
```bash
sudo nano /etc/pam.d/common-session
```

**Add at the end:**
```
session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
```

This creates home directories automatically on first login.

### Step 10: Test Domain Login

**Method 1: SSH (if enabled)**
```bash
ssh alice@lab05.lan@localhost
# or if short names enabled:
ssh alice@localhost
```

**Method 2: GUI Login**

1. Log out of current session
2. At login screen, click **Not listed?**
3. Enter: `alice@lab05.lan`
4. Password: `admin_21`
5. Should successfully log in

**Method 3: Switch user**
```bash
su - alice@lab05.lan
```

### Step 11: Mount Shared Folders

**Install CIFS utilities:**
```bash
sudo apt install -y cifs-utils
```

**Manual mount:**
```bash
# Create mount point
sudo mkdir -p /mnt/public

# Mount share
sudo mount -t cifs //192.168.1.1/Public /mnt/public \
  -o username=alice,domain=LAB05

# List contents
ls -la /mnt/public/

# Unmount
sudo umount /mnt/public
```

**Permanent mount with credentials:**

Create credentials file:
```bash
nano ~/.smbcredentials
```

Content:
```
username=alice
password=admin_21
domain=LAB05
```

Protect file:
```bash
chmod 600 ~/.smbcredentials
```

Add to `/etc/fstab`:
```bash
sudo nano /etc/fstab
```

Add line:
```
//192.168.1.1/Public  /mnt/public  cifs  credentials=/home/alice/.smbcredentials,_netdev  0  0
```

Mount:
```bash
sudo mount -a
```

### Ubuntu Desktop Client Summary

✅ Ubuntu Desktop joined to lab05.lan  
✅ Realm configured with SSSD  
✅ Domain users can log in  
✅ Home directories created automatically  
✅ Network shares accessible via CIFS  
✅ Kerberos authentication working  

---

## Cross-Domain Authentication Testing

### Test 1: LAB05 User Authenticating to LAB06

**From ls05 domain controller:**

```bash
# Authenticate as testuser from LAB06
kinit testuser@LAB06.LAN
```

Enter password: `admin_21`

Expected output:
```
Warning: Your password will expire in 41 days on vie 06 mar 2026
```

**List Kerberos tickets:**
```bash
klist
```

Expected output:
```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: testuser@LAB06.LAN

Valid starting     Expires            Service principal
23/01/26 08:35:28  23/01/26 18:35:28  krbtgt/LAB06.LAN@LAB06.LAN
```

✅ **Result:** Cross-domain authentication successful

### Test 2: Access LAB06 Shares from LAB05

**List shares on LAB06:**
```bash
smbclient -L ls06.lab06.lan -U testuser@LAB06.LAN
```

Expected output:
```
Sharename       Type      Comment
---------       ----      -------
sysvol          Disk      
netlogon        Disk      
IPC$            IPC       IPC Service (Samba 4.19.5-Ubuntu)
```

✅ **Result:** Can access LAB06 shares from LAB05

### Test 3: Verify Trust Status

**From ls05:**
```bash
sudo samba-tool domain trust list
```

Expected output:
```
Type[Forest]   Transitive[Yes] Direction[BOTH]     Name[lab06.lan]
```

**Show trust details:**
```bash
sudo samba-tool domain trust show lab06.lan
```

Expected output:
```
LocalDomain Netbios[LAB05] DNS[lab05.lan]
TrustedDomain:
NetbiosName:    LAB06
DnsName:        lab06.lan
Type:           0x2 (UPLEVEL)
Direction:      0x3 (BOTH)
Attributes:     0x8 (FOREST_TRANSITIVE)
```

**Validate trust:**
```bash
sudo samba-tool domain trust validate lab06.lan
```

Expected output:
```
OK: LocalValidation: DC[\\ls06.lab06.lan] CONNECTION[WERR_OK] TRUST[WERR_OK]
OK: LocalRediscover: DC[\\ls06.lab06.lan] CONNECTION[WERR_OK]
```

Note: Netlogon error is normal in Samba - trust is working correctly.

### Cross-Domain Testing Summary

✅ LAB05 users can authenticate to LAB06  
✅ Kerberos tickets obtained successfully  
✅ Share access working across domains  
✅ Forest trust validated and operational  
✅ Bidirectional trust confirmed  

---

## Complete Command Reference

### Samba Domain Management

```bash
# Domain information
sudo samba-tool domain level show
sudo samba-tool domain info 127.0.0.1

# Password policies
sudo samba-tool domain passwordsettings show
sudo samba-tool domain passwordsettings set --min-pwd-length=12
sudo samba-tool domain passwordsettings set --complexity=on

# Service management
sudo systemctl status samba-ad-dc
sudo systemctl restart samba-ad-dc
sudo systemctl reload samba-ad-dc

# View logs
sudo journalctl -u samba-ad-dc -f
```

### User Management

```bash
# List users
sudo samba-tool user list

# Create user
sudo samba-tool user create <username> <password> \
  --given-name=<First> --surname=<Last>

# Delete user
sudo samba-tool user delete <username>

# Reset password
sudo samba-tool user setpassword <username> --newpassword=<password>

# Set password never expires
sudo samba-tool user setexpiry <username> --noexpiry

# Enable/disable user
sudo samba-tool user enable <username>
sudo samba-tool user disable <username>

# Show user details
sudo samba-tool user show <username>
```

### Group Management

```bash
# List groups
sudo samba-tool group list

# Create group
sudo samba-tool group add <groupname>

# Delete group
sudo samba-tool group delete <groupname>

# Add members
sudo samba-tool group addmembers <group> <user1,user2,user3>

# Remove members
sudo samba-tool group removemembers <group> <user>

# List group members
sudo samba-tool group listmembers <groupname>

# Show group details
sudo samba-tool group show <groupname>
```

### Organizational Unit Management

```bash
# List OUs
sudo samba-tool ou list

# Create OU
sudo samba-tool ou create "OU=<Name>,DC=lab05,DC=lan"

# Delete OU (must be empty)
sudo samba-tool ou delete "OU=<Name>,DC=lab05,DC=lan"

# Create user in specific OU
sudo samba-tool user create <user> <pass> \
  --userou="OU=IT_Department,DC=lab05,DC=lan"
```

### Computer Management

```bash
# List computers
sudo samba-tool computer list

# Delete computer
sudo samba-tool computer delete <computername>

# Show computer details
sudo samba-tool computer show <computername>
```

### Trust Management

```bash
# List trusts
sudo samba-tool domain trust list

# Show trust details
sudo samba-tool domain trust show <domain>

# Validate trust
sudo samba-tool domain trust validate <domain>

# Create forest trust
sudo samba-tool domain trust create <domain> \
  --type=forest --direction=both -U administrator@<domain>

# Delete trust
sudo samba-tool domain trust delete <domain>
```

### DNS Management

```bash
# Test DNS resolution
host -t A <hostname>
host -t SRV _ldap._tcp.<domain>
host -t SRV _kerberos._tcp.<domain>

# Query DNS server
nslookup <hostname> <dns-server>

# Reverse lookup
host <ip-address>
```

### Kerberos Authentication

```bash
# Get ticket
kinit <user>@<REALM>

# List tickets
klist

# Destroy tickets
kdestroy

# Test specific user
kinit alice@LAB05.LAN
```

### LDAP Queries

```bash
# Authenticate first
kinit administrator@LAB05.LAN

# Search users
ldapsearch -Y GSSAPI -H ldap://ls05.lab05.lan \
  -b "DC=lab05,DC=lan" "(objectClass=user)" cn sAMAccountName

# Search groups
ldapsearch -Y GSSAPI -H ldap://ls05.lab05.lan \
  -b "DC=lab05,DC=lan" "(objectClass=group)" cn

# Search OUs
ldapsearch -Y GSSAPI -H ldap://ls05.lab05.lan \
  -b "DC=lab05,DC=lan" "(objectClass=organizationalUnit)" dn

# Clean up
kdestroy
```

### Share Access

```bash
# List shares on server
smbclient -L <server> -U <user>

# Connect to share
smbclient //<server>/<share> -U <user>

# Mount share
sudo mount -t cifs //<server>/<share> /mnt/point \
  -o username=<user>,domain=<DOMAIN>

# Unmount share
sudo umount /mnt/point
```

### ACL Management

```bash
# View ACLs
getfacl <file_or_directory>

# Set ACL for user
setfacl -m u:<user>:rwx <file_or_directory>

# Set ACL for group
setfacl -m g:<group>:rwx <file_or_directory>

# Set default ACL
setfacl -d -m g:<group>:rwx <directory>

# Remove ACL
setfacl -x u:<user> <file_or_directory>

# Remove all ACLs
setfacl -b <file_or_directory>

# Copy ACLs
getfacl <source> | setfacl --set-file=- <destination>
```

### Network Diagnostics

```bash
# Show listening ports
sudo ss -tulnp | grep -E ':(53|88|389|445|636|3268)'

# Test connectivity
ping -c 4 <host>

# Trace route
traceroute <host>

# Check network configuration
ip a
ip route
cat /etc/resolv.conf
```

### System Management

```bash
# Check hostname
hostname
hostname -f

# Set hostname
sudo hostnamectl set-hostname <name>

# View system info
hostnamectl
uname -a

# Check service status
systemctl status samba-ad-dc
systemctl status sssd

# View system logs
journalctl -u samba-ad-dc
journalctl -u sssd
```

---

## Troubleshooting Guide

### DNS Resolution Issues

**Problem:** DNS queries not resolving domain names

**Solutions:**

1. **Verify DNS service is running:**
   ```bash
   sudo systemctl status samba-ad-dc
   ```

2. **Check resolv.conf:**
   ```bash
   cat /etc/resolv.conf
   # First nameserver should be 127.0.0.1
   ```

3. **Test DNS resolution:**
   ```bash
   host -t A ls05.lab05.lan
   host -t SRV _ldap._tcp.lab05.lan
   ```

4. **Check DNS forwarder:**
   ```bash
   cat /etc/samba/smb.conf | grep "dns forwarder"
   ```

5. **Restart Samba:**
   ```bash
   sudo systemctl restart samba-ad-dc
   ```

### Kerberos Authentication Failures

**Problem:** "Cannot find KDC" or authentication fails

**Solutions:**

1. **Verify krb5.conf:**
   ```bash
   cat /etc/krb5.conf
   # Should have correct realm and KDC settings
   ```

2. **Copy Samba's Kerberos config:**
   ```bash
   sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
   ```

3. **Test authentication:**
   ```bash
   kinit administrator@LAB05.LAN
   klist
   ```

4. **Check time synchronization:**
   ```bash
   timedatectl
   # Kerberos requires accurate time (< 5 min difference)
   ```

### Domain Join Failures

**Problem:** Cannot join Windows/Linux client to domain

**Solutions:**

1. **Verify DNS from client:**
   ```bash
   nslookup lab05.lan
   nslookup _ldap._tcp.lab05.lan
   ```

2. **Check network connectivity:**
   ```bash
   ping ls05.lab05.lan
   ```

3. **Verify Administrator password:**
   - Try: `Admin_21` (with capital A)
   - Reset if needed:
     ```bash
     sudo samba-tool user setpassword Administrator --newpassword='Admin_21'
     ```

4. **Check password expiration:**
   ```bash
   sudo samba-tool user setexpiry Administrator --noexpiry
   ```

5. **For Linux realm join:**
   ```bash
   # Check realm can discover domain
   sudo realm discover lab05.lan
   
   # Verbose join for debugging
   sudo realm join --verbose --user=administrator lab05.lan
   ```

### Trust Validation Errors

**Problem:** Trust validation fails or shows errors

**Solutions:**

1. **Verify DNS forwarding:**
   ```bash
   # Each DC should forward to the other
   cat /etc/samba/smb.conf | grep "dns forwarder"
   ```

2. **Test DNS resolution between DCs:**
   ```bash
   # From ls05
   host -t SRV _ldap._tcp.lab06.lan
   
   # From ls06
   host -t SRV _ldap._tcp.lab05.lan
   ```

3. **Restart Samba on both DCs:**
   ```bash
   sudo systemctl restart samba-ad-dc
   ```

4. **Re-validate trust:**
   ```bash
   sudo samba-tool domain trust validate lab06.lan
   ```

**Note:** Error about netlogon server is normal in Samba - check for "OK: LocalValidation"

### Share Access Denied

**Problem:** Users cannot access shared folders

**Solutions:**

1. **Verify user group membership:**
   ```bash
   sudo samba-tool group listmembers <groupname>
   sudo samba-tool user show <username> | grep memberOf
   ```

2. **Check share configuration:**
   ```bash
   testparm -s --section-name=<ShareName>
   ```

3. **Verify ACLs:**
   ```bash
   getfacl /srv/samba/<folder>
   ```

4. **Check Samba is running:**
   ```bash
   sudo systemctl status samba-ad-dc
   ```

5. **View Samba logs:**
   ```bash
   sudo tail -f /var/log/samba/log.samba
   ```

### Password Policy Violations

**Problem:** Cannot create users or change passwords

**Solutions:**

1. **Check current policy:**
   ```bash
   sudo samba-tool domain passwordsettings show
   ```

2. **Temporarily lower requirements:**
   ```bash
   sudo samba-tool domain passwordsettings set --min-pwd-length=8
   ```

3. **Use compliant password:**
   - At least 12 characters (for LAB05)
   - Contains uppercase, lowercase, numbers, symbols
   - Example: `SecureP@ss2026!`

### Port 53 Conflicts

**Problem:** Samba cannot start - port 53 in use

**Solutions:**

1. **Check what's using port 53:**
   ```bash
   sudo ss -tulnp | grep :53
   ```

2. **Disable systemd-resolved:**
   ```bash
   sudo systemctl disable --now systemd-resolved
   sudo unlink /etc/resolv.conf
   ```

3. **Create manual resolv.conf:**
   ```bash
   sudo nano /etc/resolv.conf
   # Add:
   # nameserver 127.0.0.1
   # nameserver 10.239.3.7
   ```

4. **Restart Samba:**
   ```bash
   sudo systemctl restart samba-ad-dc
   ```

### Network Configuration Not Persistent

**Problem:** Network settings reset after reboot

**Solutions:**

1. **Re-apply netplan:**
   ```bash
   sudo netplan apply
   ```

2. **Make netplan persistent:**
   ```bash
   # Ensure file is in /etc/netplan/
   ls -la /etc/netplan/
   
   # File should NOT be in /run/netplan/
   ```

3. **Disable cloud-init network config (if needed):**
   ```bash
   sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
   ```
   
   Add:
   ```
   network: {config: disabled}
   ```

### SSSD Not Starting on Linux Client

**Problem:** SSSD service fails to start

**Solutions:**

1. **Check SSSD configuration:**
   ```bash
   sudo sssctl config-check
   ```

2. **Verify permissions:**
   ```bash
   sudo chmod 600 /etc/sssd/sssd.conf
   ```

3. **Clear SSSD cache:**
   ```bash
   sudo systemctl stop sssd
   sudo rm -rf /var/lib/sss/db/*
   sudo systemctl start sssd
   ```

4. **Check logs:**
   ```bash
   sudo journalctl -u sssd -f
   ```

---

## Project Statistics

### Infrastructure

- **Domain Controllers:** 2 (LAB05, LAB06)
- **Domains:** 2 (lab05.lan, lab06.lan)
- **Forest Trust:** 1 (Bidirectional)
- **Clients:** 2 (Windows 11, Ubuntu Desktop)

### Active Directory Objects

- **Organizational Units:** 3 (IT_Department, HR_Department, Students)
- **Security Groups:** 5 (IT_Admins, HR_Staff, Students, Finance, Tech_Support)
- **Domain Users:** 8 in LAB05, 2 in LAB06
- **Computer Accounts:** 2 (WC-05, LSLC)

### Shared Resources

- **Shares:** 3 (FinanceDocs, HRDocs, Public)
- **ACL Rules:** 9+ (granular permissions per share)
- **Sticky Bit:** Applied to Finance share

### Security Policies

- **Password Complexity:** Enabled on both domains
- **Minimum Length:** 12 chars (LAB05), 7 chars (LAB06)
- **Password History:** 24 previous passwords
- **Password Expiration:** 42 days
- **Account Lockout:** 30 minutes duration

### Services Active

| Port | Protocol | Service | Status |
|------|----------|---------|--------|
| 53 | TCP/UDP | DNS | ✅ Active |
| 88 | TCP/UDP | Kerberos | ✅ Active |
| 389 | TCP | LDAP | ✅ Active |
| 636 | TCP | LDAPS | ✅ Active |
| 445 | TCP | SMB/CIFS | ✅ Active |
| 3268 | TCP | Global Catalog | ✅ Active |

---

## Key Learnings and Best Practices

### 1. DNS is Critical

- DNS must resolve correctly for AD to function
- SRV records are essential for domain operations
- Always set DC as primary DNS for clients
- DNS forwarding required for trust relationships

### 2. Samba-Specific Considerations

- systemd-resolved MUST be disabled (port 53 conflict)
- Netlogon errors in trust validation are normal
- Samba 4 provides good AD compatibility but has limitations
- Use samba-tool for most management tasks

### 3. Permission Architecture

- Two-layer permissions: Share + Filesystem
- Most restrictive permission wins
- ACLs provide Windows-like granularity
- Sticky bit useful for preventing deletions

### 4. Cross-Domain Operations

- DNS forwarding enables trust functionality
- Forest trust provides bidirectional authentication
- Kerberos tickets work across trusted domains
- Resource access requires proper group membership

### 5. Client Integration

- Windows 11 Pro/Enterprise required for domain join
- Linux requires SSSD + Realmd for modern AD integration
- DNS configuration critical on clients
- Home directory auto-creation improves user experience

### 6. Password Policies

- Domain-level policies apply to all users
- Cannot be overridden per-user
- Policy changes affect new passwords only
- Administrator account should have non-expiring password

### 7. Troubleshooting Approach

1. Always check DNS first
2. Verify Kerberos authentication
3. Confirm network connectivity
4. Check service status and logs
5. Validate permissions and group memberships

---

## Future Enhancements

### Recommended

- Advanced GPOs via RSAT from Windows (desktop restrictions, software deployment)
- LAM (LDAP Account Manager) web interface for easier management
- Email integration (Postfix/Dovecot with AD authentication)
- Certificate Authority (CA) for LDAPS with proper certificates
- Single Sign-On (SSO) for web applications
- Centralized logging and monitoring
- High availability with multiple DCs per domain
- Site-based replication for geographically distributed DCs
- Backup and disaster recovery procedures

### Advanced Topics

- PowerShell remoting to Samba DC
- Advanced GPO management
- Quota management on shares
- DFS (Distributed File System) namespace
- RODC (Read-Only Domain Controllers)
- Fine-Grained Password Policies

---

## References and Resources

### Official Documentation

- [Samba Wiki - Active Directory](https://wiki.samba.org/index.php/Active_Directory)
- [Samba AD DC HOWTO](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller)
- [Ubuntu Server Guide - Samba](https://ubuntu.com/server/docs/samba-active-directory)

### Community Resources

- [Samba Mailing Lists](https://lists.samba.org/)
- [Ask Ubuntu - Samba Tag](https://askubuntu.com/questions/tagged/samba)
- [Stack Overflow - Samba](https://stackoverflow.com/questions/tagged/samba)

### Tools

- samba-tool: Primary AD management tool
- testparm: Samba configuration validator
- smbclient: SMB/CIFS client for testing
- ldapsearch: LDAP query tool
- kinit/klist: Kerberos authentication tools
- realm: Domain join utility for Linux
- sssd: System Security Services Daemon

---

## Document Information

| Field | Value |
|-------|-------|
| **Project Name** | LAB05 - Samba 4 Active Directory Implementation |
| **Document Version** | 2.0 |
| **Last Updated** | January 23, 2026 |
| **Author** | System Administrator |
| **Environment** | Laboratory/Testing |
| **Total Duration** | 24+ hours (4 sprints + client setup) |
| **Difficulty Level** | Intermediate to Advanced |

---

## Sprint Completion Summary

| Sprint | Topic | Duration | Status |
|--------|-------|----------|--------|
| **Sprint 1** | Domain Controller Setup | 6 hours | ✅ Complete |
| **Sprint 2** | Users, Groups, OUs & Password Policy (GPO) | 6 hours | ✅ Complete |
| **Sprint 3** | Shared Folders, Permissions & Auto-Mapping | 6 hours | ✅ Complete |
| **Sprint 4** | Forest Trust | 6 hours | ✅ Complete |
| **Additional** | Windows 11 Client | 2 hours | ✅ Complete |
| **Additional** | Ubuntu Desktop Client | 2 hours | ✅ Complete |
| **Additional** | Cross-Domain Testing | 1 hour | ✅ Complete |

**Total Implementation Time:** ~29 hours

---

## Acknowledgments

This project demonstrates a complete enterprise Active Directory implementation using open-source tools. Special thanks to:

- The Samba Team for creating and maintaining Samba 4
- Ubuntu community for excellent documentation
- All contributors to SSSD, Realmd, and related tools

---

**End of Documentation**

*For questions, issues, or contributions, please refer to the project repository.*
