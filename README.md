# 🖥️ Lab 3: Windows Server 2019 — Domain Member & SQL Server

**Installation → Configuration → Domain Join → SQL Server 2019 → Verification**

---

## Overview

Install Windows Server 2019 from scratch and configure it as an Active Directory domain member server running SQL Server 2019. This replicates a real enterprise database server integrated into an AD domain environment.

## Prerequisites

- ✅ Working Windows Server 2022 Domain Controller (Lab 1)
- ✅ Windows Server 2019 ISO
- ✅ SQL Server 2019 Express installer
- ✅ SQL Server Management Studio (SSMS) installer
- ✅ VMware/VirtualBox with 4GB RAM, 60GB disk, 2 vCPU
- ✅ Isolated internal virtual network (same as DC)

## Lab Environment Specifications

| Component | Specification | Notes |
|-----------|---------------|-------|
| Platform | VMware / VirtualBox | Any version supporting Server 2019 |
| OS | Windows Server 2019 Standard Desktop Experience | GUI required for this lab |
| RAM | 4GB minimum (8GB recommended) | Server 2019 runs leaner than Windows 11 |
| Storage | 60GB virtual disk | Use dynamic allocation |
| CPU | 2 vCPUs minimum | No TPM required |
| Network | Internal/Host-Only | Must match DC network |
| Role | Domain Member Server | Not a domain controller |

---

## Installation Procedure

### Phase 1: Virtual Machine Creation

#### Step 01 — Create the Virtual Machine

1. Open VMware Workstation / VirtualBox
2. Create a new virtual machine
3. Select Windows Server 2019 ISO as installer disc
4. Name the VM (e.g., `DB-Server01`, `AppServer01`)
5. Set disk size: **60GB** → store as single file
6. Customize hardware:
   - Memory: **4096 MB** (8192 MB recommended)
   - Processors: **2 vCPUs**
   - Network Adapter: Set to **same internal network as Domain Controller**
   - Skip TPM (not required for Server 2019)
7. Power on the VM

---

### Phase 2: Windows Server 2019 Installation

#### Step 02 — Boot from ISO and Begin Installation

1. Press any key to boot from ISO when prompted
2. Select installation language, time/currency format, keyboard input → **Next**
3. Click **Install now**
4. Select operating system edition:
   - Choose: **Windows Server 2019 Standard (Desktop Experience)** ✅
   - Avoid: Server Core editions (CLI only)
5. Accept license terms → **Next**
6. Select **Custom: Install Windows only (advanced)**

#### Step 03 — Partition Disk and Install

1. On disk selection screen → click **New** to create partition
2. Click **Apply** (Windows creates system partitions automatically)
3. Select the **largest Primary partition** → **Next**
4. Windows installation begins (files copy, features install, updates run)
5. Server restarts automatically multiple times — **do not interrupt**
6. Installation completes when **Customize settings screen appears**

#### Step 04 — Set Administrator Password (First Boot)

1. After final restart, **Customize settings screen** appears
2. Enter strong Administrator password → confirm password → **Finish**
3. Windows logs in to desktop
4. **Server Manager** opens automatically (primary management tool)

---

### Phase 3: Initial Server Configuration

#### Step 05a — Set Computer Name

1. In **Server Manager Dashboard** → click **Local Server** (left pane)
2. Click the **Computer Name** link (blue, e.g., WIN-XXXXXXXXX)
3. **System Properties** opens → click **Change** (under Computer Name tab)
4. Enter meaningful server name (e.g., `DB-Server01`)
5. Click **OK** → **OK** → click **Restart Now**

**Alternative (PowerShell):**
```powershell
Rename-Computer -NewName 'DB-Server01' -Restart
Step 05b — Configure Network Adapter and DNS
This is critical for domain integration — DNS must point to Domain Controller.

In Server Manager → Local Server → click Ethernet (blue link next to network adapter)
Network Connections opens → right-click network adapter → Properties
Select Internet Protocol Version 4 (TCP/IPv4) → Properties
Select Use the following IP address:
IP address: Choose unused address on lab subnet
Subnet mask: Match your lab network (e.g., 255.255.255.0)
Default gateway: Lab network gateway (or blank for isolated lab)
Select Use the following DNS server addresses:
Preferred DNS server: IP address of your Domain Controller
Alternate DNS server: Leave blank or repeat DC IP
Click OK → OK → close Network Connections
Test connectivity:
Open Command Prompt
ping <DC-IP> → should get replies
nslookup yourdomain.local → must return DC IP
Alternative (PowerShell):

PowerShell
New-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.x.x' -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses '<DC-IP>'

# Verify:
Get-NetIPAddress | Select InterfaceAlias, IPAddress
Get-DnsClientServerAddress
nslookup yourdomain.local
Step 05c — Disable IE Enhanced Security Configuration (ESC)
ESC blocks downloads needed for SQL Server installation.

Server Manager Dashboard → Local Server
Find IE Enhanced Security Configuration → click On (blue link)
Set both Administrators and Users to Off → OK
Step 05d — Set Windows Updates Policy
Server Manager → Local Server → Windows Update → click link
Configure as appropriate for your lab (manual updates recommended for stability)
Optionally run Windows Update now
Step 05e — Configure Windows Firewall
Server Manager → Local Server → Windows Defender Firewall → click On
Verify Domain network profile is active
Note: Port 1433 (SQL Server) will be opened in Step 11c
Phase 4: Active Directory Domain Join
Step 06 — Join Windows Server 2019 to the Domain
DNS must be correctly configured (Step 05b) before proceeding.

Server Manager Dashboard → Local Server → click Computer Name link (blue)
System Properties → click Change (under Computer Name tab)
In Computer Name/Domain Changes window:
Under Member of → select Domain
Type your domain name (e.g., yourdomain.local)
Click OK
Windows Security prompt appears:
Enter domain administrator credentials
Format: DOMAIN\Administrator or Administrator@yourdomain.local
Click OK → confirmation message: "Welcome to the [domain] domain" → OK
Click OK on System Properties → click Restart Now
After restart — log in with domain administrator account
Alternative (PowerShell):

PowerShell
Add-Computer -DomainName 'yourdomain.local' `
    -Credential (Get-Credential) `
    -OUPath 'OU=Servers,DC=yourdomain,DC=local' `
    -Restart
Step 07 — Post-Domain-Join Verification
Log in after restart with domain administrator credentials
Open Command Prompt and run verification commands:
cmd
whoami
systeminfo | findstr /i "domain"
ping domaincontroller.yourdomain.local
klist
Expected results:

whoami returns: yourdomain\administrator
systeminfo shows your domain name
ping succeeds (DNS resolution working)
klist shows Kerberos tickets
Server Manager → Local Server → confirm Domain shows your domain name
On Domain Controller: open Active Directory Users and Computers → Computers container → verify your server appears
Phase 5: SQL Server 2019 Installation
Step 08 — Install SQL Server 2019
Step 08a — Download and Launch Installer
Download SQL Server 2019 Express from Microsoft
Run installer as Administrator
SQL Server Installation Center opens
Click Installation (left pane) → click New SQL Server stand-alone installation...
Accept license terms → Next
Feature Selection — select:
✅ Database Engine Services (required)
⚪ SQL Server Replication (optional)
⚪ Full-Text and Semantic Extractions for Search (optional)
✅ Client Tools Connectivity (recommended)
Click Next
Step 08b — Instance Configuration
Instance Configuration screen appears
Select instance type:
Default instance (connects via server IP/name only)
Named instance (connects via server\instancename)
Click Next
Step 08c — Server Configuration
Server Configuration screen → leave defaults for lab
SQL Server Database Engine → Startup Type: Automatic
SQL Server Browser → Startup Type: Automatic (required for named instances)
Click Next
Step 08d — Database Engine Configuration
Authentication Mode → select Mixed Mode (SQL Server and Windows Authentication)
This allows both Windows AND SQL Server logins
Set SA (System Administrator) account password
SQL Server administrators section:
Click Add Current User to add current Windows account
Optionally click Add → add domain Administrator account
Click Next → Install
Wait for installation to complete (5–15 minutes) → Close
Step 09 — Install SQL Server Management Studio (SSMS)
Download SSMS from aka.ms/ssmsfullsetup
Run installer as Administrator
Click Install → wait 5–10 minutes → Restart (or restart later)
After installation, open SSMS from Start menu
Connect to Server dialog:
Server type: Database Engine
Server name: your server name or IP (e.g., DB-Server01\SQLEXPRESS)
Authentication: Windows Authentication
Click Connect
Step 10 — Create SQL Server Logins and Assign Roles
SSMS → expand server node → Security → right-click Logins → New Login
Login — New window:
Select SQL Server authentication
Enter login name (e.g., AppServiceAccount, DBReadUser)
Enter and confirm password
Uncheck Enforce password policy (lab only; always enforce in production)
Click Server Roles (left pane) → assign appropriate roles:
Example roles: bulkadmin, dbcreator, securityadmin, serveradmin, sysadmin
Click OK to create login
Verify login works:
SSMS → File → Connect Object Explorer
Use SQL Server authentication
Enter new login credentials
Click Connect
Phase 6: SQL Server Network Configuration
Step 11 — Configure SQL Server to Accept Remote Connections on Port 1433
Step 11a — Enable TCP/IP Protocol
Open SQL Server Configuration Manager (search in Start menu)
Expand SQL Server Network Configuration → click Protocols for [instance]
Right-click TCP/IP → Enable → OK on warning
Right-click TCP/IP → Properties → IP Addresses tab
Scroll to IPAll section (bottom):
Clear TCP Dynamic Ports field (leave blank)
Set TCP Port to 1433
Click Apply → OK
Left pane → SQL Server Services → right-click SQL Server ([instance]) → Restart
Step 11b — Enable SQL Server Browser Service
SQL Server Configuration Manager → SQL Server Services
Right-click SQL Server Browser → Properties
Service tab → Start Mode: Automatic → Apply
Right-click SQL Server Browser → Start
Step 11c — Allow SQL Server Through Windows Firewall
Open Windows Defender Firewall with Advanced Security (search Start menu)
Click Inbound Rules → New Rule → select Port → Next
Select TCP → Specific local ports → type 1433 → Next
Select Allow the connection → Next → apply to all profiles → Next
Name the rule: SQL Server Port 1433 → Finish
Repeat for UDP port 1434 (SQL Server Browser service)
Alternative (PowerShell):

PowerShell
New-NetFirewallRule -DisplayName 'SQL Server Port 1433' `
    -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow

New-NetFirewallRule -DisplayName 'SQL Server Browser 1434' `
    -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow
Phase 7: End-to-End Verification
Step 12 — Verify Full Setup
From the SQL Server machine itself:
Open SSMS → connect with SQL Server authentication using your test login
Run verification queries:
SQL
SELECT @@SERVERNAME AS ServerName,
       @@VERSION AS SQLVersion,
       SYSTEM_USER AS CurrentLogin,
       DB_NAME() AS CurrentDatabase;

SELECT name, state_desc FROM sys.databases;
From another domain machine (Windows workstation, another server, or DC):
Verify port 1433 is open:
PowerShell
Test-NetConnection -ComputerName DB-Server01 -Port 1433
If SSMS is installed on the remote machine:
Connect to: DB-Server01\SQLEXPRESS or <server-IP>,1433
Use SQL Server authentication
From an attack machine (Kali/Parrot Linux):
Scan for SQL Server:
bash
nmap -sV -p 1433,1434 <server-IP>
Verify port 1433 responds
Key Commands Reference
PowerShell Commands
PowerShell
# Rename server
Rename-Computer -NewName 'DB-Server01' -Restart

# Configure static IP and DNS
New-NetIPAddress -InterfaceAlias 'Ethernet0' -IPAddress '192.168.x.x' -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias 'Ethernet0' -ServerAddresses '<DC-IP>'

# Verify network configuration
Get-NetIPAddress
Get-DnsClientServerAddress
nslookup yourdomain.local

# Domain join
Add-Computer -DomainName 'yourdomain.local' -Credential (Get-Credential) -Restart

# Verify domain membership
(Get-WmiObject Win32_ComputerSystem).Domain
whoami

# Open SQL Server firewall ports
New-NetFirewallRule -DisplayName 'SQL Server 1433' -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow
New-NetFirewallRule -DisplayName 'SQL Server Browser 1434' -Direction Inbound -Protocol UDP -LocalPort 1434 -Action Allow

# Verify SQL Server is listening
netstat -an | findstr 1433

# Test remote SQL connectivity
Test-NetConnection -ComputerName DB-Server01 -Port 1433
T-SQL Commands
SQL
-- Create SQL login
CREATE LOGIN [LoginName] WITH PASSWORD = N'YourPassword!', CHECK_POLICY = OFF;

-- Assign server role
ALTER SERVER ROLE sysadmin ADD MEMBER [LoginName];

-- List all logins and roles
SELECT sp.name AS Login, spr.name AS ServerRole
FROM sys.server_principals sp
LEFT JOIN sys.server_role_members srm ON sp.principal_id = srm.member_principal_id
LEFT JOIN sys.server_principals spr ON srm.role_principal_id = spr.principal_id
WHERE sp.type NOT IN ('R')
ORDER BY sp.name;

-- Verify server configuration
SELECT @@SERVERNAME, @@VERSION, SYSTEM_USER;

-- List all databases
SELECT name, state_desc, recovery_model_desc FROM sys.databases;
Security Concepts Covered
Concept	Details
Mixed Mode Authentication	Both Windows and SQL logins enabled — doubles attack surface
Port 1433 Exposure	SQL Server default port — prime target for scanning and brute-force
SA Account	System Administrator with full control — high-value target
SQL Server Browser (UDP 1434)	Leaks instance info — information disclosure vector
sysadmin Role	Enables xp_cmdshell for OS command execution — privilege escalation path
Firewall Configuration	Must restrict 1433 access — should never be open to all sources
Group Policy Inheritance	Domain-joined servers inherit DC policies — centralised control and risk
Domain vs Member Server	This is a member server, not a DC — lower value but still sensitive
Troubleshooting Guide
Problem	Likely Cause	Solution
Domain join fails — cannot find domain	DNS not pointing to DC	Set DNS to DC IP. Run nslookup yourdomain.local
Domain join fails — access denied	Wrong credentials	Use DOMAIN\Administrator format. Verify permissions
SQL Server not accepting remote connections	TCP/IP disabled or port not configured	Enable TCP/IP in Configuration Manager. Set port 1433
Cannot connect from another machine	Firewall blocking 1433	Add inbound firewall rule for TCP 1433
SSMS cannot find instance	SQL Server Browser not running	Start SQL Server Browser service. Set to Automatic
SA login fails	SA account disabled	In SSMS: Security → Logins → SA → Properties → Status → Enabled
Mixed Mode not available	Not set at install time	Re-run setup → modify instance → change authentication mode
IE Enhanced Security blocks downloads	ESC enabled by default	Server Manager → Local Server → IE ESC → Off
Installation fails	Insufficient disk space	Ensure 60+ GB available. SQL Server needs ~6 GB minimum
Skills Demonstrated
✅ Windows Server 2019 installation from ISO
✅ Server Manager administration and post-install configuration
✅ Static IP and DNS configuration for domain integration
✅ Active Directory domain join procedure
✅ SQL Server 2019 installation and configuration
✅ SQL Server user management and authentication modes
✅ SQL Server network configuration (TCP/IP, ports, services)
✅ Windows Firewall rule creation and management
✅ T-SQL querying and server verification
✅ Remote connectivity testing and troubleshooting
