# Hack The Box - Cap Walkthrough

A personal writeup and command cheat sheet documenting the full exploitation process for the Hack The Box machine: Cap.

---

## Phase 1: Reconnaissance & Scanning

**Concept:** Scanning the target system to identify open ports, active services, and operating system details.

**Command:**

```bash
nmap -sV -sC -O 10.129.137.50
```

**How it works in practice:** Before attacking a machine, you must map out its attack surface. Using `nmap` with service detection (`-sV`), default scripts (`-sC`), and OS fingerprinting (`-O`) reveals exactly what services (like HTTP, FTP, and SSH) are listening for incoming traffic on the remote host.

---

## Phase 2: Web Exploitation & IDOR

**Concept:** Bypassing web application access controls via Insecure Direct Object Reference (IDOR).

**Walkthrough Details:**
* **The Redirect Path:** After navigating to the sidebar and running a "Security Snapshot", the web browser redirects the user to a path formatted as `/[something]/[id]`. The value of **`[something]`** is **`data`** (e.g., `/data/1`).
* **The Vulnerability:** The application fails to validate if the user has authorization to view specific scan files. 
* **The Target ID:** By manually tampering with the URL parameter and changing the ID to **`0`** (`/data/0`), we can access the very first packet capture file generated on the system, which contains sensitive historical data.

---

## Phase 3: Packet Analysis (Wireshark)

**Concept:** Inspecting raw network captures to recover plaintext credentials sent over unencrypted protocols.

**Analysis Steps:**
1. Download the `.pcap` file from `/data/0` and open it inside **Wireshark**.
2. Apply a display filter for the **`ftp`** (File Transfer Protocol) application layer protocol to isolate traffic.
3. Right-click an FTP packet and select **Follow -> TCP Stream** to view the raw communication text.

**Recovered Credentials:**
```text
USER nathan
PASS Buck3tH4TF0RM3!
```

---

## Phase 4: Initial Access (User Flag)

**Concept:** Using the recovered credentials to establish a secure remote management terminal session.

**Command:**

```bash
ssh nathan@10.129.137.50
cat user.txt
```

**How it works in practice:** Since users frequently reuse passwords across multiple services, the credentials harvested from the unencrypted FTP traffic are tried against the **SSH** service. Login is successful, allowing us to drop into a system shell and capture the user flag.

---

## Phase 5: Privilege Escalation (Root Flag)

**Concept:** Abusing Linux binary capabilities (`cap_setuid`) to elevate privileges to the system administrator.

**Command:**

```bash
# Locate binaries with elevated capabilities
getcap -r / 2>/dev/null

# Abuse the Python capability to spoof the root User ID and spawn a shell
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Capture the final root flag
cat /root/root.txt
```

**How it works in practice:** Running `getcap` reveals that the host's native Python interpreter is dangerously misconfigured with the `cap_setuid` flag. By executing a one-line Python script, we can explicitly force the system process to set its User ID to `0` (Root) and spawn a fresh `bash` shell, granting full administrative ownership over the server.
