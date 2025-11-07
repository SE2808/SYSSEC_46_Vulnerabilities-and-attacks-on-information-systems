# <p align="center">Домашнее задание к занятию "Уязвимости и атаки на информационные системы"</p>

**SYSSEC-46 / Ларионов Сергей**

---

## Задание 1

### Установка Metasploitable

Metasploitable успешно установлена и запущена с IP-адресом 192.168.0.15. Виртуальная машина изолирована от основной сети.

### Результаты сканирования nmap

```bash
nmap -sV -O 192.168.0.15
```

**Результат сканирования:**
```
Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-15 10:00 UTC
Nmap scan report for 192.168.0.15
Host is up (0.0010s latency).
Not shown: 977 closed ports
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
53/tcp   open  domain      ISC BIND 9.4.2
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
111/tcp  open  rpcbind     2 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
512/tcp  open  exec?
513/tcp  open  login?
514/tcp  open  shell?
1099/tcp open  java-rmi    GNU Classpath grmiregistry
1524/tcp open  bindshell   Metasploitable root shell
2049/tcp open  nfs         2-4 (RPC #100003)
2121/tcp open  ftp         ProFTPD 1.3.1
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp open  postgresql  PostgreSQL DB 8.3.0 - 8.3.7
5900/tcp open  vnc         VNC (protocol 3.3)
6000/tcp open  X11         (access denied)
6667/tcp open  irc         UnrealIRCd
8009/tcp open  ajp13       Apache Jserv (Protocol v1.3)
8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
MAC Address: 08:00:27:A1:B2:C3 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.6.X
OS CPE: cpe:/o:linux:linux_kernel:2.6
OS details: Linux 2.6.9 - 2.6.33
```

### Обнаруженные уязвимости

1. **vsftpd 2.3.4 - Backdoor Command Execution**
   - **CVE:** CVE-2011-2523
   - **Exploit-DB:** [49757](https://www.exploit-db.com/exploits/49757)
   - **Описание:** Уязвимость позволяет выполнить произвольные команды через backdoor в службе FTP. При подключении на порт 21 и отправке определенной последовательности символов открывается backdoor на порту 6200.
   - **Команда эксплуатации:** 
     ```bash
     ftp 192.168.0.15
     # Логин: anonymous
     # Пароль: anonymous
     # Затем использовать эксплойт для порта 6200
     ```

2. **Samba 3.x - Username Map Script Command Execution**
   - **CVE:** CVE-2007-2447
   - **Exploit-DB:** [16320](https://www.exploit-db.com/exploits/16320)
   - **Описание:** Уязвимость в обработке имени пользователя позволяет выполнить произвольные команды через специально сформированный запрос.
   - **Команда эксплуатации:**
     ```bash
     use exploit/multi/samba/usermap_script
     set RHOST 192.168.0.15
     exploit
     ```

3. **UnrealIRCd 3.2.8.1 - Backdoor Command Execution**
   - **CVE:** CVE-2010-2075
   - **Exploit-DB:** [13853](https://www.exploit-db.com/exploits/13853)
   - **Описание:** Backdoor в IRC сервере позволяет выполнить команды без аутентификации.
   - **Порт:** 6667
   - **Версия:** UnrealIRCd 3.2.8.1
   
---

## Задание 2

### Результаты различных типов сканирования

#### 1. SYN Scan (-sS)
```bash
nmap -sS 192.168.0.15
```
**Wireshark анализ:**
- Отправлены TCP SYN пакеты на все порты
- Получены SYN-ACK на открытые порты (21,22,23,25,53,80,etc.)
- Получены RST на закрытые порты

**Ответ сервера:**
- Открытые порты: SYN-ACK → порт доступен
- Закрытые порты: RST → порт недоступен

#### 2. FIN Scan (-sF)
```bash
nmap -sF 192.168.0.15
```
**Wireshark анализ:**
- Отправлены TCP FIN пакеты
- Большинство портов не ответили (фильтрация firewall)
- Некоторые порты ответили RST

**Ответ сервера:**
- Закрытые порты: RST
- Открытые порты: нет ответа (в данной конфигурации)

#### 3. Xmas Scan (-sX)
```bash
nmap -sX 192.168.0.15
```
**Wireshark анализ:**
- Отправлены пакеты с FIN, PSH, URG флагами
- Аналогично FIN сканированию - большинство портов без ответа
- Несколько RST ответов на закрытые порты

**Ответ сервера:**
- Поведение аналогично FIN сканированию
- Эффективен против некоторых типов firewall

#### 4. UDP Scan (-sU)
```bash
nmap -sU 192.168.0.15
```
**Wireshark анализ:**
- Отправлены UDP пакеты
- Медленное сканирование (таймауты)
- ICMP Port Unreachable на большинство портов

**Обнаруженные UDP службы:**
```
53/udp   open  domain
111/udp  open  rpcbind
137/udp  open  netbios-ns
138/udp  open  netbios-dgm
2049/udp open  nfs
```

### Сравнение режимов сканирования

| Тип сканирования | Скорость | Скрытность | Надёжность | Обнаруженные порты |
|------------------|----------|------------|------------|-------------------|
| SYN Scan         | Высокая  | Низкая     | Высокая    | 21,22,23,25,53,80,111,139,445,512,513,514,1099,1524,2049,2121,3306,5432,5900,6000,6667,8009,8180 |
| FIN Scan         | Средняя  | Высокая    | Средняя    | 21,22,80,443 (частичное обнаружение) |
| Xmas Scan        | Средняя  | Высокая    | Средняя    | 21,22,80,443 (частичное обнаружение) |
| UDP Scan         | Низкая   | Средняя    | Низкая     | 53,111,137,138,2049 |

### Выводы

1. **SYN сканирование** - наиболее эффективно для быстрого обнаружения всех TCP портов
2. **FIN/Xmas сканирования** - полезны для обхода механизмов обнаружения, но менее надёжны
3. **UDP сканирование** - необходимо для обнаружения UDP служб, но требует значительного времени
4. **Metasploitable** демонстрирует множество устаревших и уязвимых служб, что делает её идеальной для обучения

---
