# Hack The Box - Cap Walkthrough

**Machine Author(s):** infosecjack  
**Difficulty:** Easy  
**Classification:** Walkthrough / Notes  
**Prepared By:** cpeAdrian  

## Synopsis
Cap is an easy difficulty Linux machine running a web server that performs administrative network captures. Due to missing access controls, an Insecure Direct Object Reference (IDOR) vulnerability allows access to other users' historical packet captures. Analyzing the capture leaks plaintext credentials for a local user, which are used to gain SSH access. Privilege escalation is achieved by locating and abusing a misconfigured Linux binary capability.

---

## Phase 1: Reconnaissance & Scanning

**Concept:** Scanning the target host to discover open ports and active services.

**Command:**

```bash
nmap -sV -sC -O 10.129.137.50
```

**How it works in practice:** Before interacting with the system, we map its entry points. Running `nmap` with service version detection (`-sV`), default scripts (`-sC`), and operating system detection (`-O`) reveals exactly what software components are listening for connections.

**Discovered Services:**
* **Port 21 (FTP):** File Transfer Protocol service.
* **Port 22 (SSH):** Secure Shell remote terminal service.
* **Port 80 (HTTP):** Web application dashboard executing background network utilities.

---

## Phase 2: Web Exploitation & IDOR

**Concept:** Navigating web functionalities to look for object manipulation vulnerabilities.

**Walkthrough Details:**
* **The Redirect Path:** Triggering a **Security Snapshot** via the sidebar menu runs a network capture and redirects the web browser path format to `/[something]/[id]`. The value of **`[something]`** is **`data`** (resulting in a path like `/data/1`).
* **The Vulnerability:** The web page references saved files using a simple, predictable numbering system without verifying if the requesting user owns that data.
* **The Target ID:** By manually editing the URL address parameter back to **`0`** (`http://10.129.137`), we successfully access and download the very first packet capture file generated on the system.

---

## Phase 3: Packet Analysis (Wireshark)

**Concept:** Extracting unencrypted data from network captures.

**Analysis Steps:**
1. Open the downloaded `.pcap` capture file inside **Wireshark**.
2. Filter the packet stream for the **`ftp`** application layer protocol.
3. Right-click an FTP packet entry, navigate to **Follow**, and select **TCP Stream** to read the communications transcript in plaintext.

**Recovered Credentials:**
```text
USER nathan
PASS Buck3tH4TF0RM3!
```

---

## Phase 4: Initial Access (User Flag)

**Concept:** Exploiting credential reuse to gain a remote terminal access shell.

**Command:**

```bash
# Log into the target system using the recovered credentials
ssh nathan@10.129.137.50

# View the user flag string
cat user.txt
```

**How it works in practice:** Because credentials are often reused across services, the plaintext login recovered from the unencrypted FTP traffic is supplied directly to the target's **SSH** service. The connection succeeds, granting a stable Linux shell as the user `nathan` to capture the user flag.

---

## Phase 5: Privilege Escalation (Root Flag)

**Concept:** Manually auditing system privileges to locate and abuse dangerous Linux capabilities.

**Command:**

```bash
# Audit the file system to locate binaries with extra capabilities
getcap -r / 2>/dev/null

# Leverage Python's cap_setuid privilege to spawn an administrative root shell
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Display the final root flag string
cat /root/root.txt
```

**How it works in practice:** Running the native `getcap` command scans the system for binaries with advanced process permissions. The output reveals that the target's Python binary (`/usr/bin/python3.8`) has a dangerous **`cap_setuid`** capability assigned to it. By running a quick one-liner script, Python alters the running process User ID to `0` (the universal ID for root) and launches `bash`, granting immediate and full root control over the machine.
