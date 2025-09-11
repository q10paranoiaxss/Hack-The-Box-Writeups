# 📝 Write-up: Editor HTB

![editor](https://www.hackthebox.com/storage/avatars/7d4a6dfe5b291cbe7b4e13e5f5c4d6c3.png)

**Difficulty:** Easy  
**Platform:** Hack The Box  
**Link:** [Editor @ HTB](https://app.hackthebox.com/machines/Editor)

---

## 🎯 Summary
Editor is an **easy Linux machine** that involves:  
- Exploiting an **XWiki RCE** (CVE-2025-24893) for initial access.  
- Leveraging **password reuse** for horizontal privilege escalation.  
- Abusing a **Netdata SUID binary** (CVE-2024-32019) to get root.  

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

## 🕵️ Enumeration

Port 80/8000 - HTTP Servers

    Port 80 hosts a static page with project documentation.
    Port 8000 runs XWiki (version 14.10.5), a Java-based wiki platform.

Port 3000 - Express Server

    Node.js application with login functionality.

    Discovered default credentials admin:admin → leads to project management dashboard.

ort 9000 - Netdata

    Real-time performance monitoring tool (version 1.32.1).

    Vulnerable to CVE-2024-32019 (SUID privilege escalation).