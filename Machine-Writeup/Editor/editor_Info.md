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