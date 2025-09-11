---
title: "Write-up: Editor HTB"
author: "Mirena"
date: 2025-09-11
tags: [htb, linux, rce, privesc, xwiki, netdata]
---

# 📝 Write-up: Editor HTB

![editor](https://www.hackthebox.com/storage/avatars/7d4a6dfe5b291cbe7b4e13e5f5c4d6c3.png)

**Сложность:** Легкая  
**Платформа:** Hack The Box  
**Ссылка:** [Editor @ HTB](https://app.hackthebox.com/machines/Editor)

---

## 🎯 Резюме
Editor — это **легкая Linux-машина**, которая включает:  
- Использование **XWiki RCE** (CVE-2025-24893) для получения первоначального доступа  
- Использование **повторного пароля** для горизонтального повышения привилегий  
- Злоупотребление **SUID-бинарником Netdata** (CVE-2024-32019) для получения root  

---

## 📋 Содержание
1. [Разведка](#-разведка)  
2. [Перечисление сервисов](#-перечисление-сервисов)  
3. [Эксплуатация: начальный доступ](#-эксплуатация-начальный-доступ)  
4. [Повышение привилегий: пользователь](#-повышение-привилегий-пользователь)  
5. [Повышение привилегий: root](#-повышение-привилегий-root)  
6. [Выводы и уроки](#-выводы-и-уроки)  
7. [Ссылки](#-ссылки)  

---

## 🔍 Разведка

Начали с **полного сканирования портов** с помощью `nmap`:

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
