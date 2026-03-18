<div align="center">

# 🔍 Active Directory Detection Lab

### Enterprise AD Environment with Splunk Security Monitoring

[![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)](https://github.com/yassine674/active-directory-detection-lab)
[![Platform](https://img.shields.io/badge/AD-Windows_Server_2019-0078D4?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows-server)
[![SIEM](https://img.shields.io/badge/SIEM-Splunk_Enterprise-000000?style=for-the-badge&logo=splunk&logoColor=white)](https://www.splunk.com/)
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)](LICENSE)

*Multi-OS security lab for AD attack detection and log correlation*

[📖 Documentation](#documentation) • [🚀 Quick Start](#quick-start) • [🎯 Features](#features) • [🔥 Attack Scenarios](#attack-scenarios)

</div>

---

## 📋 Table of Contents

- [Overview](#overview)
- [Lab Architecture](#lab-architecture)
- [Features](#features)
- [Technical Stack](#technical-stack)
- [Setup Guide](#setup-guide)
- [Splunk Configuration](#splunk-configuration)
- [Attack Scenarios](#attack-scenarios)
- [Detection Rules](#detection-rules)
- [Screenshots](#screenshots)
- [Learning Outcomes](#learning-outcomes)
- [Resources](#resources)
- [Author](#author)

---

## 🎯 Overview

This lab creates a **complete Active Directory environment** integrated with **Splunk Enterprise** for advanced security monitoring. It simulates real-world corporate infrastructure with multiple attack vectors for hands-on detection engineering practice.

### Project Objectives

- ✅ Build enterprise AD infrastructure (Domain Controller, users, GPOs)
- ✅ Deploy Splunk Universal Forwarders for log collection
- ✅ Configure Kerberos authentication monitoring
- ✅ Create correlation rules for attack detection
- ✅ Simulate realistic AD attacks (Kerberoasting, Golden Ticket, etc.)
- ✅ Perform behavioral anomaly analysis

---

## 🏗️ Lab Architecture
```
┌────────────────────────────────────────────────────────────────┐
│                     Splunk Enterprise Server                    │
│                    (Ubuntu 22.04 / 8GB RAM)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Indexer    │  │Search Head   │  │  Forwarder   │         │
│  │              │◄─┤              │◄─┤ Management   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Syslog / WinEvent forwarding
                              │
        ┌─────────────────────┼─────────────────────────┐
        │                     │                         │
┌───────▼─────────┐  ┌────────▼────────┐  ┌────────────▼──────┐
│ Windows Server  │  │  Windows 10     │  │   Kali Linux      │
│   2019 (DC)     │  │  (Domain User)  │  │  (Attack Box)     │
│                 │  │                 │  │                   │
│ • AD DS         │  │ • Domain Join   │  │ • Mimikatz        │
│ • DNS/DHCP      │  │ • Splunk UF     │  │ • Impacket        │
│ • Splunk UF     │  │ • Group Policy  │  │ • BloodHound      │
└─────────────────┘  └─────────────────┘  └───────────────────┘
    Domain: lab.local
```

**Network Configuration:**
- **Domain**: `lab.local`
- **DC IP**: `192.168.10.10`
- **Splunk Server IP**: `192.168.10.20`
- **Subnet**: `192.168.10.0/24`

---

## ✨ Features

### 🔐 Active Directory Components
- **Domain Controller** with DNS and DHCP services
- **Organizational Units (OUs)** structure
- **Group Policy Objects (GPOs)** for security hardening
- **Service Accounts** for Kerberoasting simulation
- **Privileged users** in Domain Admins group

### 📊 Splunk Integration
- **Universal Forwarders** on all Windows hosts
- **Windows Event Log** collection (Security, System, Application)
- **Sysmon** integration for advanced endpoint telemetry
- **Custom parsing** for Kerberos (Event ID 4768, 4769, 4771)
- **Real-time dashboards** for authentication monitoring

### 🎯 Detection Capabilities
- Kerberos anomaly detection (TGT/TGS requests)
- Failed authentication correlation
- Privilege escalation attempts
- Lateral movement detection
- Golden/Silver Ticket identification
- Pass-the-Hash detection

---

## 🔧 Technical Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| **Domain Controller** | Windows Server 2019 | Standard |
| **Client Machine** | Windows 10 Pro | 22H2 |
| **Attack Platform** | Kali Linux | 2024.1 |
| **SIEM** | Splunk Enterprise | 9.1.3 |
| **Endpoint Monitoring** | Sysmon | 15.0 |
| **Hypervisor** | VirtualBox / VMware | Latest |

---

## 🚀 Setup Guide

### Phase 1: Domain Controller Setup

**1. Install Windows Server 2019**
```powershell
# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.10 -PrefixLength 24 -DefaultGateway 192.168.10.1

# Set DNS to localhost
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

**2. Promote to Domain Controller**
```powershell
# Install AD DS role
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Configure domain
Install-ADDSForest -DomainName "lab.local" -DomainMode WinThreshold -ForestMode WinThreshold -InstallDns -Force
```

**3. Create Organizational Units**
```powershell
New-ADOrganizationalUnit -Name "LabUsers" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "LabComputers" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "ServiceAccounts" -Path "DC=lab,DC=local"
```

**4. Create Test Users**
```powershell
# Standard user
New-ADUser -Name "John Doe" -SamAccountName "jdoe" -UserPrincipalName "jdoe@lab.local" -Path "OU=LabUsers,DC=lab,DC=local" -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) -Enabled $true

# Service account (for Kerberoasting)
New-ADUser -Name "SQL Service" -SamAccountName "sqlsvc" -UserPrincipalName "sqlsvc@lab.local" -Path "OU=ServiceAccounts,DC=lab,DC=local" -AccountPassword (ConvertTo-SecureString "MySecretP@ss!" -AsPlainText -Force) -Enabled $true
Set-ADUser -Identity "sqlsvc" -ServicePrincipalNames @{Add="MSSQLSvc/sql01.lab.local:1433"}
```

---

### Phase 2: Splunk Server Setup

**1. Install Splunk on Ubuntu**
```bash
# Download Splunk
wget -O splunk-9.1.3-linux-2.6-amd64.deb 'https://download.splunk.com/products/splunk/releases/9.1.3/linux/splunk-9.1.3-linux-2.6-amd64.deb'

# Install
sudo dpkg -i splunk-9.1.3-linux-2.6-amd64.deb

# Start Splunk
sudo /opt/splunk/bin/splunk start --accept-license

# Enable boot start
sudo /opt/splunk/bin/splunk enable boot-start
```

**2. Configure Receiving Port**
```bash
# Enable receiver on port 9997
/opt/splunk/bin/splunk enable listen 9997 -auth admin:changeme
```

**3. Install Splunk Add-ons**
- **Splunk Add-on for Microsoft Windows**
- **Splunk Add-on for Sysmon**
- **Splunk Security Essentials**

---

### Phase 3: Deploy Universal Forwarders

**On Windows DC:**
```powershell
# Download UF
Invoke-WebRequest -Uri "https://download.splunk.com/products/universalforwarder/releases/9.1.3/windows/splunkforwarder-9.1.3-x64-release.msi" -OutFile "splunkforwarder.msi"

# Install silently
msiexec /i splunkforwarder.msi RECEIVING_INDEXER="192.168.10.20:9997" AGREETOLICENSE=Yes /quiet

# Configure inputs
$InputsConf = @"
[WinEventLog://Security]
disabled = 0
index = windows

[WinEventLog://System]
disabled = 0
index = windows

[WinEventLog://Application]
disabled = 0
index = windows
"@

Set-Content -Path "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf" -Value $InputsConf

# Restart forwarder
Restart-Service SplunkForwarder
```

---

### Phase 4: Install Sysmon
```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive -Path "Sysmon.zip" -DestinationPath "C:\Tools\Sysmon"

# Download SwiftOnSecurity config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Tools\Sysmon\sysmonconfig.xml"

# Install Sysmon
C:\Tools\Sysmon\Sysmon64.exe -accepteula -i C:\Tools\Sysmon\sysmonconfig.xml
```

**Add Sysmon to Splunk inputs:**
```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = windows
renderXml = true
```

---

## ⚙️ Splunk Configuration

### Create Index for Windows Logs
```spl
# In Splunk Web
Settings > Indexes > New Index
Index Name: windows
Max Size: 500 MB
```

### Create Dashboard for Authentication Monitoring

**SPL Query for Failed Logins:**
```spl
index=windows EventCode=4625
| stats count by Account_Name, src_ip
| sort - count
| head 20
```

**SPL Query for Kerberos TGT Requests:**
```spl
index=windows EventCode=4768
| stats count by Account_Name, src_ip
| where count > 10
```

**SPL Query for Service Ticket Requests (Kerberoasting Detection):**
```spl
index=windows EventCode=4769 Service_Name!="*$"
| stats count by Account_Name, Service_Name
| where count > 5
```

---

## 🔥 Attack Scenarios

### Scenario 1: Kerberoasting Attack

**Attack (from Kali):**
```bash
# Request service tickets
impacket-GetUserSPNs lab.local/jdoe:P@ssw0rd123! -dc-ip 192.168.10.10 -request

# Crack offline
hashcat -m 13100 tickets.txt /usr/share/wordlists/rockyou.txt
```

**Detection in Splunk:**
```spl
index=windows EventCode=4769 Ticket_Encryption_Type=0x17
| stats count by Account_Name, Service_Name
| where count > 3
```

**Alert Triggered:**
```
User: jdoe
Service Tickets Requested: 15
Time Window: 5 minutes
⚠️ ALERT: Potential Kerberoasting Attack
```

---

### Scenario 2: Brute Force Attack

**Attack:**
```bash
# Hydra brute force
hydra -l jdoe -P /usr/share/wordlists/rockyou.txt smb://192.168.10.10
```

**Detection:**
```spl
index=windows EventCode=4625
| stats count by Account_Name, src_ip
| where count > 5
```

---

### Scenario 3: Pass-the-Hash Detection

**Attack:**
```bash
# Extract NTLM hash with Mimikatz
mimikatz # sekurlsa::logonpasswords

# Use hash for lateral movement
impacket-psexec -hashes :NTLMHASH administrator@192.168.10.11
```

**Detection:**
```spl
index=windows EventCode=4624 Logon_Type=3 Logon_Process=NtLmSsp
| stats count by Account_Name, src_ip
```

---

## 📊 Detection Rules

### Custom Alert: Multiple Failed Logins

**Create Saved Search:**
```spl
index=windows EventCode=4625
| stats count by Account_Name, src_ip
| where count > 5
```

**Alert Trigger**: Run every 5 minutes  
**Action**: Send email to SOC

---

### Custom Alert: Suspicious Kerberos Activity
```spl
index=windows (EventCode=4768 OR EventCode=4769)
| stats count by Account_Name
| where count > 20
```

---

## 📸 Screenshots

### Active Directory Users and Computers
<p align="center">
  <img src="assets/ad-users.png" width="800" alt="AD Users">
</p>

### Splunk Dashboard - Authentication Monitoring
<p align="center">
  <img src="assets/splunk-auth-dashboard.png" width="800" alt="Splunk Dashboard">
</p>

### Kerberoasting Detection Alert
<p align="center">
  <img src="assets/kerberoasting-alert.png" width="800" alt="Kerberoasting Alert">
</p>

---

## 🎓 Learning Outcomes

By completing this lab, you will:

✅ **Understand Active Directory** architecture and authentication protocols  
✅ **Deploy Splunk Enterprise** with Universal Forwarders  
✅ **Configure log collection** from Windows Event Logs and Sysmon  
✅ **Create SPL queries** for threat hunting  
✅ **Detect real-world attacks** (Kerberoasting, Pass-the-Hash, Brute Force)  
✅ **Build correlation rules** for behavioral analysis  
✅ **Document security incidents** with evidence collection  

---

## 📚 Resources

### Official Documentation
- [Microsoft Active Directory Documentation](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/)
- [Splunk Documentation](https://docs.splunk.com/)
- [Sysmon Documentation](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)

### Attack Techniques
- [MITRE ATT&CK - Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
- [MITRE ATT&CK - Pass the Hash](https://attack.mitre.org/techniques/T1550/002/)

### Tools
- [Impacket Toolkit](https://github.com/SecureAuthCorp/impacket)
- [Mimikatz](https://github.com/gentilkiwi/mimikatz)
- [BloodHound](https://github.com/BloodHoundAD/BloodHound)


<div align="center">

**⭐ Star this repository if you found it useful!**

</div>
