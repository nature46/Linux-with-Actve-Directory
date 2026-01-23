# LAB05 - Samba 4 Active Directory Implementation

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20LTS-E95420?logo=ubuntu)
![Samba](https://img.shields.io/badge/Samba-4.19.5-blue)
![Status](https://img.shields.io/badge/status-complete-success)

A comprehensive enterprise Active Directory implementation using Samba 4 on Ubuntu Server 24.04, featuring dual domain controllers, forest trust, client integration, and automatic resource mapping.

## 🎯 Project Overview

This project demonstrates a production-ready Active Directory infrastructure using open-source tools, suitable for small to medium enterprises or educational environments.

### Key Features

- **Dual Domain Controllers**: Two independent AD forests (LAB05 and LAB06) with bidirectional trust
- **User Management**: 8 domain users organized across 3 organizational units and 5 security groups
- **Secure Authentication**: Kerberos + LDAP with strong password policies (12-char minimum, complexity requirements)
- **Shared Resources**: 3 network shares with granular ACL permissions and automatic mounting
- **Cross-Platform Clients**: Windows 11 and Ubuntu Desktop domain integration
- **Enterprise Features**: GPOs, delegation of control, forest trusts, and automatic folder mapping

## 📊 Infrastructure

```
┌─────────────────────┐         ┌─────────────────────┐
│   LAB05 Domain      │◄────────►│   LAB06 Domain      │
│   ls05.lab05.lan    │  Trust  │   ls06.lab06.lan    │
│   192.168.1.1       │         │   192.168.1.2       │
└──────────┬──────────┘         └─────────────────────┘
           │
           │ Internal Network (192.168.1.0/24)
           │
    ┌──────┴──────┐
    │   Clients   │
    │  wc-05 .100 │ Windows 11
    │  lslc  .10  │ Ubuntu Desktop
    └─────────────┘
```

### System Specifications

| Component | LAB05 | LAB06 |
|-----------|-------|-------|
| **OS** | Ubuntu Server 24.04 LTS | Ubuntu Server 24.04 LTS |
| **Samba** | 4.19.5 | 4.19.5 |
| **Domain** | lab05.lan | lab06.lan |
| **Realm** | LAB05.LAN | LAB06.LAN |
| **Internal IP** | 192.168.1.1 | 192.168.1.2 |
| **RAM** | 4 GB | 4 GB |
| **Disk** | 40 GB | 40 GB |

## 🚀 Quick Start

### Prerequisites

- VirtualBox or VMware
- Ubuntu Server 24.04 LTS ISO
- Basic networking knowledge
- ~8 GB RAM total for both DCs
- ~80 GB disk space

### Installation Steps

1. **Clone this repository**
   ```bash
   git clone https://github.com/yourusername/LAB05-Samba-AD.git
   cd LAB05-Samba-AD
   ```

2. **Follow the implementation guide**
   - Read: [`LAB05_Complete_Implementation_Guide.md`](LAB05_Complete_Implementation_Guide.md)
   - Start with Sprint 1: Domain Controller Setup
   - Progress through all 4 sprints sequentially

3. **Estimated time**: 24-29 hours total
   - Sprint 1: 6 hours (DC setup)
   - Sprint 2: 6 hours (Users, groups, password policies)
   - Sprint 3: 6 hours (Shares, permissions, auto-mounting)
   - Sprint 4: 6 hours (Forest trust)
   - Client setup: 4-5 hours

## 📚 Documentation

### Main Guide
- **[LAB05 Complete Implementation Guide](LAB05_Complete_Implementation_Guide.md)** - Full step-by-step implementation (60+ pages)

### Quick Reference
- **Sprint 1**: Domain Controller Setup with Samba 4
- **Sprint 2**: Users, Groups, OUs, and Password Security Policies
- **Sprint 3**: Shared Folders with ACLs and Automatic Folder Mapping
- **Sprint 4**: Forest Trust Between LAB05 and LAB06

## 🏗️ Architecture Highlights

### Domain Structure

**LAB05.LAN:**
- **Organizational Units**: IT_Department, HR_Department, Students
- **Security Groups**: IT_Admins, HR_Staff, Students, Finance, Tech_Support
- **Users**: alice, bob, charlie, iosif, karl, lenin, vladimir, techsupport
- **Shared Folders**: FinanceDocs, HRDocs, Public

**LAB06.LAN:**
- **Test Users**: testuser, testuser2
- **Trust**: Bidirectional forest trust with LAB05

### Network Configuration

```
Bridge Network (172.30.20.0/25)
├── ls05: 172.30.20.41 (Internet access)
└── ls06: 172.30.20.42 (Internet access)

Internal Network (192.168.1.0/24)
├── ls05: 192.168.1.1 (Primary DNS/DC)
├── ls06: 192.168.1.2 (Secondary DNS/DC)
├── wc-05: 192.168.1.100 (Windows 11 client)
└── lslc: 192.168.1.10 (Ubuntu Desktop client)
```

## 🔐 Security Features

### Password Policy (GPO)
- **Minimum length**: 12 characters (LAB05), 7 characters (LAB06)
- **Complexity**: Enabled (uppercase, lowercase, numbers, symbols required)
- **History**: 24 previous passwords remembered
- **Expiration**: 42 days maximum age
- **Lockout**: 30-minute duration after failed attempts

### Access Control
- **Granular ACLs** on all shared folders
- **Sticky bit** on Finance folder (prevents unauthorized deletion)
- **Group-based permissions** for resource access
- **Delegation of control** for help desk operations

## 🛠️ Technologies Used

- **Samba 4.19.5** - Active Directory Domain Controller
- **Ubuntu Server 24.04 LTS** - Operating system
- **Kerberos** - Authentication protocol
- **LDAP** - Directory access protocol
- **SSSD** - System Security Services Daemon (Linux clients)
- **Realmd** - Domain join utility (Linux clients)
- **ACLs** - Access Control Lists for granular permissions
- **Netplan** - Network configuration

## 📖 Learning Outcomes

After completing this project, you will understand:

✅ How to deploy and configure Samba 4 as an Active Directory DC  
✅ DNS integration and SRV record management  
✅ Kerberos authentication and ticket management  
✅ LDAP directory structure and querying  
✅ User and group management in Active Directory  
✅ Organizational Unit design and GPO application  
✅ Share creation with granular ACL permissions  
✅ Forest trust configuration between domains  
✅ Client integration (Windows and Linux)  
✅ Automatic folder mapping using logon scripts  
✅ Troubleshooting AD authentication and access issues  

## 🧪 Testing & Validation

### Verification Checklist

- [x] DNS resolution working (A, SRV records)
- [x] Kerberos authentication functional
- [x] LDAP queries returning correct results
- [x] User login from both Windows and Linux clients
- [x] Shared folder access with correct permissions
- [x] Forest trust validated and operational
- [x] Cross-domain authentication working
- [x] Automatic folder mapping on client login
- [x] Password policy enforced on user creation

### Test Commands

```bash
# Verify DNS
host -t A ls05.lab05.lan
host -t SRV _ldap._tcp.lab05.lan

# Test Kerberos
kinit alice@LAB05.LAN
klist

# Check trust
sudo samba-tool domain trust list
sudo samba-tool domain trust validate lab06.lan

# Cross-domain authentication
kinit testuser@LAB06.LAN

# View users and groups
sudo samba-tool user list
sudo samba-tool group listmembers IT_Admins
```

## 🐛 Troubleshooting

Common issues and solutions are documented in the [Troubleshooting Guide](LAB05_Complete_Implementation_Guide.md#troubleshooting-guide) section of the main documentation.

**Quick fixes:**
- **DNS not resolving**: Check `/etc/resolv.conf` has `nameserver 127.0.0.1`
- **Port 53 conflict**: Disable systemd-resolved
- **Kerberos errors**: Copy `/var/lib/samba/private/krb5.conf` to `/etc/krb5.conf`
- **Trust fails**: Verify DNS forwarding in smb.conf
- **Password policy violations**: Reset with 12+ character password

## 📈 Project Statistics

- **Lines of Configuration**: 500+
- **Commands Documented**: 150+
- **Services Configured**: 6 (DNS, Kerberos, LDAP, LDAPS, SMB, Global Catalog)
- **Total Implementation Time**: 29 hours
- **Documentation Pages**: 60+

## 🤝 Contributing

This is an educational project. Contributions, suggestions, and improvements are welcome!

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add new feature'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🙏 Acknowledgments

- **Samba Team** - For creating and maintaining Samba 4
- **Ubuntu Community** - For excellent documentation and support
- **SSSD/Realmd Developers** - For seamless Linux AD integration

## 📧 Contact

For questions or feedback about this implementation:
- Open an issue in this repository
- Check the troubleshooting guide in the main documentation

## 🔗 Related Resources

- [Samba Official Wiki](https://wiki.samba.org/)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
- [Active Directory Documentation (Microsoft)](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/)
- [SSSD Documentation](https://sssd.io/)

---

**Project Status**: ✅ Complete and Production-Ready

*Last Updated: January 23, 2026*
